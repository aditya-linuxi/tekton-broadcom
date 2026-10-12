# Tekton Installation and Integration with Buildpacks, BuildKit, and Cosign on VKS

## 1. Introduction

- **VKS (vSphere Kubernetes Service):** A Kubernetes service provided by VMware Cloud Foundation for creating and managing Kubernetes clusters on vSphere infrastructure.
- **Tekton:** A Kubernetes-native CI/CD framework used to create and run automated pipelines as Kubernetes resources.
- **Cloud Native Buildpacks (CNB):** A technology that converts application source code into container images without requiring the developer to write a Dockerfile.
- **BuildKit:** A modern container image build engine used to build images from Dockerfiles efficiently and supports advanced build features such as caching and parallel builds.
- **Cosign:** A tool used to digitally sign and verify container images and other OCI artifacts. It helps confirm that an image came from a trusted source and has not been tampered with.
- **SBOM (Software Bill of Materials):** A detailed list of the software components, libraries, and dependencies contained in a container image. It helps with vulnerability tracking and software supply-chain security.
- **Harbor:** A private container registry used to store and manage the container images produced by the Tekton pipeline.
- **Tekton Triggers:** A Tekton component that listens for events (such as a repository webhook) and automatically creates PipelineRuns. It is made of an EventListener, a TriggerBinding and a TriggerTemplate.
- **Webhook:** An HTTP call the repository sends to the Tekton EventListener every time someone pushes code.
- **External DNS:** A Kubernetes component that automatically creates DNS records (here in Cloudflare) for exposed services, so the Tekton Dashboard is reached by a DNS name.
- **Istio + Kubernetes Gateway API:** Provides the HTTPS entry point (a Gateway and an HTTPRoute) used to reach the Tekton Dashboard.
- **cert-manager:** Issues the TLS certificate for the Dashboard hostname from the letsencrypt-prod ClusterIssuer.

## 2. Architecture

### 2.1 Architecture Diagram and Flow

All of this — Tekton, BuildKit, Buildpacks, and Cosign — runs inside the VKS cluster itself. Nothing extra needs to be installed outside it. Everything after the developer's push happens automatically, with no manual steps.

```text
  Developer                              User (browser)
      │ git push origin main                  │ https://tekton.k8slabs.com
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

### 2.2 Namespaces

| Namespace | Contents |
|---|---|
| `tekton-pipelines` | Tekton Pipelines, Tekton Triggers, Tekton Dashboard, Gateway, Certificate, HTTPRoute |
| `cicd` | Application Tasks, Pipelines, PipelineRuns and the Triggers objects (Sections 16 to 21) |
| `tanzu-system-service-discovery` | External DNS |
| `istio-system`, `istio-ingress` | Istio |
| `cert-manager` | cert-manager |

## 3. Why use BuildKit and Buildpacks with Tekton on VKS?

In plain terms, here's why this combination makes sense:

**One automatic process for everyone.** Whether or not a team wrote a Dockerfile, their code still gets built the same reliable way — Tekton decides which tool to use, so nobody has to remember or configure it manually.

**Nothing gets built by hand.** A person pushing code is the only manual step. Everything after that — building, checking for security issues, signing, publishing — happens on its own.

**Trust is built in, not bolted on.** Every image gets signed and gets an ingredients list (SBOM) the moment it's built — not added later as an afterthought. Harbor can be configured to enforce signature or artifact-security policies before images are promoted or deployed.

**It all runs on the same platform.** VKS already gives us the cluster, the storage, the networking, Harbor (image store), and ArgoCD (deployment) — Tekton just plugs into what's already there instead of needing its own separate servers.

**Works with no internet too.** Everything described here — BuildKit, Buildpacks, Tekton, and Cosign — can run fully offline, which matters for secure/air-gapped environments.

**One dashboard to watch it all.** Tekton's Dashboard shows every build, whether it succeeded or failed, and why — in a web page, not just log files.

**A push starts the build by itself.** The repository sends a webhook to Tekton Triggers, which creates the PipelineRun. Nobody has to start it manually, and commits made by the pipeline itself never start a second build.

## 4. Benefits

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

## 5. Prerequisites

### 5.1 Platform and components

| Component | Requirement |
|---|---|
| VMware Cloud Foundation / VKS | 9.x, target VKS workload cluster available |
| Kubernetes | 1.28 or later (Tekton requirement) |
| cert-manager | Installed through VKS package management |
| Istio | Installed through VKS package management, with Gateway API CRDs present |
| ClusterIssuer | `letsencrypt-prod` available (DNS-01, Cloudflare) |
| External DNS | VKS package `external-dns.kubernetes.vmware.com` (installed in Section 7.8, or already installed) |
| Harbor | Regional Harbor available (VCF Automation / Supervisor Service) |
| DNS | Cloudflare zone for the lab domain and an API token with DNS edit permission |
| Permissions | cluster-admin on the VKS cluster |

Dashboard DNS name: `tekton.k8slabs.com` is the actual hostname used in this document. If several VKS labs share the `k8slabs.com` zone, confirm the name does not collide with another lab (TO VERIFY).

### 5.2 Required CLI tools

```bash
vcf version
kubectl version --client
git --version
curl --version
```

## 6. Manifest Repository

The YAML manifests are not pasted inline. They are stored in the repository below. Each section names the exact file, lists its important arguments, and links to the YAML.

### 6.1 Repository Locations

| Item | Location |
|---|---|
| Tekton pipeline, External DNS and Triggers manifests | https://github.com/kondurupurandhar/keda-vks/tree/main/manifest/tekton-catalyst |
| Smoke test Task / TaskRun manifests | https://github.com/kondurupurandhar/keda-vks/tree/main/manifest/test-task |

**Note:** The keda-vks repository is private. You need access before the links open or the clone works.

### 6.2 Get the file#### 7.8.1 Cloudflare API token

File `cloudflare-secret.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/cloudflare-secret.yaml

