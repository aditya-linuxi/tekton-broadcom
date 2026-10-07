# Tekton Installation and Integration with Buildpacks, BuildKit, and Cosign on VKS

## Introduction

**VKS (vSphere Kubernetes Service):** A Kubernetes service provided by VMware Cloud Foundation for creating and managing Kubernetes clusters on vSphere infrastructure.

**Tekton:** A Kubernetes-native CI/CD framework used to create and run automated pipelines as Kubernetes resources.

**Cloud Native Buildpacks (CNB):** A technology that converts application source code into container images without requiring the developer to write a Dockerfile.

**BuildKit:** A modern container image build engine used to build images from Dockerfiles efficiently and supports advanced build features such as caching and parallel builds.

**Cosign:** A tool used to digitally sign and verify container images and other OCI artifacts. It helps confirm that an image came from a trusted source and has not been tampered with.

**SBOM (Software Bill of Materials):** A detailed list of the software components, libraries, and dependencies contained in a container image. It helps with vulnerability tracking and software supply-chain security.

**Harbor:** A private container registry used to store and manage the container images produced by the Tekton pipeline.

**Tekton Triggers:** A Tekton component that listens for events (such as a repository webhook) and automatically creates PipelineRuns. It is made of an EventListener, a TriggerBinding and a TriggerTemplate.

**Webhook:** An HTTP call the repository sends to the Tekton EventListener every time someone pushes code.

**External DNS:** A Kubernetes component that automatically creates DNS records (here in Cloudflare) for exposed services, so the Tekton Dashboard is reached by a DNS name.

## Architecture

All of this — Tekton, BuildKit, Buildpacks, and Cosign — runs inside the VKS cluster itself. Nothing extra needs to be installed outside it. Everything after the developer's push happens automatically, with no manual steps.

```text
  Developer                              User (browser)
      │ git push origin main                  │ https://<dashboard-dns-name>
      ▼                                       ▼
  Git repository                         Cloudflare DNS
      │ webhook (POST)                        │ resolves to the Dashboard
      ▼                                       ▼
┌─────────────────────────────── VKS cluster ────────────────────────────────┐
│                                                                            │
│  ┌─ namespace: tekton-pipelines ────────────────────────────────────────┐  │
│  │ Tekton Pipelines     Tekton Triggers     Tekton Dashboard  ◄── User  │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                            │
│  ┌─ namespace: tanzu-system-service-discovery ──────────────────────────┐  │
│  │ External DNS ─► creates the Dashboard DNS record in Cloudflare       │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                            │
│  ┌─ namespace: cicd ────────────────────────────────────────────────────┐  │
│  │ EventListener ─► CEL filter ─► TriggerBinding / TriggerTemplate      │  │
│  │                       │ creates                                      │  │
│  │                       ▼                                              │  │
│  │ PipelineRun  (travelportal-pipeline-values-update)                   │  │
│  │  1. clone         git-clone-update                                   │  │
│  │  2. detect        detect-build-type (Dockerfile present?)            │  │
│  │         ┌──── yes ────┴──── no ────┐                                 │  │
│  │         ▼                          ▼                                 │  │
│  │     BuildKit                  Buildpacks                             │  │
│  │         └────────────┬─────────────┘                                 │  │
│  │                      ▼                                               │  │
│  │  3. sign          Syft SBOM ─► Cosign sign + attest (image digest)   │  │
│  │  4. update-values commit image digest to helm-charts/values.yaml     │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                            │
│  ArgoCD ─► watches the Git repository ─► deploys the signed image to VKS   │
└────────────────────────────────────────────────────────────────────────────┘
      │ image + signature + SBOM            │ values.yaml commit (GitOps)
      ▼                                     ▼
   Harbor                             Git repository
```