```yaml
stringData:
  # Replace with the Cloudflare API token created in the Cloudflare account.
  # Do NOT commit the real value to Git.
  api-token: <CLOUDFLARE_API_TOKEN>s

```bash
git clone https://github.com/kondurupurandhar/keda-vks.git
cd keda-vks/manifest/tekton-catalyst
```

## 7. Cluster Pre-checks

### 7.1 Access the VKS cluster

```bash
vcf context list
vcf context use wl:<TARGET_VKS_CLUSTER_NAME>
```

### 7.2 Validate cluster and permissions

```bash
kubectl config current-context        # confirm the right cluster
kubectl cluster-info
kubectl version
kubectl get nodes -o wide             # all nodes Ready

kubectl auth can-i create customresourcedefinitions   # expected: yes
kubectl auth can-i create clusterroles                # expected: yes
kubectl auth can-i create namespaces                  # expected: yes
```

### 7.3 Internet connectivity

A successful ping is not required and does not prove HTTPS access. Test the actual endpoints.

From your workstation:

```bash
curl -I https://infra.tekton.dev
curl -I https://registry-1.docker.io/v2/
curl -I https://ghcr.io/v2/
```

### 7.4 Install cert-manager

Install cert-manager Using VKS Package Management

### Retrieve details about the available cert-manager package:

```
vcf package available get \
  cert-manager.kubernetes.vmware.com \
  -n vmware-system-tkg
```

#### 7.4.3 Set the Package Version

The following version is from the source runbook. Confirm its availability in the target repository.
```bash
export CERT_MANAGER_VERSION=1.19.2+vmware.1-vks.1
export CERT_MANAGER_RELEASE_NAME=cert-manager.kubernetes.vmware.com
```

#### 7.4.4 Generate and Review the Data Values

```bash
vcf package available get \
  cert-manager.kubernetes.vmware.com/${CERT_MANAGER_VERSION} \
  -n vmware-system-tkg \
  --default-values-file-output cert-manager-data-values.yaml
cat cert-manager-data-values.yaml
```

#### 7.4.5 Perform a Dry Run

```bash
vcf package install cert-manager \
  --package cert-manager.kubernetes.vmware.com \
  --version "${CERT_MANAGER_VERSION}" \
  -n vmware-system-tkg \
  --values-file cert-manager-data-values.yaml \
  --dry-run
```

#### 7.4.6 Install cert-manager

```bash
vcf package install cert-manager \
  --package cert-manager.kubernetes.vmware.com \
  --version "${CERT_MANAGER_VERSION}" \
  -n vmware-system-tkg \
  --values-file cert-manager-data-values.yaml \
  --wait
```

#### 7.4.7 Verify Installation

```bash
kubectl get packageinstall -n vmware-system-tkg
kubectl get namespaces
kubectl get all -n cert-manager
```





### 7.6 Install Istio Using VKS Package Management

#### 7.6.1 Check available releases

```bash
vcf package available list -A

vcf package available get \
  istio.kubernetes.vmware.com \
  -n vmware-system-tkg
```

#### 7.6.2 Set the Istio version

```
export ISTIO_VERSION=1.28.2+vmware.1-vks.1
export ISTIO_RELEASE_NAME=istio.kubernetes.vmware.com
```

#### 7.6.3 Create the namespace

```bash
kubectl create namespace istio-system
```

#### 7.6.4 Generate the data values file

```bash
vcf package available get \
  istio.kubernetes.vmware.com/${ISTIO_VERSION} \
  -n vmware-system-tkg \
  --default-values-file-output istio-data-values.yaml
cat istio-data-values.yaml
```

#### 7.6.5 Enable the ingress gateway

Edit `istio-data-values.yaml` to include the following configuration while retaining other required package settings:

```yaml
istio:
  gateways:
    ingress:
      enabled: true
```

Review the updated file:

```bash
cat istio-data-values.yaml
```

#### 7.6.6 Perform a dry run

```bash
vcf package install istio \
  --package istio.kubernetes.vmware.com \
  --version "${ISTIO_VERSION}" \
  -n vmware-system-tkg \
  --values-file istio-data-values.yaml \
  --dry-run
```

#### 7.6.7 Install Istio

```bash
vcf package install istio \
  --package istio.kubernetes.vmware.com \
  --version "${ISTIO_VERSION}" \
  -n vmware-system-tkg \
  --values-file istio-data-values.yaml \
  --wait
```

#### 7.6.8 Verify installation

```bash
kubectl get packageinstall istio \
  -n vmware-system-tkg

kubectl get namespaces
kubectl get all -n istio-ingress
kubectl get all -n istio-system
```

### 7.7 Confirm Istio, cert-manager and the ClusterIssuer are ready

```bash
kubectl get pods -n cert-manager
kubectl get pods -n istio-system
kubectl get pods -n istio-ingress
kubectl get gatewayclass istio
```

Expected result: all Pods `Running`, GatewayClass `istio` is `ACCEPTED=True`, ClusterIssuer is `READY=True`.


### 7.8 External DNS

#### 7.8.1 Cloudflare API token

File `cloudflare-secret.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/cloudflare-secret.yaml

```yaml
stringData:
  # Replace with the Cloudflare API token created in the Cloudflare account.
  # Do NOT commit the real value to Git.
  api-token: <CLOUDFLARE_API_TOKEN>
```
### create  Namespace and Cloudflare Secret 

```bash 
kubectl create namespace tanzu-system-service-discovery
kubectl apply -f cloudflare-secret.yaml

```

#### 7.8.2 Install External DNS

File `external-dns-values.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/external-dns-values.yaml

```bash
# 2. Install the package from the Supervisor context
vcf context use supervisor:<TARGET_VKS_CLUSTER_NAMESPACE>
vcf package install external-dns \
  -p external-dns.kubernetes.vmware.com \
  -v 0.20.0+vmware.1-vks.1 \
  -n vmware-system-tkg \
  --values-file external-dns-values.yaml \
  --wait
```

#### 7.8.3 Create the External DNS ClusterRole and ClusterRoleBinding

**Create the External DNS ClusterRole**

The ClusterRole provides External DNS with read-only permissions to discover Kubernetes resources such as Services, Ingresses, Gateways, and HTTPRoutes.

FILE: `external-dns-cluster-roles.yaml`

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/external-dns-cluster-roles.yaml

**Create the External DNS ClusterRoleBinding**

The ClusterRoleBinding assigns these permissions to the External DNS ServiceAccount so it can monitor these resources and manage the corresponding DNS records.

FILE: `external-dns-clusterrolebinding.yaml`

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/external-dns-clusterrolebinding.yaml

```bash
kubectl apply -f  external-dns-cluster-roles.yaml
kubectl apply -f external-dns-clusterrolebinding.yaml
```

#### 7.8.4 Restart external-dns and Monitor Logs

```bash
kubectl rollout restart deployment external-dns -n tanzu-system-service-discovery
kubectl rollout status deployment external-dns -n tanzu-system-service-discovery
kubectl get pods -n tanzu-system-service-discovery
kubectl logs -n tanzu-system-service-discovery  -l app=external-dns -f
```

#### 7.8.5 Confirm HTTPRoute permissions

```bash
kubectl auth can-i get httproutes.gateway.networking.k8s.io \
  --as=system:serviceaccount:tanzu-system-service-discovery:external-dns
kubectl auth can-i list httproutes.gateway.networking.k8s.io \
  --as=system:serviceaccount:tanzu-system-service-discovery:external-dns
kubectl auth can-i watch httproutes.gateway.networking.k8s.io \
  --as=system:serviceaccount:tanzu-system-service-discovery:external-dns
```

Expected result: `yes` for all three.

#### 7.8.6 Validation: External DNS

```bash
kubectl get pods -n tanzu-system-service-discovery
kubectl get packageinstall -n vmware-system-tkg      # run in the Supervisor context
kubectl logs -n tanzu-system-service-discovery -l app.kubernetes.io/name=external-dns
```

Expected result: the Pod is `Running` and the logs show no errors. If the label returns nothing, check the real labels with `kubectl get pods -n tanzu-system-service-discovery --show-labels`. The old document used `app.kubernetes.io/name=external-dns` and the Istio runbook used `app=external-dns` (TO VERIFY).

## 8. Install Tekton Pipelines

Create namespace for tekton pipline

```bash
kubectl create namespace tekton-pipelines
```

Add the label to namespace

```bash
kubectl label namespace tekton-pipelines pod-security.kubernetes.io/enforce=privileged --overwrite
```

verify tekton-pipelines namespace

```bash
kubectl get namespace tekton-pipelines --show-labels
kubectl get namespace tekton-pipelines -o jsonpath='{.metadata.labels.pod-security\.kubernetes\.io/enforce}{"\n"}'
```

Expected output for the last command

```text
privileged
```

```bash
kubectl apply --filename https://infra.tekton.dev/tekton-releases/pipeline/latest/release.yaml
```

Many resources are created (CRDs, the `tekton-pipelines` namespace, service accounts, cluster roles, controller and webhook deployments). Wait until the Pods are ready.

### 8.1 Validation: Tekton Pipelines

```bash
kubectl get pods -n tekton-pipelines
kubectl get crd | grep tekton
kubectl api-resources | grep tekton
kubectl logs deployment/tekton-pipelines-controller -n tekton-pipelines --tail=50
```

```text
Namespace tekton-pipelines    → Active
tekton-pipelines-controller   → 1/1 Running
tekton-pipelines-webhook      → 1/1 Running
CRDs                          → tasks, taskruns, pipelines, pipelineruns (tekton.dev) present
Controller logs               → no ERROR / failed / panic / connection refused
```

## 9. Create the cicd Namespace with its Security Label

The namespace and its label are one setup step. A namespace without the label causes avoidable build-workload failures later.

```bash
# 1. Check
kubectl get namespace cicd

# 2. Create only if it does not exist
kubectl create namespace cicd

# 3. Apply the label immediately
kubectl label namespace cicd \
  pod-security.kubernetes.io/enforce=privileged \
  --overwrite
```

**Security note:** this label enables privileged capabilities for build workloads. In production, use the least-privilege settings your VKS, BuildKit and Buildpacks implementation and security policy support.

### 9.1 Validation

```bash
kubectl get namespace cicd --show-labels
```

Expected result: `Active`, with `pod-security.kubernetes.io/enforce=privileged` in the labels.

## 10. Install Tekton Dashboard

```bash
kubectl apply -f https://infra.tekton.dev/tekton-releases/dashboard/latest/release.yaml
```

The Dashboard is installed into `tekton-pipelines`. It stays reachable only inside the cluster until Sections 7.8 and 11 to 13 are complete.

### 10.1 Validation: Tekton Dashboard

```bash
kubectl get pods,deployment,svc -n tekton-pipelines | grep dashboard
```

Expected result: `tekton-dashboard` Pod and Deployment `1/1 Running`; Service `tekton-dashboard` on port `9097`.

## 11. Istio Integration: Expose the Dashboard

All three resources are created in `tekton-pipelines`, the namespace of the `tekton-dashboard` Service, so no cross-namespace reference is needed. Apply them in this order: Certificate, Gateway, HTTPRoute. All files are in `manifest/tekton-catalyst`.

### 11.1 TLS certificate (cert-manager)

File `tekton-dashboard-certificate.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/tekton-dashboard-certificate.yaml