| Step | What it means in plain words |
|---|---|
| Developer pushes code | The only thing a person actually has to do |
| Webhook | The repository's call to Tekton on every push |
| EventListener | Receives the webhook, filters it (main branch only, ignores the pipeline's own commits) and creates the PipelineRun |
| Self-trigger filter | Stops the values.yaml commit from the last step from starting the pipeline again |
| Tekton | The automation engine — runs the whole pipeline |
| Dockerfile check | The one decision point: which tool builds the image |
| BuildKit / Buildpacks | The two ways an image actually gets built (Dockerfile present → BuildKit, absent → Buildpacks) |
| SBOM (ingredients list) | A record of everything that went into the image, for security and audit |
| Cosign signing | Proves the image really came from our pipeline and hasn't been tampered with |
| Harbor | Where images and OCI security artifacts are stored and scanned; signature enforcement depends on the configured security policy |
| update-values (GitOps) | Writes the new image digest into the Helm values file in Git |
| ArgoCD | Takes the newly signed image and rolls it out to the running application |
| External DNS + Cloudflare | Publishes the DNS name used to open the Tekton Dashboard |

## Why use BuildKit and Buildpacks with Tekton on VKS?

In plain terms, here's why this combination makes sense:

One automatic process for everyone. Whether or not a team wrote a Dockerfile, their code still gets built the same reliable way — Tekton decides which tool to use, so nobody has to remember or configure it manually.

Nothing gets built by hand. A person pushing code is the only manual step. Everything after that — building, checking for security issues, signing, publishing — happens on its own.

Trust is built in, not bolted on. Every image gets signed and gets an ingredients list (SBOM) the moment it's built — not added later as an afterthought. Harbor can be configured to enforce signature or artifact-security policies before images are promoted or deployed.

It all runs on the same platform. VKS already gives us the cluster, the storage, the networking, Harbor (image store), and ArgoCD (deployment) — Tekton just plugs into what's already there instead of needing its own separate servers.

Works with no internet too. Everything described here — BuildKit, Buildpacks, Tekton, and Cosign — can run fully offline, which matters for secure/air-gapped environments.

One dashboard to watch it all. Tekton's Dashboard shows every build, whether it succeeded or failed, and why — in a web page, not just log files.

A push starts the build by itself. The repository sends a webhook to Tekton Triggers, which creates the PipelineRun. Nobody has to start it manually, and commits made by the pipeline itself never start a second build.

## Benefits

| Benefit | In simple words |
|---|---|
| No guesswork on how to build | Tekton always knows: Dockerfile present → BuildKit, absent → Buildpacks. |
| Less work for app teams | Teams don't need to write or maintain a Dockerfile if they don't want to — Buildpacks handles it for them. |
| Fully automatic | A code push is the only human action; the build, sign, and publish steps run by themselves. |
| Every image is signed | Cosign signs the image and its ingredients list (SBOM) right after it's built, so nothing unverified reaches production. |
| Safer builds | BuildKit and Buildpacks both run without needing root/admin access inside the cluster. |
| One place for images | Every built image lands in Harbor, where it's scanned, verified, and stored. |
| Same steps everywhere | The same pipeline can be used offline after all required artifacts and dependencies have been mirrored into the environment. |
| You can always prove what's running | Images are tracked by a fixed ID (a digest), not a name that can change, and each one has a matching signed ingredients list. |
| One dashboard for visibility | Anyone can open the Tekton Dashboard by its DNS name and see the status of every build, without needing cluster access. |
| Builds start on their own | A push to the main branch sends a webhook to Tekton Triggers, which creates the PipelineRun — no manual PipelineRun needed. |
| No endless build loops | The commit the pipeline makes to update values.yaml is filtered out, so it does not trigger another build. |

## Prerequisites

### Platform

| Component | Version | What it's for |
|---|---|---|
| VMware Cloud Foundation (VCF) | 9.x | Sets up and manages the Supervisor and workload clusters |
| vSphere Kubernetes Service (VKS) | VCF 9.x bundled | Runs the actual Kubernetes workload cluster where builds happen |
| Regional Harbor | Installed via VCF Automation / Supervisor Service | Stores, scans, and signs images |
| External DNS + Cloudflare | Namespace `tanzu-system-service-discovery` | Creates the DNS record used to reach the Tekton Dashboard |
| Tekton Pipelines | Latest version supported by your cluster | Runs the CI pipeline — the automation engine described above |
| Tekton Dashboard | Same release train as Pipelines | The web page for watching build status |
| Cosign | Latest stable | Signs images and their ingredients lists (SBOMs) |
| Syft | Latest stable | Generates the ingredients list (SBOM) for each image |
| Tekton Triggers | Same release train as Pipelines | Receives the repository webhook and starts the pipeline automatically |

### Tools needed on your build machine

```bash
kubectl version
helm version
git --version
curl --version
docker --version    # optional, only needed for local testing
```

## Manifest repository (where the YAML files live)

To keep this document short, the YAML manifests are **not** pasted inline. Every manifest is stored in the repository below, and each section links directly to the file it needs.

| Item | Location |
|---|---|
| Tekton pipeline and External DNS manifests | https://github.com/kondurupurandhar/keda-vks/tree/main/manifest/tekton-catalyst |
| Test Task / TaskRun manifests | https://github.com/kondurupurandhar/keda-vks/tree/main/manifest/test-task |

To get the Tekton pipeline and External DNS files at once:

```bash
git clone https://github.com/kondurupurandhar/keda-vks.git
cd keda-vks/manifest/tekton-catalyst
```

> **Note:** The `keda-vks` repository is private. You need access to it before the links open or the clone works.

> **Validation approach:** Each section ends with a single **Validation** block. Run the steps in the section first, then validate once at the end.

## Installation of Tekton on VKS – Internet Connected VKS

Checked the current Tekton documentation. The current stable/LTS line is Tekton Pipelines v1.15.0, and Tekton's prerequisites include Kubernetes 1.28+, kubectl, and cluster-admin privileges.

### Pre-checks

Confirm the cluster, its version, and your permissions:

```bash
kubectl version                       # Tekton version must be compatible with the Kubernetes version
kubectl cluster-info
kubectl config current-context        # confirm you are connected to the right cluster
kubectl get nodes -o wide             # all nodes should be Ready

kubectl auth can-i create customresourcedefinitions   # expected: yes
kubectl auth can-i create clusterroles                # expected: yes
kubectl auth can-i create namespaces                  # expected: yes
```

### Check Internet connectivity

A successful ICMP ping is not required for Tekton installation and does not prove HTTPS connectivity. Test the actual endpoints.

From your workstation:

```bash
curl -I https://infra.tekton.dev
curl -I https://registry-1.docker.io/v2/
curl -I https://ghcr.io/v2/
```

From inside the VKS cluster (`kubectl get nodes -o wide` does not test internet access, so run an HTTP test from a Pod):

```bash
kubectl run net-test --rm -it --restart=Never \
  --image=curlimages/curl -- \
  curl -I https://registry-1.docker.io/v2/
```

If your cluster cannot pull `curlimages/curl`, use an already available diagnostic image or an existing Pod that contains `curl`. A successful HTTP response confirms that a workload Pod can reach the external registry. Also verify DNS resolution and proxy/firewall rules when required by your VKS environment.

Later, the worker nodes will need to pull images such as:

```text
paketobuildpacks/builder-jammy-base
moby/buildkit
```

### Install Tekton Pipelines

```bash
kubectl apply --filename https://infra.tekton.dev/tekton-releases/pipeline/latest/release.yaml
```

Many resources are created (CRDs, the `tekton-pipelines` namespace, service accounts, cluster roles, and the controller and webhook deployments). Wait until the pods are ready.

### Validation – Tekton Pipelines

```bash
kubectl get pods -n tekton-pipelines
kubectl get crd | grep tekton
kubectl api-resources | grep tekton
kubectl logs deployment/tekton-pipelines-controller -n tekton-pipelines --tail=50
```

Expected result:

```text
Nodes                         → Ready
Namespace tekton-pipelines    → Active
tekton-pipelines-controller   → 1/1 Running
tekton-pipelines-webhook      → 1/1 Running
CRDs                          → tasks, taskruns, pipelines, pipelineruns (tekton.dev) present
Controller logs               → no ERROR / failed / panic / connection refused
```

## Integration for Pipeline creation + Tekton Dashboard

The Dashboard runs in the `tekton-pipelines` namespace. The application Tasks, Pipelines and PipelineRuns live in a separate `cicd` namespace.

### Create a namespace for your CI/CD objects

Don't put your application Pipelines directly into `tekton-pipelines`. Create a separate namespace (used for the rest of this document):

```bash
kubectl create namespace cicd
```

If it already exists, `Error from server (AlreadyExists)` is fine.

### Install Tekton Dashboard

Since your cluster has Internet access, use the official Tekton Dashboard release manifest:

```bash
kubectl apply -f https://infra.tekton.dev/tekton-releases/dashboard/latest/release.yaml
```

The Dashboard is installed into the `tekton-pipelines` namespace.

### External DNS Configuration for Tekton Dashboard

The Tekton Dashboard is exposed using **External DNS with Cloudflare**. The required configuration files are maintained in the GitHub repository (`manifest/tekton-catalyst`). Apply them one by one, in the order below.

#### 1. External DNS ClusterRole

Defines the RBAC permissions required by External DNS.

File `cluster-roles.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/cluster-roles.yaml

```bash
kubectl apply -f cluster-roles.yaml
```

#### 2. External DNS ClusterRoleBinding

Binds the External DNS ServiceAccount to the ClusterRole.

File `external-dns-clusterrolebinding.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/external-dns-clusterrolebinding.yaml

```bash
kubectl apply -f external-dns-clusterrolebinding.yaml
```

#### 3. Cloudflare API token Secret

Creates the Kubernetes Secret used to store the Cloudflare API token.

File `secret.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/secret.yaml

```bash
kubectl apply -f secret.yaml
```

**Cloudflare API token:** The token must **not be committed to the GitHub repository**. `secret.yaml` contains only the placeholder `<CLOUDFLARE_API_TOKEN>` in the `api-token` key of the `external-dns-cloudflare` Secret (namespace `tanzu-system-service-discovery`). Replace the placeholder with the actual token before applying the Secret.

#### 4. External DNS configuration

Configures External DNS with the Cloudflare provider, domain filter, DNS sources, and TXT registry.

File `external-dns-values.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/external-dns-values.yaml

```bash
kubectl apply -f external-dns-values.yaml
```

#### 5. Cloudflare API token overlay

Injects the Cloudflare API token from the Kubernetes Secret into the External DNS Deployment.

File `cloudflare-secret-overlay.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/cloudflare-secret-overlay.yaml

```bash
kubectl apply -f cloudflare-secret-overlay.yaml
```

### Validation – Tekton Dashboard and External DNS

```bash
kubectl get pods,deployment,svc -n tekton-pipelines | grep dashboard
kubectl get pods -n tanzu-system-service-discovery
kubectl logs -n tanzu-system-service-discovery \
  -l app.kubernetes.io/name=external-dns
```

Expected result:

```text
tekton-dashboard pod/deployment → 1/1 Running
External DNS pod                → Running, no errors in the logs
Cloudflare                      → the expected Tekton Dashboard DNS record exists
```

Open the Dashboard using the configured external DNS name. You should see the Tekton Dashboard. If `tekton-dashboard` isn't ready, don't continue to Pipeline testing yet.

For production, also configure HTTPS, authentication, RBAC, and network restrictions for the Dashboard.

### Test Task and TaskRun

Before creating your actual Buildpacks/BuildKit Pipeline, verify that Tekton can execute a Task. A Task is the definition; a TaskRun actually executes it.

Task file `test-task.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/test-task/test-task.yaml

TaskRun file `taskrun.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/test-task/taskrun.yaml

Apply both:

```bash
kubectl apply -f test-task.yaml
kubectl apply -f taskrun.yaml
```

### Validation – Test Task and TaskRun

```bash
kubectl get task,taskrun -n cicd
kubectl logs -l tekton.dev/taskRun=hello-taskrun -n cicd
```

Expected result:

```text
hello-taskrun   SUCCEEDED=True   REASON=Succeeded

Hello from Tekton!
Tekton Pipeline integration is working.
```

In the Dashboard (opened by its DNS name), select Namespace → `cicd`. The Task and TaskRun should be visible.

# Tekton pipeline setup with Buildpacks, BuildKit, and Cosign

## Prepare the namespace

The `cicd` namespace was created in the previous section. Label it for the build workloads:

```bash
kubectl label namespace cicd pod-security.kubernetes.io/enforce=privileged --overwrite
```

> **Security note:** This lab label enables privileged capabilities. In production, use the least-privilege settings supported by your VKS/BuildKit/Buildpacks implementation and security policy.

## Tasks

Create each Task below by applying its manifest. A single validation at the end of this section checks all of them.

### git-clone-update

`git-clone-update` is a Tekton Task used to clone source code from a Git repository into a shared workspace. It cleans the workspace, clones the required branch/revision, and validates the Git repository. The cloned source code is then used by BuildKit or Buildpacks to build the container image.

File `git-clone-update-task.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/git-clone-update-task.yaml

```bash
kubectl apply -f git-clone-update-task.yaml
```

### detect-build-type

`detect-build-type` is a Tekton Task that checks whether the source repository contains a Dockerfile. It selects BuildKit when a Dockerfile exists, otherwise it selects Cloud Native Buildpacks. The selected build type is stored as a Tekton result for the next pipeline task to use.

File `detect-build-type.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/detect-build-type.yaml

```bash
kubectl apply -f detect-build-type.yaml
```

### buildpacks-phases

A ready-made Tekton Task that builds an image from source code using Cloud Native Buildpacks. It builds the image when there is no Dockerfile.

File `buildpacks-phase-task.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/buildpacks-phase-task.yaml

```bash
kubectl apply -f buildpacks-phase-task.yaml -n cicd
```

### buildkit-build

`buildkit-build` is a Tekton Task that runs when detect-build-type detects a Dockerfile, using BuildKit to build the application container image. It takes the source code from the shared workspace, builds the image from the Dockerfile, and pushes it to Harbor with the configured credentials.

File `buildkit-build-task-update.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/buildkit-build-task-update.yaml

```bash
kubectl apply -f buildkit-build-task-update.yaml
```

### sign-image (SBOM + Cosign)

This `sign-image` Task is the security/signing stage after BuildKit or Buildpacks. Its job is to generate an SBOM, sign the image with Cosign, attach the SBOM as an attestation, and return the image digest.

> `COSIGN_TLOG_UPLOAD` defaults to `false` for the air-gapped/private-registry lab flow. Enable it only when the cluster can reach the approved Sigstore transparency-log service.

File `sign-image-task-update.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/sign-image-task-update.yaml

```bash
kubectl apply -f sign-image-task-update.yaml
```

### update-values

`update-values` is the GitOps update stage of the Tekton pipeline. It updates the Helm chart's values.yaml with the newly built image and its digest, commits the change, and pushes it back to the repository.

File `update-values.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/update-values.yaml

```bash
kubectl apply -f update-values.yaml
```

### Validation – Tasks

```bash
kubectl get tasks -n cicd
```

Expected result (all six Tasks listed):

```text
git-clone-update
detect-build-type
buildpacks-phases
buildkit-build
sign-image
update-values
```

## Pipeline

Tasks are added to the Pipeline using `taskRef`, which references previously created Tekton Tasks. params pass required values to each Task, workspaces share files/credentials, and runAfter/when control the execution order and conditions. The Pipeline first clones the repository, then detects whether a Dockerfile exists and runs BuildKit or Buildpacks accordingly. After the image is built, the sign Task generates the SBOM and signs/attests the image. Finally, the update-values Task receives the image digest from the sign Task, updates values.yaml, commits the change, and pushes it back to the same Git repository—completing the GitOps update.

File `travelportal-pipeline-values-update.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/travelportal-pipeline-values-update.yaml

```bash
kubectl apply -f travelportal-pipeline-values-update.yaml
```

The Pipeline is validated together with the PipelineRun at the end of the next sections.

## Pipeline Dependencies (Storage, ConfigMap, Secrets, Service Account)

Before triggering the PipelineRun, create the required storage, configuration, and credentials. A single validation at the end of this section checks everything.

### Source and Build Cache Storage

The PipelineRun creates dedicated `source` and `cache` PVCs for each run using the `lab-gold-storage-policy` StorageClass. This prevents concurrent PipelineRuns from overwriting the same shared source workspace.

If the StorageClass has a different name in your VKS environment, replace `lab-gold-storage-policy` in the PipelineRun and TriggerTemplate workspace definitions.

### Harbor CA ConfigMap

Create a ConfigMap named `harbor-ca-cert` in the `cicd` namespace from your Harbor / vCenter CA certificate (the key must be `ca.crt`):

```bash
kubectl create configmap harbor-ca-cert \
  -n cicd \
  --from-file=ca.crt=<path-to-harbor-ca.crt>
```

### Secrets

The pipeline needs four secrets in the `cicd` namespace. Create them one by one. Replace every placeholder value with your real credentials before applying.

#### 1. Git credentials Secret (`repo-git-credentials`)

Git repository username and token, used by the clone and update-values Tasks.

File `github-push-secret.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/github-push-secret.yaml

```bash
kubectl apply -f github-push-secret.yaml
```

If you need to create the repository credentials directly instead of from the manifest:

```bash
kubectl create secret generic repo-git-credentials \
  -n cicd \
  --from-literal=username=admin \
  --from-literal=token='<REPO_TOKEN>'
```

#### 2. Harbor registry Secret (`harbor-registry-secret`)

Harbor Docker config (`dockerconfigjson`), used to push and sign images.

File `harbor-credentials.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/harbor-credentials.yaml

```bash
kubectl apply -f harbor-credentials.yaml
```

#### 3. Cosign password Secret (`cosign-password`)

Password of the Cosign private key.

File `cosign-pass.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/cosign-pass.yaml

```bash
kubectl apply -f cosign-pass.yaml
```

#### 4. Cosign key pair Secret (`cosign-key`)

Cosign private and public key. Create it from the generated key pair:

```bash
kubectl create secret generic cosign-key \
  -n cicd \
  --from-file=cosign.key=cosign.key \
  --from-file=cosign.pub=cosign.pub
```

### ServiceAccount

The TriggerTemplate and PipelineRun use the dedicated `tekton-pipeline-sa` ServiceAccount.

File `tekton-pipeline-sa.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/tekton-pipeline-sa.yaml

```bash
kubectl apply -f tekton-pipeline-sa.yaml
```

### Validation – Dependencies

```bash
kubectl get storageclass lab-gold-storage-policy
kubectl get configmap harbor-ca-cert -n cicd
kubectl get secret repo-git-credentials harbor-registry-secret cosign-password cosign-key -n cicd
kubectl get sa tekton-pipeline-sa -n cicd
```

Expected result: every object is found, with no `NotFound` errors.

## PipelineRun

The PipelineRun is used to start and execute the travelportal-pipeline-values-update Pipeline. It provides the pipeline parameters, connects the required workspaces and secrets, and applies additional Pod configuration needed during the pipeline execution. The `pipelineRef` selects the travelportal-pipeline-values-update Pipeline, while params provide the Git repository, branch, Harbor image, and Buildpacks builder image that the Pipeline will use. The workspaces provide the required storage and credentials: the source and cache PVCs are created per PipelineRun, dockerconfig provides Harbor authentication, git-credentials provides repository push credentials, and cosign-key provides the image-signing key. The SBOM is stored in the source workspace, so no separate SBOM workspace is required. The taskRunSpecs customize the BuildKit TaskRun by adding the harbor-ca ConfigMap as a volume. This allows the BuildKit Pod to access the Harbor CA certificate. The BuildKit registry configuration references that CA and does not use `registry.insecure=true` for the Harbor HTTPS endpoint.

File `travelportal-pipelinerun-pack-values-update.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/travelportal-pipelinerun-pack-values-update.yaml

```bash
kubectl apply -f travelportal-pipelinerun-pack-values-update.yaml
```

### Validation – Pipeline, PipelineRun, and output

Pipeline and run status:

```bash
kubectl get pipeline -n cicd
kubectl get pipelinerun -n cicd -w
kubectl get pvc -n cicd
```

Expected result: the Pipeline exists, the PipelineRun finishes with `SUCCEEDED=True`, and the `source` and `cache` PVCs are `Bound`. You can also follow the run in the Tekton Dashboard.

Image in Harbor (from a machine that can reach Harbor):

```bash
docker login lab25-harbor.lab25.sunfire.lab
docker pull lab25-harbor.lab25.sunfire.lab/cicd/travelportal:latest
```

Cosign signature and SBOM attestation:

```bash
IMAGE=lab25-harbor.lab25.sunfire.lab/cicd/travelportal
IMAGE_DIGEST=sha256:<IMAGE_DIGEST>

cosign verify \
  --offline \
  --key cosign.pub \
  "${IMAGE}@${IMAGE_DIGEST}"

cosign verify-attestation \
  --offline \
  --key cosign.pub \
  --type spdxjson \
  "${IMAGE}@${IMAGE_DIGEST}"
```

Expected result: the image pulls, the signature verifies, and the SBOM attestation is associated with the image digest.

# Automatic pipeline trigger with Tekton Triggers

Until now the PipelineRun was started by hand with kubectl apply. In this section a push to the repository starts it automatically. The repository server sits on the internal lab network, so it calls the Tekton EventListener NodePort directly.

## Lab values used in this section

| Item | Value |
|---|---|
| Namespace | `cicd` |
| Pipeline | `travelportal-pipeline-values-update` |
| EventListener | `travelportal-repository-listener` |
| EventListener Service | `el-travelportal-repository-listener` (NodePort) |
| Webhook port | `8080 -> 31877` (observed lab mapping) |
| Node used for webhook | `10.12.92.3` |
| Webhook URL | `http://10.12.92.3:31877` (observed lab URL) |
| Repository server | `http://10.12.90.62` |
| Repository | `http://10.12.90.62/admin/travelPortal-test-buildpack.git` |
| Branch | `main` |

## The Tekton Triggers objects created in this section

| Object | Name |
|---|---|
| ServiceAccount | `repository-trigger-sa` |
| Role | `repository-trigger-role` |
| RoleBinding | `repository-trigger-rolebinding` |
| ClusterRole | `repository-trigger-cluster-role` |
| ClusterRoleBinding | `repository-trigger-cluster-rolebinding` |
| EventListener | `travelportal-repository-listener` |
| CEL interceptor | `ClusterInterceptor` |
| TriggerBinding | `travelportal-repository-binding` |
| TriggerTemplate | `travelportal-repository-template` |

The EventListener/Trigger object names retain `repository` terminology. The names do not change EventListener functionality; the repository endpoint used by the binding is the internal repository server.

## Install Tekton Triggers

For an Internet-connected VKS cluster, install the Tekton Triggers release before creating the EventListener, TriggerBinding, and TriggerTemplate objects.

```bash
kubectl apply --filename https://infra.tekton.dev/tekton-releases/triggers/latest/release.yaml
```

For production, pin the Triggers version that is tested with your selected Tekton Pipelines release instead of using `latest`.

### Validation – Tekton Triggers

```bash
kubectl get crd | grep triggers.tekton.dev
kubectl get pods -n tekton-pipelines | grep triggers
```

Expected result:

```text
CRDs: clusterinterceptors, clustertriggerbindings, eventlisteners, interceptors,
      triggerbindings, triggers, triggertemplates (all .triggers.tekton.dev)

Pods (all 1/1 Running):
      tekton-triggers-controller-...
      tekton-triggers-core-interceptors-...
      tekton-triggers-webhook-...
```

## EventListener ServiceAccount and RBAC

The EventListener runs with `serviceAccountName: repository-trigger-sa`. It needs permission to read the Triggers resources in `cicd`, read the cluster-scoped Triggers resources (`clusterinterceptors`, `clustertriggerbindings`), and create PipelineRuns.

### ServiceAccount

File `webhook-sa.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/webhook-sa.yaml

```bash
kubectl apply -f webhook-sa.yaml
```

### Role, RoleBinding, ClusterRole and ClusterRoleBinding

`eventlistener-rbac.yaml` contains the Role (`repository-trigger-role`), RoleBinding (`repository-trigger-rolebinding`), ClusterRole (`repository-trigger-cluster-role`) and ClusterRoleBinding (`repository-trigger-cluster-rolebinding`).

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/eventlistener-rbac.yaml

```bash
kubectl apply -f eventlistener-rbac.yaml
```

### Validation – RBAC

```bash
kubectl get sa repository-trigger-sa -n cicd
kubectl get role,rolebinding -n cicd | grep repository-trigger
kubectl get clusterrole,clusterrolebinding | grep repository-trigger

for r in triggerbindings triggertemplates eventlisteners; do
  kubectl auth can-i list $r.triggers.tekton.dev \
    --as=system:serviceaccount:cicd:repository-trigger-sa -n cicd
done

for r in clusterinterceptors clustertriggerbindings; do
  kubectl auth can-i list $r.triggers.tekton.dev \
    --as=system:serviceaccount:cicd:repository-trigger-sa
done

kubectl auth can-i create pipelineruns.tekton.dev \
  --as=system:serviceaccount:cicd:repository-trigger-sa -n cicd
```

Expected result: all objects exist and every `can-i` answers `yes`.

By default the repository server blocks outbound webhook calls to hosts that are not on its allow list. Without the change below, the webhook fails with:

```text
webhook can only call allowed HTTP servers
(check your security.ALLOWED_HOST_LIST setting)
```

## TriggerBinding, TriggerTemplate and EventListener

Create the three objects below, then validate them once at the end.

### TriggerBinding

File `gitea-triggerbinding.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/gitea-triggerbinding.yaml

```bash
kubectl apply -f gitea-triggerbinding.yaml
```

The important values are:

```text
REPO_URL = http://10.12.90.62/admin/travelPortal-test-buildpack.git
REVISION = main
```

### TriggerTemplate

File `gitea-triggertemplate.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/gitea-triggertemplate.yaml

```bash
kubectl apply -f gitea-triggertemplate.yaml
```

The PipelineRun it creates must reference `travelportal-pipeline-values-update`, use the dedicated `tekton-pipeline-sa` service account, and pass the repository URL `http://10.12.90.62/admin/travelPortal-test-buildpack.git`.

### EventListener

File `gitea-eventlistener.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/gitea-eventlistener.yaml

```bash
kubectl apply -f gitea-eventlistener.yaml
```

The EventListener uses the cluster-scoped `cel` interceptor and exposes application port `8080` through a Kubernetes `NodePort`. In this lab the observed NodePort is `31877`, so the webhook URL is `http://10.12.92.3:31877`. Because the manifest specifies only `serviceType: NodePort`, Kubernetes may assign a different NodePort in another environment, so always confirm the actual value in the validation below.

### Validation – TriggerBinding, TriggerTemplate and EventListener

Check the objects, the CEL interceptor, and the generated Service:

```bash
kubectl get triggerbinding travelportal-repository-binding -n cicd
kubectl get triggertemplate travelportal-repository-template -n cicd
kubectl get eventlistener travelportal-repository-listener -n cicd
kubectl get clusterinterceptor cel
kubectl get svc,endpoints el-travelportal-repository-listener -n cicd
kubectl get pods -n cicd -l eventlistener=travelportal-repository-listener -o wide
```

Expected result:

```text
EventListener   → AVAILABLE=True, READY=True
CEL interceptor → present
Service         → 8080:31877/TCP
Endpoint        → <pod-ip>:8080
```

Then send a webhook-style test POST to the NodePort (before adding the real webhook):

```bash
cat >/tmp/test-event.json <<'EOF'
{
  "ref": "refs/heads/main",
  "after": "test-sha",
  "head_commit": {
    "message": "test webhook"
  }
}
EOF

curl -v \
  -H 'Content-Type: application/json' \
  --data-binary @/tmp/test-event.json \
  http://10.12.92.3:31877
```

A working EventListener returns `HTTP/1.1 202 Accepted` with a Tekton event ID. This proves the path jumpbox → NodePort → EventListener works, and a new `travelportal-build-xxxxx` PipelineRun appears.

## Add the webhook in the repository

Repository:

```text
admin/travelPortal-test-buildpack
```

Open:

```text
Repository
  -> Settings
  -> Webhooks
  -> Add Webhook
```

Use:

| Setting | Value |
|---|---|
| Target URL | `http://10.12.92.3:31877` |
| Method | `POST` |
| Content type | `application/json` |
| Event | `Push events` |
| Branch | `main` |

Save the webhook. The values read from the push payload are:

```text
body.ref
body.after
body.head_commit.message
```

## Pipeline changes for automatic runs

### Image digest

BuildKit and Buildpacks return their digest under different result names:

| Build | Result |
|---|---|
| BuildKit | `IMAGE_DIGEST` |
| Buildpacks | `APP_IMAGE_DIGEST` |

Both Tasks also write the same file to the shared workspace:

```text
$(workspaces.source.path)/image-digest
```

### Repository credentials for update-values

The clone and update-values Tasks read the repository credentials from the `username` and `token` keys of the `repo-git-credentials` Secret (created in the dependencies section). The token is a Personal Access Token created in the repository server.

## Final validation – end to end

The developer does not create a PipelineRun manually. Make a change and push it:

```bash
git add .
git commit -m "developer change"
git push origin main
```

Then watch the automation:

```bash
kubectl logs -f deployment/el-travelportal-repository-listener -n cicd
kubectl get pipelineruns -n cicd -w
```

Expected result:

```text
Webhook delivery   → HTTP 202 Accepted
New PipelineRun    → travelportal-build-xxxxx
Build              → BuildKit if the repo has a Dockerfile, otherwise Buildpacks
Pipeline           → clone → detect → build → sign → update-values all succeed
Self-trigger       → no second PipelineRun after the CI commit
Harbor             → image digest signed, SBOM attestation present
```

When `update-values` pushes `values.yaml` with the commit message prefix `ci: update TravelPortal image digest`, the webhook fires again, the CEL filter rejects that CI commit, and no second PipelineRun starts.

| Problem | Cause | Action |
|---|---|---|
| Pipeline triggers itself repeatedly | The CI commit message does not match the CEL prefix used by CEL | Ensure `update-values` commits with `ci: update TravelPortal image digest` and the EventListener filters the same prefix |

## Conclusion

This implementation provides a Kubernetes-native CI workflow on VKS using Tekton, BuildKit, Buildpacks, Syft, Cosign, and Harbor. The pipeline automatically selects BuildKit or Buildpacks based on the application source, generates an SBOM, signs the immutable image digest, and stores the resulting image and security artifacts in Harbor.

The architecture also separates the application source and internal CI/GitOps responsibilities across two repositories, providing a clear and controlled workflow for source management and deployment configuration.

Overall, the solution provides an automated, traceable, and security-focused CI process that can be operated in both connected and disconnected VKS environments, provided the required images, dependencies, credentials, and registry artifacts are available locally.