| Important argument | Value |
|---|---|
| issuerRef | ClusterIssuer `letsencrypt-prod` |
| dnsNames | `tekton.k8slabs.com` |
| secretName | `tekton-dashboard-tls-cert` (created by cert-manager, used by the Gateway) |

```bash
kubectl apply -f tekton-dashboard-certificate.yaml
kubectl get certificate -n tekton-pipelines     # wait for READY=True
```

### 11.2 Gateway

File `tekton-dashboard-gateway.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/tekton-dashboard-gateway.yaml

| Important argument | Value |
|---|---|
| gatewayClassName | `istio` |
| Listener | HTTPS, port 443, `tls.mode: Terminate` |
| hostname | `tekton.k8slabs.com` |
| certificateRefs | Secret `tekton-dashboard-tls-cert` (from Section 11.1) |
| allowedRoutes | Same namespace only |

```bash
kubectl apply -f tekton-dashboard-gateway.yaml
kubectl get gateway tekton-dashboard-gateway -n tekton-pipelines
```

Expected result: `PROGRAMMED=True` with an `ADDRESS`. External DNS publishes this IP.

```bash
kubectl get gateway tekton-dashboard-gateway -n tekton-pipelines \
  -o jsonpath='{.status.addresses[0].value}{"\n"}'
```

### 11.3 HTTPRoute

File `tekton-dashboard-httproute.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/tekton-dashboard-httproute.yaml

| Important argument | Value |
|---|---|
| parentRefs | Gateway `tekton-dashboard-gateway` |
| hostnames | `tekton.k8slabs.com` (External DNS reads this value) |
| backendRefs | Service `tekton-dashboard`, port `9097` |

```bash
kubectl apply -f tekton-dashboard-httproute.yaml
```

External DNS takes the hostname from `spec.hostnames` and the IP from the Gateway status. No extra annotation is strictly required for the core design. However, per lab convention, apply the necessary external DNS proxy annotation and record it:

```bash
kubectl annotate httproute tekton-dashboard-route -n tekton-pipelines external-dns.alpha.kubernetes.io/cloudflare-proxied="false"
```

**External DNS Cloudflare Proxy Configuration**

```yaml
annotations:
  external-dns.alpha.kubernetes.io/cloudflare-proxied: "false"
spec:
  parentRefs:
    - name: tekton-dashboard-gateway
```

This annotation tells ExternalDNS not to enable Cloudflare's proxy for `tekton.k8slabs.com`. Setting it to `"false"` ensures that the DNS record remains DNS-only, allowing traffic to connect directly to the configured endpoint instead of passing through Cloudflare's proxy servers.

### 11.4 Validation: Istio routing

```bash
kubectl get certificate,gateway,httproute -n tekton-pipelines
kubectl describe httproute tekton-dashboard-route -n tekton-pipelines
```

Expected result: Certificate `READY=True`, Gateway `PROGRAMMED=True`, and HTTPRoute conditions `Accepted=True` and `ResolvedRefs=True`.

## 12. External DNS Reconciliation

### 12.1 Rolling restart (when required)

If the record is not created after the HTTPRoute is applied, restart External DNS so it re-reads the resources:

```bash
kubectl rollout restart deployment external-dns -n tanzu-system-service-discovery
kubectl rollout status deployment external-dns -n tanzu-system-service-discovery

kubectl get pods -n tanzu-system-service-discovery
kubectl logs -n tanzu-system-service-discovery -l app.kubernetes.io/name=external-dns --tail=50
```

Confirm the deployment name (`kubectl get deployment -n tanzu-system-service-discovery`) and whether the restart is needed in the validated sequence (TO VERIFY).

Expected result: a log line showing the record for `tekton.k8slabs.com` being created or updated.

### 12.2 Verify the DNS record

```bash
dig +short tekton.k8slabs.com
```

Expected result: the IP printed equals the Gateway address from Section 11.2. Also confirm in Cloudflare that the A record and its TXT ownership record exist.

## 13. Dashboard Access

Open the Dashboard:

```text
https://tekton.k8slabs.com
```
The browser must show a valid certificate and the Tekton Dashboard.

<img width="1911" height="980" alt="image (2)" src="https://github.com/user-attachments/assets/6c03b674-69dc-43c0-93ca-ff335acdf7e8" />


**Production:** the Dashboard can act on the cluster. Add authentication, RBAC and network restrictions before wider use.

## 14. Smoke Test: Task and TaskRun

This minimal test proves Tekton itself works. Application Tasks (clone, build, sign, update) start in Section 16.

| File | Purpose |
|---|---|
| `test-task.yaml` | Task that prints a test message |
| `taskrun.yaml` | TaskRun `hello-taskrun` that executes the Task |

Task: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/test-task/test-task.yaml

TaskRun: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/test-task/taskrun.yaml

```bash
kubectl apply -f test-task.yaml
kubectl apply -f taskrun.yaml
```

### 14.1 Validation

```bash
kubectl get task,taskrun -n cicd
kubectl logs -l tekton.dev/taskRun=hello-taskrun -n cicd
```

Screenshot from 2026-10-08 14-57-10

In the Dashboard: Namespace → `cicd`. The Task and TaskRun are visible.

<img width="1873" height="636" alt="Screenshot from 2026-10-08 14-57-10" src="https://github.com/user-attachments/assets/39199f3e-b49f-4af3-8c25-05d1ec498786" />


## 15. Installation Validation and Final Verification

Run once at the end. Every line must pass.

| Area | Command | Expected |
|---|---|---|
| Cluster | `kubectl get nodes -o wide` | All nodes Ready |
| Tekton | `kubectl get pods -n tekton-pipelines` | Controller, webhook, dashboard Running |
| Tekton CRDs | `kubectl get crd \| grep tekton` | tekton.dev CRDs present |
| Namespace | `kubectl get namespace cicd --show-labels` | Active, enforce=privileged |
| Dashboard | `kubectl get pods,svc -n tekton-pipelines \| grep dashboard` | 1/1 Running, port 9097 |
| External DNS | `kubectl get pods -n tanzu-system-service-discovery` | Running, no log errors |
| Istio | `kubectl get certificate,gateway,httproute -n tekton-pipelines` | Ready, Programmed, Accepted |
| DNS | `dig +short tekton.k8slabs.com` | Gateway IP |
| User access | Open `https://tekton.k8slabs.com` | Dashboard loads |

## 16. Tekton Pipeline Tasks

Create each Task below by applying its manifest from `manifest/tekton-catalyst` (Figure 1). Each Task lists its important arguments. A single validation at the end of this section checks all of them.

### 16.1 git-clone-update

`git-clone-update` is a Tekton Task used to clone source code from a Git repository into a shared workspace. It cleans the workspace, clones the required branch/revision, and validates the Git repository. The cloned source code is then used by BuildKit or Buildpacks to build the container image.

File `git-clone-update-task.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/git-clone-update-task.yaml

| Important argument | Meaning |
|---|---|
| Repository URL | Git repository to clone (`REPO_URL` from the TriggerBinding in Section 20) |
| Revision | Branch or revision to clone (`main`) |
| Workspace `source` | Shared workspace that receives the cloned code |
| Credentials | `username` and `token` keys of Secret `repo-git-credentials` (Section 18) |

```bash
kubectl apply -f git-clone-update-task.yaml
```

### 16.2 detect-build-type

`detect-build-type` is a Tekton Task that checks whether the source repository contains a Dockerfile. It selects BuildKit when a Dockerfile exists, otherwise it selects Cloud Native Buildpacks. The selected build type is stored as a Tekton result for the next pipeline task to use.

File `detect-build-type.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/detect-build-type.yaml

| Important argument | Meaning |
|---|---|
| Workspace `source` | Cloned source code that is checked |
| Check | Dockerfile present → BuildKit, absent → Buildpacks |
| Result | Selected build type, read by the Pipeline `when` conditions (Section 17) |

```bash
kubectl apply -f detect-build-type.yaml
```

### 16.3 buildpacks-phases

A ready-made Tekton Task that builds an image from source code using Cloud Native Buildpacks. It builds the image when there is no Dockerfile.

File `buildpacks-phase-task.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/buildpacks-phase-task.yaml

| Important argument | Meaning |
|---|---|
| Builder image | Buildpacks builder image, passed by the PipelineRun (Section 19) |
| Workspaces `source`, `cache` | Source code and per-run build cache PVCs |
| Result | `APP_IMAGE_DIGEST` |
| Digest file | `$(workspaces.source.path)/image-digest` |

```bash
kubectl apply -f buildpacks-phase-task.yaml -n cicd
```

### 16.4 buildkit-build

`buildkit-build` is a Tekton Task that runs when `detect-build-type` detects a Dockerfile, using BuildKit to build the application container image. It takes the source code from the shared workspace, builds the image from the Dockerfile, and pushes it to Harbor with the configured credentials.

File `buildkit-build-task-update.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/buildkit-build-task-update.yaml

| Important argument | Meaning |
|---|---|
| Harbor image | Target image in Harbor, passed by the PipelineRun |
| Workspace `dockerconfig` | Harbor credentials (Secret `harbor-registry-secret`) |
| Harbor CA | ConfigMap `harbor-ca-cert` mounted through `taskRunSpecs` in the PipelineRun |
| Registry setting | References the Harbor CA. Does not use `registry.insecure=true` |
| Result | `IMAGE_DIGEST` |
| Digest file | `$(workspaces.source.path)/image-digest` |

```bash
kubectl apply -f buildkit-build-task-update.yaml
```

### 16.5 sign-image (SBOM + Cosign)

This `sign-image` Task is the security/signing stage after BuildKit or Buildpacks. Its job is to generate an SBOM, sign the image with Cosign, attach the SBOM as an attestation, and return the image digest.

File `sign-image-task-update.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/sign-image-task-update.yaml

| Important argument | Meaning |
|---|---|
| `COSIGN_TLOG_UPLOAD` | Defaults to `false` for the air-gapped/private-registry lab flow. Set to `true` only when the cluster can reach the approved Sigstore transparency-log service |
| Image digest | Digest from the build Task. The image is signed by digest, not by tag |
| Workspace `cosign-key` | Cosign key pair (Secret `cosign-key`) |
| Cosign password | Secret `cosign-password` |
| SBOM | Generated with Syft, stored in the `source` workspace, attached as an `spdxjson` attestation |
| Output | Image digest, passed to `update-values` |

```bash
kubectl apply -f sign-image-task-update.yaml
```

### 16.6 update-values

`update-values` is the GitOps update stage of the Tekton pipeline. It updates the Helm chart's `values.yaml` with the newly built image and its digest, commits the change, and pushes it back to the repository.

File `update-values.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/update-values.yaml

| Important argument | Meaning |
|---|---|
| Image digest | Received from the `sign-image` Task |
| Target file | `helm-charts/values.yaml` |
| Commit message prefix | `ci: update TravelPortal image digest` (the EventListener CEL filter ignores commits with this prefix) |
| Credentials | `username` and `token` keys of Secret `repo-git-credentials` |

```bash
kubectl apply -f update-values.yaml
```

### 16.7 Validation – Tasks

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

## 17. Pipeline

Tasks are added to the Pipeline using `taskRef`, which references previously created Tekton Tasks. `params` pass required values to each Task, `workspaces` share files/credentials, and `runAfter`/`when` control the execution order and conditions. The Pipeline first clones the repository, then detects whether a Dockerfile exists and runs BuildKit or Buildpacks accordingly. After the image is built, the sign Task generates the SBOM and signs/attests the image. Finally, the `update-values` Task receives the image digest from the sign Task, updates `values.yaml`, commits the change, and pushes it back to the same Git repository—completing the GitOps update.

File `travelportal-pipeline-values-update.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/travelportal-pipeline-values-update.yaml

| Important argument | Meaning |
|---|---|
| Pipeline name | `travelportal-pipeline-values-update` |
| `taskRef` | The six Tasks from Section 16 |
| Order | clone → detect → (BuildKit or Buildpacks) → sign → update-values |
| `when` | Selects BuildKit or Buildpacks from the `detect-build-type` result |
| `runAfter` | Sets the order of the Tasks |
| Workspaces | `source`, `cache`, `dockerconfig`, `git-credentials`, `cosign-key` |

```bash
kubectl apply -f travelportal-pipeline-values-update.yaml
```

The Pipeline is validated together with the PipelineRun in Section 19.

## 18. Pipeline Dependencies (Storage, ConfigMap, Secrets, Service Account)

Before triggering the PipelineRun, create the required storage, configuration, and credentials. A single validation at the end of this section checks everything.

### 18.1 Source and Build Cache Storage

The PipelineRun creates dedicated `source` and `cache` PVCs for each run using the `lab-gold-storage-policy` StorageClass. This prevents concurrent PipelineRuns from overwriting the same shared source workspace.

If the StorageClass has a different name in your VKS environment, replace `lab-gold-storage-policy` in the PipelineRun and TriggerTemplate workspace definitions.

### 18.2 Harbor CA ConfigMap

Create a ConfigMap named `harbor-ca-cert` in the `cicd` namespace from your Harbor / vCenter CA certificate (the key must be `ca.crt`):

```bash
kubectl create configmap harbor-ca-cert \
  -n cicd \
  --from-file=ca.crt=<path-to-harbor-ca.crt>
```

### 18.3 Secrets

The pipeline needs four secrets in the `cicd` namespace. Create them one by one. Replace every placeholder value with your real credentials before applying, and never commit real credentials to Git.

#### 18.3.1 Git credentials Secret (repo-git-credentials)

Git repository username and token, used by the clone and `update-values` Tasks.

File `github-push-secret.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/github-push-secret.yaml

| Important argument | Value |
|---|---|
| Secret name | `repo-git-credentials` |
| Keys | `username`, `token` |
| Token | Personal Access Token from the repository server (placeholder in Git) |

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

#### 18.3.2 Harbor registry Secret (harbor-registry-secret)

Harbor Docker config (`dockerconfigjson`), used to push and sign images.

File `harbor-credentials.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/harbor-credentials.yaml

| Important argument | Value |
|---|---|
| Secret name | `harbor-registry-secret` |
| Type | `dockerconfigjson` |
| Used by | Workspace `dockerconfig` |

```bash
kubectl apply -f harbor-credentials.yaml
```

#### 18.3.3 Cosign password Secret (cosign-password)

Password of the Cosign private key.

File `cosign-pass.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/cosign-pass.yaml

| Important argument | Value |
|---|---|
| Secret name | `cosign-password` |
| Content | Password of the Cosign private key (placeholder in Git) |

```bash
kubectl apply -f cosign-pass.yaml
```

#### 18.3.4 Cosign key pair Secret (cosign-key)

Cosign private and public key. Create it from the generated key pair (no manifest file):

```bash
kubectl create secret generic cosign-key \
  -n cicd \
  --from-file=cosign.key=cosign.key \
  --from-file=cosign.pub=cosign.pub
```

### 18.4 ServiceAccount

The TriggerTemplate and PipelineRun use the dedicated `tekton-pipeline-sa` ServiceAccount.

File `tekton-pipeline-sa.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/tekton-pipeline-sa.yaml

| Important argument | Value |
|---|---|
| ServiceAccount | `tekton-pipeline-sa` in `cicd` |
| Used by | PipelineRun and TriggerTemplate |

```bash
kubectl apply -f tekton-pipeline-sa.yaml
```

### 18.5 Validation – Dependencies

```bash
kubectl get storageclass lab-gold-storage-policy
kubectl get configmap harbor-ca-cert -n cicd
kubectl get secret repo-git-credentials harbor-registry-secret cosign-password cosign-key -n cicd
kubectl get sa tekton-pipeline-sa -n cicd
```

Expected result: every object is found, with no `NotFound` errors.

## 19. PipelineRun

The PipelineRun is used to start and execute the `travelportal-pipeline-values-update` Pipeline. It provides the pipeline parameters, connects the required workspaces and secrets, and applies additional Pod configuration needed during the pipeline execution. The `pipelineRef` selects the `travelportal-pipeline-values-update` Pipeline, while `params` provide the Git repository, branch, Harbor image, and Buildpacks builder image that the Pipeline will use. The workspaces provide the required storage and credentials: the `source` and `cache` PVCs are created per PipelineRun, `dockerconfig` provides Harbor authentication, `git-credentials` provides repository push credentials, and `cosign-key` provides the image-signing key. The SBOM is stored in the `source` workspace, so no separate SBOM workspace is required. The `taskRunSpecs` customize the BuildKit TaskRun by adding the `harbor-ca` ConfigMap as a volume. This allows the BuildKit Pod to access the Harbor CA certificate. The BuildKit registry configuration references that CA and does not use `registry.insecure=true` for the Harbor HTTPS endpoint.

File `travelportal-pipelinerun-pack-values-update.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/travelportal-pipelinerun-pack-values-update.yaml

| Important argument | Meaning |
|---|---|
| `pipelineRef` | `travelportal-pipeline-values-update` |
| `serviceAccountName` | `tekton-pipeline-sa` |
| `params` | Git repository, branch, Harbor image, Buildpacks builder image |
| Workspace `source`, `cache` | PVCs created per run from StorageClass `lab-gold-storage-policy` |
| Workspace `dockerconfig` | Secret `harbor-registry-secret` |
| Workspace `git-credentials` | Secret `repo-git-credentials` |
| Workspace `cosign-key` | Secret `cosign-key` |
| `taskRunSpecs` | Adds ConfigMap `harbor-ca-cert` as a volume to the BuildKit TaskRun |

```bash
kubectl apply -f travelportal-pipelinerun-pack-values-update.yaml
```

### 19.1 Validation – Pipeline, PipelineRun, and output

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
cosign verify \
  --offline \
  --key cosign.pub \
  "lab25-harbor.lab25.sunfire.lab/cicd/travelportal@sha256:<IMAGE_DIGEST>"

cosign verify-attestation \
  --offline \
  --key cosign.pub \
  --type spdxjson \
  "lab25-harbor.lab25.sunfire.lab/cicd/travelportal@sha256:<IMAGE_DIGEST>"
```

Expected result: the image pulls, the signature verifies, and the SBOM attestation is associated with the image digest.

## 20. Automatic Pipeline Trigger with Tekton Triggers

Until now the PipelineRun was started by hand with `kubectl apply`. In this section a push to the repository starts it automatically. The repository server sits on the internal lab network, so it calls the Tekton EventListener NodePort directly.

### 20.1 Lab values used in this section

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

### 20.2 The Tekton Triggers objects created in this section

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

The EventListener/Trigger object names retain repository terminology. The names do not change EventListener functionality; the repository endpoint used by the binding is the internal repository server.

### 20.3 Install Tekton Triggers

For an Internet-connected VKS cluster, install the Tekton Triggers release before creating the EventListener, TriggerBinding, and TriggerTemplate objects.

```bash
kubectl apply --filename https://infra.tekton.dev/tekton-releases/triggers/latest/release.yaml
```

For production, pin the Triggers version that is tested with your selected Tekton Pipelines release instead of using `latest`.

#### 20.3.1 Validation – Tekton Triggers

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

### 20.4 EventListener ServiceAccount and RBAC

The EventListener runs with `serviceAccountName: repository-trigger-sa`. It needs permission to read the Triggers resources in `cicd`, read the cluster-scoped Triggers resources (`clusterinterceptors`, `clustertriggerbindings`), and create PipelineRuns.

#### 20.4.1 ServiceAccount

File `webhook-sa.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/webhook-sa.yaml

| Important argument | Value |
|---|---|
| ServiceAccount | `repository-trigger-sa` in `cicd` |

```bash
kubectl apply -f webhook-sa.yaml
```

#### 20.4.2 Role, RoleBinding, ClusterRole and ClusterRoleBinding

File `eventlistener-rbac.yaml` contains the Role (`repository-trigger-role`), RoleBinding (`repository-trigger-rolebinding`), ClusterRole (`repository-trigger-cluster-role`) and ClusterRoleBinding (`repository-trigger-cluster-rolebinding`).

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/eventlistener-rbac.yaml

| Important argument | Meaning |
|---|---|
| Role (namespace `cicd`) | Read `triggerbindings`, `triggertemplates`, `eventlisteners`; create `pipelineruns` |
| ClusterRole | Read `clusterinterceptors`, `clustertriggerbindings` |
| Subject | ServiceAccount `repository-trigger-sa` |

```bash
kubectl apply -f eventlistener-rbac.yaml
```

#### 20.4.3 Validation – RBAC

```bash
kubectl get sa repository-trigger-sa -n cicd
kubectl get role,rolebinding -n cicd | grep repository-trigger
kubectl get clusterrole,clusterrolebinding | grep repository-trigger

kubectl auth can-i list triggerbindings.triggers.tekton.dev \
  --as=system:serviceaccount:cicd:repository-trigger-sa -n cicd
kubectl auth can-i list triggertemplates.triggers.tekton.dev \
  --as=system:serviceaccount:cicd:repository-trigger-sa -n cicd
kubectl auth can-i list eventlisteners.triggers.tekton.dev \
  --as=system:serviceaccount:cicd:repository-trigger-sa -n cicd

kubectl auth can-i list clusterinterceptors.triggers.tekton.dev \
  --as=system:serviceaccount:cicd:repository-trigger-sa
kubectl auth can-i list clustertriggerbindings.triggers.tekton.dev \
  --as=system:serviceaccount:cicd:repository-trigger-sa

kubectl auth can-i create pipelineruns.tekton.dev \
  --as=system:serviceaccount:cicd:repository-trigger-sa -n cicd
```

Expected result: all objects exist and every `can-i` answers `yes`.

### 20.5 TriggerBinding, TriggerTemplate and EventListener

Create the three objects below, then validate them once at the end.

#### 20.5.1 TriggerBinding

File `gitea-triggerbinding.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/gitea-triggerbinding.yaml

| Important argument | Value |
|---|---|
| Name | `travelportal-repository-binding` |
| `REPO_URL` | `http://10.12.90.62/admin/travelPortal-test-buildpack.git` |
| `REVISION` | `main` |
| Payload fields read | `body.ref`, `body.after`, `body.head_commit.message` |

```bash
kubectl apply -f gitea-triggerbinding.yaml
```

#### 20.5.2 TriggerTemplate

File `gitea-triggertemplate.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/gitea-triggertemplate.yaml

| Important argument | Value |
|---|---|
| Name | `travelportal-repository-template` |
| `pipelineRef` | `travelportal-pipeline-values-update` |
| `serviceAccountName` | `tekton-pipeline-sa` |
| Repository URL | `http://10.12.90.62/admin/travelPortal-test-buildpack.git` |
| PipelineRun name | `travelportal-build-xxxxx` (generated) |
| Workspaces | Per-run `source` and `cache` PVCs from StorageClass `lab-gold-storage-policy` |

```bash
kubectl apply -f gitea-triggertemplate.yaml
```

#### 20.5.3 EventListener

File `gitea-eventlistener.yaml`:

YAML: https://github.com/kondurupurandhar/keda-vks/blob/main/manifest/tekton-catalyst/gitea-eventlistener.yaml

| Important argument | Value |
|---|---|
| Name | `travelportal-repository-listener` |
| `serviceAccountName` | `repository-trigger-sa` |
| Interceptor | Cluster-scoped `cel` interceptor (main branch only, ignores CI commits) |
| Binding / Template | `travelportal-repository-binding` / `travelportal-repository-template` |
| `serviceType` | `NodePort` |
| Application port | `8080` (observed NodePort in this lab: `31877`) |

```bash
kubectl apply -f gitea-eventlistener.yaml
```

Because the manifest specifies only `serviceType: NodePort`, Kubernetes may assign a different NodePort in another environment, so always confirm the actual value in the validation below.

#### 20.5.4 Validation – TriggerBinding, TriggerTemplate and EventListener

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

### 20.6 Add the webhook in the repository

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
| Event | Push events |
| Branch | `main` |

Save the webhook. The values read from the push payload are:

```text
body.ref
body.after
body.head_commit.message
```

Troubleshooting: by default the repository server blocks outbound webhook calls to hosts that are not on its allow list. If the webhook fails with the message below, add the Tekton node to the allowed host list setting named in the error (TO VERIFY the exact setting on your repository server).

```text
webhook can only call allowed HTTP servers
(check your security.ALLOWED_HOST_LIST setting)
```

### 20.7 Pipeline changes for automatic runs

#### 20.7.1 Image digest

BuildKit and Buildpacks return their digest under different result names:

| Build | Result |
|---|---|
| BuildKit | `IMAGE_DIGEST` |
| Buildpacks | `APP_IMAGE_DIGEST` |

Both Tasks also write the same file to the shared workspace:

```text
$(workspaces.source.path)/image-digest
```

#### 20.7.2 Repository credentials for update-values

The clone and `update-values` Tasks read the repository credentials from the `username` and `token` keys of the `repo-git-credentials` Secret (created in Section 18.3.1). The token is a Personal Access Token created in the repository server.

## 21. Final Validation – End to End

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
| Pipeline triggers itself repeatedly | The CI commit message does not match the prefix used by the CEL filter | Ensure `update-values` commits with `ci: update TravelPortal image digest` and the EventListener filters the same prefix |

## 22. Open Verification Items

| # | Item | Action before freezing |
|---|---|---|
| 1 | Offline / Harbor tag format | Out of scope here. Validate the Harbor-accepted tag and digest format for offline artifacts; do not document an unverified syntax |
| 2 | Tekton versions | Record the exact Pipelines and Dashboard versions and keep the release copies in the repository |
| 3 | Pinned URL pattern | Confirm `previous/<version>/release.yaml` resolves for the chosen versions |
| 4 | Repository restructuring | Files are still flat in `manifest/tekton-catalyst` (Figure 1). Apply the planned `manifest/tekton/` and `manifest/external-dns/` split, then update every link |
| 5 | External DNS install sequence and context switching | Re-run Section 7.8.2 and confirm the order and the Supervisor vs workload contexts |
| 6 | External DNS Pod label and Deployment name | Confirm `app` vs `app.kubernetes.io/name`, and the Deployment name |
| 7 | `--txt-owner-id` | Decide the per-lab value |
| 8 | Dashboard hostname | Confirm `tekton.k8slabs.com` does not collide with another lab, and that `--domain-filter=k8slabs.com` covers it (the old document used `vks.k8slabs.com`) |
| 9 | HTTPRoute annotation | Confirm whether the lab convention requires an External DNS annotation |
| 10 | Rolling restart | Confirm whether it is needed in the validated sequence |
| 11 | Important arguments (Sections 16 to 20) | Confirm each argument name and value against the YAML; the repository is private and these were taken from this document |
| 12 | `buildpacks-source-pvc.yaml` | Confirm whether it is still needed |
| 13 | Final Dashboard image | Add after the final validation run |

## 23. Conclusion

This implementation provides a Kubernetes-native CI workflow on VKS using Tekton, BuildKit, Buildpacks, Syft, Cosign, and Harbor. The pipeline automatically selects BuildKit or Buildpacks based on the application source, generates an SBOM, signs the immutable image digest, and stores the resulting image and security artifacts in Harbor.

The architecture also separates the application source and internal CI/GitOps responsibilities across two repositories, providing a clear and controlled workflow for source management and deployment configuration.

Overall, the solution provides an automated, traceable, and security-focused CI process that can be operated in both connected and disconnected VKS environments, provided the required images, dependencies, credentials, and registry artifacts are available locally.
