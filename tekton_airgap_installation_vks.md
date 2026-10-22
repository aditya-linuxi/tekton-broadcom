# Tekton Pipelines and Tekton Dashboard Installation on VKS (Air-Gapped Environment)

## 1. Introduction

- **VKS (vSphere Kubernetes Service):** A Kubernetes service provided by VMware Cloud Foundation for creating and managing Kubernetes clusters on vSphere infrastructure.
- **Air-Gapped Environment:** A network-isolated environment with no outbound internet access. All container images, VKS packages, YAML manifests, and TLS certificates are served from internal infrastructure.
- **Tekton Pipelines:** A Kubernetes-native CI/CD framework used to create and run automated pipelines as Kubernetes resources (`Task`, `TaskRun`, `Pipeline`, `PipelineRun`).
- **Tekton Dashboard:** A web-based graphical interface for Tekton Pipelines that shows the real-time status, logs, and details of Tasks, TaskRuns, Pipelines, and PipelineRuns.
- **Harbor (`lab25-harbor.lab25.sunfire.lab`):** The private container registry inside the air-gapped environment. It serves all pre-mirrored Tekton Pipelines, Tekton Dashboard, and base utility images, and stores application images built later by the pipeline.
- **Internal Git Repository (`http://10.12.90.62`):** The internal Gitea server hosting the offline Kubernetes YAML manifests and application source repositories.
- **External DNS / Internal DNS:** Automatically creates or maps internal DNS records (`tekton.k8slabs.com`) for exposed Kubernetes services so users can reach the Tekton Dashboard by hostname.
- **Istio + Kubernetes Gateway API:** Provides the HTTPS entry point (a `Gateway` and an `HTTPRoute`) used to reach the Tekton Dashboard.
- **cert-manager:** Issues the TLS certificate for the Dashboard hostname using an internal/private Certificate Authority (`ClusterIssuer`) rather than a public internet ACME server.

## 2. Architecture

### 2.1 Architecture Diagram and Flow

In an air-gapped VKS environment, all components run inside the VKS cluster and pull container images exclusively from the local Harbor registry (`lab25-harbor.lab25.sunfire.lab`). No external internet endpoints (`infra.tekton.dev`, `ghcr.io`, `docker.io`, or `letsencrypt.org`) are contacted.

```text
  Admin (Jumpbox)                           User (Browser)
      │ 1. kubectl apply local YAMLs            │ https://tekton.k8slabs.com
      ▼                                         ▼
  Internal Git (Gitea)                      Internal DNS / External DNS
  http://10.12.90.62                            │ resolves to Istio Gateway IP
      │                                         ▼
┌─────────────────────────────── VKS Cluster ────────────────────────────────┐
│                                                                            │
│  ┌─ namespace: cert-manager ────────────────────────────────────────────┐  │
│  │ cert-manager ─► Private CA ClusterIssuer ─► issues Dashboard TLS     │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                            │
│  ┌─ namespace: istio-system / istio-ingress ────────────────────────────┐  │
│  │ Istio Ingress Gateway ─► terminates HTTPS for tekton.k8slabs.com     │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                            │
│  ┌─ namespace: tanzu-system-service-discovery ──────────────────────────┐  │
│  │ External DNS ─► publishes Dashboard DNS record to Internal DNS       │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                            │
│  ┌─ namespace: tekton-pipelines ────────────────────────────────────────┐  │
│  │ Tekton Pipelines Controller & Webhook     Tekton Dashboard  ◄── User │  │
│  │ Gateway (HTTPS :443)  ─►  HTTPRoute  ─►   Service (:9097)            │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                            │
│  ┌─ namespace: cicd ────────────────────────────────────────────────────┐  │
│  │ Smoke Test Task (hello) ─► TaskRun (hello-taskrun)                   │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
      ▲
      │ Pulls pre-mirrored Tekton, Dashboard & helper images over internal network
      │
  Harbor Registry (lab25-harbor.lab25.sunfire.lab)
```

| Component / Step | What it means in plain words |
|---|---|
| Local Harbor Registry | Replaces Docker Hub, GHCR, and Google Container Registry; holds all Tekton and Dashboard images |
| Internal Git / Jumpbox | Holds the offline YAML manifests modified to reference Harbor image paths |
| VKS Package Management | Installs `cert-manager`, `istio`, and `external-dns` from the local VCF Supervisor package repository |
| Private CA `ClusterIssuer` | Issues trusted TLS certificates offline without calling Let's Encrypt over the internet |
| Tekton Pipelines | Runs the core CI/CD controller, webhook, and resolvers inside `tekton-pipelines` |
| Tekton Dashboard | Provides the web UI on port `9097`, exposed via Istio Gateway and HTTPRoute at `https://tekton.k8slabs.com` |
| Smoke Test (`cicd`) | Runs a simple `Task` and `TaskRun` using an image from Harbor to prove Tekton works end to end |

### 2.2 Namespaces

| Namespace | Contents |
|---|---|
| `tekton-pipelines` | Tekton Pipelines, Tekton Dashboard, Harbor pull secret, Gateway, Certificate, HTTPRoute |
| `cicd` | Smoke test `Task` and `TaskRun` (and later application CI/CD Pipelines) |
| `tanzu-system-service-discovery` | External DNS |
| `istio-system`, `istio-ingress` | Istio control plane and Ingress Gateway |
| `cert-manager` | cert-manager and Private CA secret / ClusterIssuer |

## 3. Why Use Tekton Pipelines and Tekton Dashboard on Air-Gapped VKS?

In plain terms, here is why this setup works well in a disconnected environment:

**Zero internet dependency.** Every controller, webhook, internal helper container, and dashboard image is pulled directly from your local Harbor registry (`lab25-harbor.lab25.sunfire.lab`).

**Runs natively on VKS.** Tekton installs as standard Kubernetes deployments and CRDs inside your existing VKS cluster, requiring no external CI/CD virtual machines or SaaS runners.

**Uses built-in VKS platform services.** TLS certificates, ingress routing, and DNS are handled by VKS-managed packages (`cert-manager`, `istio`, and `external-dns`) already mirrored in your VCF environment.

**One dashboard for visibility.** Teams can open `https://tekton.k8slabs.com` in their browser to inspect TaskRuns, PipelineRuns, and step logs without needing `kubectl` access to the cluster.

**Ready for offline BuildKit, Buildpacks, and Cosign integration.** Once Tekton Pipelines and Tekton Dashboard are verified with the smoke test, the cluster is ready to plug in air-gapped build and signing tasks.

## 4. Benefits

| Benefit | In simple words |
|---|---|
| Fully offline operation | No outbound calls to `infra.tekton.dev`, `ghcr.io`, `docker.io`, or `letsencrypt.org`. |
| Fast and reliable image pulls | All Tekton system and helper images are pulled over the local network from Harbor. |
| Consistent with connected environments | Uses the exact same Kubernetes CRDs, namespaces, and Gateway API routing as an internet-connected VKS cluster. |
| Visual troubleshooting | The Tekton Dashboard displays step-by-step logs and status in a browser. |
| Secure HTTPS access | Istio Gateway and cert-manager secure the Dashboard with TLS using your internal Certificate Authority. |
| Controlled upgrades | Pinning Tekton and Dashboard manifests and images in Harbor ensures nothing changes unexpectedly. |

## 5. Prerequisites and Air-Gap Resource Plan

### 5.1 Platform and Components

| Component | Requirement |
|---|---|
| VMware Cloud Foundation / VKS | 9.x, target VKS workload cluster available |
| Kubernetes | 1.28 or later (Tekton requirement) |
| VKS Package Repository | Local Supervisor package repo containing `cert-manager`, `istio`, and `external-dns` |
| cert-manager | Installed through VKS package management |
| Istio | Installed through VKS package management, with Gateway API CRDs present |
| ClusterIssuer | Internal CA `ClusterIssuer` (`ca-issuer` or lab pre-configured issuer) |
| External DNS / Internal DNS | VKS package `external-dns.kubernetes.vmware.com` configured for internal DNS (or static internal DNS record for `tekton.k8slabs.com`) |
| Harbor | `lab25-harbor.lab25.sunfire.lab` reachable from the Jumpbox and all VKS worker nodes |
| Harbor CA Trust | VKS worker nodes configured to trust the Harbor TLS CA certificate |
| Internal Git Server | Gitea available at `http://10.12.90.62` |
| Permissions | `cluster-admin` on the VKS cluster |

### 5.2 Required CLI Tools on the Air-Gapped Jumpbox

```bash
vcf version
kubectl version --client
git --version
curl --version
sed --version
```

### 5.3 Air-Gap Resource Plan: Required Harbor Images (Already Pushed)

Confirm that the following Tekton Pipelines, Tekton Dashboard, and smoke-test images are present in your Harbor registry (`lab25-harbor.lab25.sunfire.lab/tekton/...` or your target Harbor project).

> **Important:** Tekton Pipelines uses several helper images passed as command-line arguments to `tekton-pipelines-controller` (such as `entrypoint`, `nop`, `workingdirinit`, `sidecarlogresults`, and `shell-image`). Every one of these must point to Harbor in `tekton-pipeline-release.yaml`, or TaskRuns will fail with `ImagePullBackOff`.

| Component | Purpose | Original Upstream Image | Target Harbor Image Path (Example) |
|---|---|---|---|
| Pipelines Controller | Core Tekton controller | `ghcr.io/tektoncd/pipeline/controller-*` | `lab25-harbor.lab25.sunfire.lab/tekton/controller:<TAG>` |
| Pipelines Webhook | Validation/mutation webhook | `ghcr.io/tektoncd/pipeline/webhook-*` | `lab25-harbor.lab25.sunfire.lab/tekton/webhook:<TAG>` |
| Pipelines Resolvers | Git/Hub/Cluster resolvers | `ghcr.io/tektoncd/pipeline/resolvers-*` | `lab25-harbor.lab25.sunfire.lab/tekton/resolvers:<TAG>` |
| Events Controller | Tekton events controller | `ghcr.io/tektoncd/pipeline/events-*` | `lab25-harbor.lab25.sunfire.lab/tekton/events:<TAG>` |
| Entrypoint | Wraps every Task step execution | `ghcr.io/tektoncd/pipeline/entrypoint-*` | `lab25-harbor.lab25.sunfire.lab/tekton/entrypoint:<TAG>` |
| Nop | Stops sidecars after task completion | `ghcr.io/tektoncd/pipeline/nop-*` | `lab25-harbor.lab25.sunfire.lab/tekton/nop:<TAG>` |
| Workingdirinit | Initializes task working directories | `ghcr.io/tektoncd/pipeline/workingdirinit-*` | `lab25-harbor.lab25.sunfire.lab/tekton/workingdirinit:<TAG>` |
| Sidecarlogresults | Extracts step results from logs | `ghcr.io/tektoncd/pipeline/sidecarlogresults-*` | `lab25-harbor.lab25.sunfire.lab/tekton/sidecarlogresults:<TAG>` |
| Kubeconfigwriter | Writes kubeconfig for cluster tasks | `ghcr.io/tektoncd/pipeline/kubeconfigwriter-*` | `lab25-harbor.lab25.sunfire.lab/tekton/kubeconfigwriter:<TAG>` |
| Imagedigestexporter | Exports built image digests | `ghcr.io/tektoncd/pipeline/imagedigestexporter-*` | `lab25-harbor.lab25.sunfire.lab/tekton/imagedigestexporter:<TAG>` |
| Pullrequest-init | PR resource initialization | `ghcr.io/tektoncd/pipeline/pullrequest-init-*` | `lab25-harbor.lab25.sunfire.lab/tekton/pullrequest-init:<TAG>` |
| Shell Image (Linux) | Runs `script:` blocks inside Tasks | `cgr.dev/chainguard/busybox` | `lab25-harbor.lab25.sunfire.lab/tekton/busybox:latest` |
| Shell Image (Windows) | Windows script runner arg | `mcr.microsoft.com/powershell:nanoserver` | `lab25-harbor.lab25.sunfire.lab/tekton/powershell:nanoserver` |
| Tekton Dashboard | Web UI for Tekton | `ghcr.io/tektoncd/dashboard/dashboard-*` | `lab25-harbor.lab25.sunfire.lab/tekton/dashboard:<TAG>` |
| Smoke Test Image | Runs the `hello` smoke test Task | `docker.io/library/ubuntu` or `busybox` | `lab25-harbor.lab25.sunfire.lab/tekton/busybox:latest` |

### 5.4 Air-Gap Resource Plan: Required Files and Manifests on Jumpbox

| File Name | Purpose | Air-Gap Modification Required |
|---|---|---|
| `harbor-ca.crt` | Root CA certificate of Harbor / vCenter | Used to verify HTTPS connections to Harbor |
| `ca.crt` and `ca.key` | Internal Root CA certificate and private key | Used by `cert-manager` to issue the Dashboard TLS certificate offline |
| `cluster-issuer.yaml` | `ClusterIssuer` definition | Configured with `spec.ca.secretName: ca-key-pair` (instead of Let's Encrypt ACME) |
| `tekton-pipeline-release.yaml` | Tekton Pipelines installation manifest | All `image:` fields and controller `args:` helper image flags updated to `lab25-harbor.lab25.sunfire.lab/tekton/...`, and `@sha256:...` digests removed if tags were re-pushed |
| `tekton-dashboard-release.yaml` | Tekton Dashboard installation manifest | Dashboard `image:` updated to `lab25-harbor.lab25.sunfire.lab/tekton/dashboard:<TAG>` |
| `tekton-dashboard-certificate.yaml` | cert-manager `Certificate` for `tekton.k8slabs.com` | `issuerRef.name` points to the air-gap `ClusterIssuer` |
| `tekton-dashboard-gateway.yaml` | Istio `Gateway` on port 443 | Unchanged from connected setup |
| `tekton-dashboard-httproute.yaml` | `HTTPRoute` pointing to `tekton-dashboard:9097` | Unchanged from connected setup |
| `test-task.yaml` | Smoke test `Task` | Step `image:` updated to `lab25-harbor.lab25.sunfire.lab/tekton/busybox:latest` (or `ubuntu`) |
| `taskrun.yaml` | Smoke test `TaskRun` (`hello-taskrun`) | Unchanged from connected setup |

## 6. Manifest Repository

In the air-gapped environment, GitHub (`github.com`) is not reachable from the jumpbox. Store and clone all manifests from your internal Gitea server (`http://10.12.90.62`) or a local directory on the jumpbox.

### 6.1 Repository Locations

| Item | Internal Air-Gap Location | Original Reference |
|---|---|---|
| Tekton installation, Istio Gateway, and DNS manifests | `http://10.12.90.62/admin/keda-vks/src/branch/main/manifest/tekton-catalyst` | `manifest/tekton-catalyst` |
| Smoke test Task / TaskRun manifests | `http://10.12.90.62/admin/keda-vks/src/branch/main/manifest/test-task` | `manifest/test-task` |

Clone the repository onto your jumpbox before starting:

```bash
git clone http://10.12.90.62/admin/keda-vks.git
cd keda-vks/manifest/tekton-catalyst
```

## 7. Cluster Pre-checks

### 7.1 Access the VKS Cluster

```bash
vcf context list
vcf context use <TARGET_VKS_CLUSTER_NAME>
```

### 7.2 Validate Cluster and Permissions

```bash
kubectl config current-context        # confirm the right cluster
kubectl cluster-info
kubectl version
kubectl get nodes -o wide             # all nodes Ready

kubectl auth can-i create customresourcedefinitions   # expected: yes
kubectl auth can-i create clusterroles                # expected: yes
kubectl auth can-i create namespaces                  # expected: yes
```

### 7.3 Air-Gap Internal Connectivity Pre-checks

In an air-gapped environment, external URLs (`infra.tekton.dev`, `registry-1.docker.io`, `ghcr.io`) are unreachable. Instead, verify HTTPS/HTTP connectivity from your jumpbox to your internal Harbor registry and internal Gitea server:

```bash
# Verify Harbor V2 API is reachable (returns HTTP 200 or 401 Unauthorized)
curl -I --cacert harbor-ca.crt https://lab25-harbor.lab25.sunfire.lab/v2/

# Verify Internal Git server (Gitea) is reachable
curl -I http://10.12.90.62

# Verify VKS Supervisor Package Repository is available in the cluster
kubectl get packagerepository -A
```

### 7.4 Install cert-manager

Install `cert-manager` using VKS Package Management (served from the local VCF Supervisor repository).

#### 7.4.1 Retrieve details about the available cert-manager package

```bash
vcf package available get \
  cert-manager.kubernetes.vmware.com \
  -n vmware-system-tkg
```

#### 7.4.2 Set the Package Version

Confirm the version available in your local VKS repository:

```bash
export CERT_MANAGER_VERSION=1.19.2+vmware.1-vks.1
export CERT_MANAGER_RELEASE_NAME=cert-manager.kubernetes.vmware.com
```

#### 7.4.3 Generate and Review the Data Values

```bash
vcf package available get \
  cert-manager.kubernetes.vmware.com/${CERT_MANAGER_VERSION} \
  -n vmware-system-tkg \
  --default-values-file-output cert-manager-data-values.yaml
```

Verify `cert-manager-data-values.yaml`:

```bash
cat cert-manager-data-values.yaml
```

#### 7.4.4 Perform a Dry Run

```bash
vcf package install cert-manager \
  --package cert-manager.kubernetes.vmware.com \
  --version "${CERT_MANAGER_VERSION}" \
  -n vmware-system-tkg \
  --values-file cert-manager-data-values.yaml \
  --dry-run
```

#### 7.4.5 Install cert-manager

```bash
vcf package install cert-manager \
  --package cert-manager.kubernetes.vmware.com \
  --version "${CERT_MANAGER_VERSION}" \
  -n vmware-system-tkg \
  --values-file cert-manager-data-values.yaml \
  --wait
```

#### 7.4.6 Verify Installation

```bash
kubectl get packageinstall -n vmware-system-tkg
kubectl get namespaces
kubectl get all -n cert-manager
```

### 7.5 Install Istio Using VKS Package Management

#### 7.5.1 Check available releases

```bash
vcf package available list -A

vcf package available get \
  istio.kubernetes.vmware.com \
  -n vmware-system-tkg
```

#### 7.5.2 Set the Istio version

```bash
export ISTIO_VERSION=1.28.2+vmware.1-vks.1
export ISTIO_RELEASE_NAME=istio.kubernetes.vmware.com
```

#### 7.5.3 Generate the data values file

```bash
vcf package available get \
  istio.kubernetes.vmware.com/${ISTIO_VERSION} \
  -n vmware-system-tkg \
  --default-values-file-output istio-data-values.yaml
cat istio-data-values.yaml
```

#### 7.5.4 Enable the ingress gateway

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

#### 7.5.5 Perform a dry run

```bash
vcf package install istio \
  --package istio.kubernetes.vmware.com \
  --version "${ISTIO_VERSION}" \
  -n vmware-system-tkg \
  --values-file istio-data-values.yaml \
  --dry-run
```

#### 7.5.6 Install Istio

```bash
vcf package install istio \
  --package istio.kubernetes.vmware.com \
  --version "${ISTIO_VERSION}" \
  -n vmware-system-tkg \
  --values-file istio-data-values.yaml \
  --wait
```

#### 7.5.7 Verify installation

```bash
kubectl get packageinstall istio \
  -n vmware-system-tkg

kubectl get namespaces
kubectl get all -n istio-ingress
kubectl get all -n istio-system
```

### 7.6 Confirm Istio and cert-manager are ready

```bash
kubectl get pods -n cert-manager
kubectl get pods -n istio-system
kubectl get pods -n istio-ingress
```

Expected result: all Pods `Running`.

### 7.7 External DNS (Internal DNS Integration)

> **Air-Gap Note:** If your air-gapped environment uses an internal DNS provider supported by External DNS (such as RFC2136 / BIND, CoreDNS, or Infoblox), configure `external-dns-values.yaml` for that internal provider below. If you manage DNS manually in your internal lab DNS server (e.g., Windows DNS / dnsmasq), you can skip to Section 8 and create a static `A` record for `tekton.k8slabs.com` pointing to the Istio Gateway IP in Section 11.

#### 7.7.1 Create Namespace

Create the `tanzu-system-service-discovery` namespace to deploy and manage External DNS:

```bash
kubectl create namespace tanzu-system-service-discovery
```

#### 7.7.2 Add label to tanzu-system-service-discovery

```bash
kubectl label namespace tanzu-system-service-discovery pod-security.kubernetes.io/enforce=privileged --overwrite
```

#### 7.7.3 Create DNS Provider Secret (If Required by Internal DNS Provider)

File `dns-secret.yaml` (or `cloudflare-secret.yaml` if running in a hybrid DNS lab):

```bash
kubectl apply -f dns-secret.yaml
```

#### 7.7.4 Install External DNS

##### 7.7.4.1 Define the ExternalDNS Version Variable

```bash
EXTERNAL_DNS_VERSION="0.20.0+vmware.1-vks.1"
EXTERNAL_DNS_PACKAGE="external-dns.kubernetes.vmware.com"
```

##### 7.7.4.2 Get the Available ExternalDNS Package

```bash
vcf package available get \
  external-dns.kubernetes.vmware.com \
  -n vmware-system-tkg
```

##### 7.7.4.3 Export Default Values to the Values File

```bash
vcf package available get \
  "external-dns.kubernetes.vmware.com/$EXTERNAL_DNS_VERSION" \
  -n vmware-system-tkg \
  --default-values-file-output external-dns-values.yaml
```

##### 7.7.4.4 Configure `external-dns-values.yaml`

Update `external-dns-values.yaml` with your internal DNS server parameters (`rfc2136` / internal provider) and target domain filter (`k8slabs.com`).

##### 7.7.4.5 Perform dry-run

```bash
vcf package install external-dns \
  -p external-dns.kubernetes.vmware.com \
  -v "$EXTERNAL_DNS_VERSION" \
  -n vmware-system-tkg \
  --values-file external-dns-values.yaml \
  --dry-run
```

##### 7.7.4.6 Install the ExternalDNS Package

```bash
vcf package install external-dns \
  -p external-dns.kubernetes.vmware.com \
  -v "$EXTERNAL_DNS_VERSION" \
  -n vmware-system-tkg \
  --values-file external-dns-values.yaml \
  --wait
```

#### 7.7.5 Create the External DNS ClusterRole and ClusterRoleBinding

FILE: `external-dns-cluster-roles.yaml`
FILE: `external-dns-clusterrolebinding.yaml`

```bash
kubectl apply -f external-dns-cluster-roles.yaml
kubectl apply -f external-dns-clusterrolebinding.yaml
```

#### 7.7.6 Restart external-dns and Monitor Logs

```bash
kubectl rollout restart deployment external-dns -n tanzu-system-service-discovery
kubectl rollout status deployment external-dns -n tanzu-system-service-discovery
kubectl get pods -n tanzu-system-service-discovery
kubectl logs -n tanzu-system-service-discovery -l app=external-dns --tail=50
```

#### 7.7.7 Confirm HTTPRoute permissions

```bash
kubectl auth can-i get httproutes.gateway.networking.k8s.io \
  --as=system:serviceaccount:tanzu-system-service-discovery:external-dns
kubectl auth can-i list httproutes.gateway.networking.k8s.io \
  --as=system:serviceaccount:tanzu-system-service-discovery:external-dns
kubectl auth can-i watch httproutes.gateway.networking.k8s.io \
  --as=system:serviceaccount:tanzu-system-service-discovery:external-dns
```

Expected result: `yes` for all three.

## 8. Install Tekton Pipelines (Air-Gapped from Harbor)

### 8.1 Create Namespace and Apply Security Label

Create the namespace for Tekton Pipelines:

```bash
kubectl create namespace tekton-pipelines
```

Add the privileged pod-security label to the namespace:

```bash
kubectl label namespace tekton-pipelines pod-security.kubernetes.io/enforce=privileged --overwrite
```

Verify the `tekton-pipelines` namespace:

```bash
kubectl get namespace tekton-pipelines --show-labels
kubectl get namespace tekton-pipelines -o jsonpath='{.metadata.labels.pod-security\.kubernetes\.io/enforce}{"\n"}'
```

Expected output for the last command:

```text
privileged
```

### 8.2 Create Harbor Pull Secret in `tekton-pipelines`

If your Harbor project (`lab25-harbor.lab25.sunfire.lab/tekton`) requires authentication to pull images, create a Docker registry secret in `tekton-pipelines`:

```bash
kubectl create secret docker-registry harbor-registry-secret \
  --docker-server=lab25-harbor.lab25.sunfire.lab \
  --docker-username='<HARBOR_USERNAME>' \
  --docker-password='<HARBOR_PASSWORD>' \
  -n tekton-pipelines
```

Verify the secret:

```bash
kubectl get secret harbor-registry-secret -n tekton-pipelines
```

### 8.3 Prepare `tekton-pipeline-release.yaml` for Harbor

In an air-gapped cluster, you cannot apply directly from `https://infra.tekton.dev/tekton-releases/pipeline/latest/release.yaml`. Use the local `tekton-pipeline-release.yaml` file on your jumpbox and ensure every image points to your Harbor registry (`lab25-harbor.lab25.sunfire.lab/tekton/...`).

| Manifest Section | What Needs to Point to Harbor |
|---|---|
| Deployment `tekton-pipelines-controller` | `image:` field **and** all `-*-image` arguments (`-kubeconfig-writer-image`, `-git-init-image`, `-entrypoint-image`, `-nop-image`, `-imagedigest-exporter-image`, `-pr-image`, `-workingdirinit-image`, `-sidecarlogresults-image`, `-shell-image`, `-shell-image-win`) |
| Deployment `tekton-pipelines-webhook` | `image:` field |
| Deployment `tekton-events-controller` | `image:` field |
| Deployment `tekton-pipelines-remote-resolvers` | `image:` field |

If your local `tekton-pipeline-release.yaml` still contains upstream image paths (`ghcr.io/tektoncd/pipeline/...` or `cgr.dev/chainguard/...`), rewrite them to your Harbor registry and strip `@sha256:...` digests if the images were re-tagged when pushed to Harbor:

```bash
export HARBOR_TEKTON_REPO="lab25-harbor.lab25.sunfire.lab/tekton"

# Backup original file
cp tekton-pipeline-release.yaml tekton-pipeline-release.yaml.bak

# Replace upstream registry paths with your Harbor path and remove @sha256 digests
sed -i -E \
  -e "s|ghcr.io/tektoncd/pipeline/([a-z0-9-]+)-[a-f0-9]+:([^@\"']+)(@sha256:[a-f0-9]+)?|${HARBOR_TEKTON_REPO}/\1:\2|g" \
  -e "s|cgr.dev/chainguard/busybox(@sha256:[a-f0-9]+)?|${HARBOR_TEKTON_REPO}/busybox:latest|g" \
  -e "s|mcr.microsoft.com/powershell:nanoserver(@sha256:[a-f0-9]+)?|${HARBOR_TEKTON_REPO}/powershell:nanoserver|g" \
  tekton-pipeline-release.yaml
```

Verify that no external registries remain in `tekton-pipeline-release.yaml`:

```bash
grep -E "image:|image=" tekton-pipeline-release.yaml
grep -E "ghcr\.io|gcr\.io|cgr\.dev|docker\.io" tekton-pipeline-release.yaml
```

Expected result: all image references show `lab25-harbor.lab25.sunfire.lab/tekton/...` and the second command returns no matches.

### 8.4 Install Tekton Pipelines from Local Manifest

File: `tekton-pipeline-release.yaml`

```bash
kubectl apply -f tekton-pipeline-release.yaml
```

If your Harbor project is private, attach `harbor-registry-secret` to the Tekton ServiceAccounts in `tekton-pipelines` and restart the deployments so the pods can pull from Harbor:

```bash
for sa in default tekton-pipelines-controller tekton-pipelines-webhook tekton-pipelines-resolvers tekton-events-controller; do
  kubectl patch serviceaccount "$sa" -n tekton-pipelines \
    -p '{"imagePullSecrets":[{"name":"harbor-registry-secret"}]}' || true
done

kubectl rollout restart deployment -n tekton-pipelines
```

### 8.5 Validation: Tekton Pipelines

```bash
kubectl get pods -n tekton-pipelines
kubectl get crd | grep tekton
kubectl api-resources | grep tekton
kubectl logs deployment/tekton-pipelines-controller -n tekton-pipelines --tail=50
```

Expected result:

```text
Namespace tekton-pipelines      → Active
tekton-pipelines-controller     → 1/1 Running
tekton-pipelines-webhook        → 1/1 Running
tekton-pipelines-remote-resolvers → 1/1 Running
tekton-events-controller        → 1/1 Running
CRDs                            → tasks, taskruns, pipelines, pipelineruns (tekton.dev) present
Controller logs                 → no ERROR / failed / panic / ImagePullBackOff
```

## 9. Install Tekton Dashboard (Air-Gapped from Harbor)

### 9.1 Prepare `tekton-dashboard-release.yaml` for Harbor

File: `tekton-dashboard-release.yaml`

Ensure the Dashboard container image inside `tekton-dashboard-release.yaml` points to your Harbor registry (`lab25-harbor.lab25.sunfire.lab/tekton/dashboard:<TAG>`):

```bash
export HARBOR_TEKTON_REPO="lab25-harbor.lab25.sunfire.lab/tekton"

cp tekton-dashboard-release.yaml tekton-dashboard-release.yaml.bak

sed -i -E \
  "s|ghcr.io/tektoncd/dashboard/dashboard-[a-f0-9]+:([^@\"']+)(@sha256:[a-f0-9]+)?|${HARBOR_TEKTON_REPO}/dashboard:\1|g" \
  tekton-dashboard-release.yaml
```

Verify the image line in `tekton-dashboard-release.yaml`:

```bash
grep "image:" tekton-dashboard-release.yaml
```

Expected result: `image: lab25-harbor.lab25.sunfire.lab/tekton/dashboard:<TAG>`.

### 9.2 Apply `tekton-dashboard-release.yaml`

```bash
kubectl apply -f tekton-dashboard-release.yaml
```

If your Harbor project is private, patch the `tekton-dashboard` ServiceAccount with `harbor-registry-secret` and restart the deployment:

```bash
kubectl patch serviceaccount tekton-dashboard -n tekton-pipelines \
  -p '{"imagePullSecrets":[{"name":"harbor-registry-secret"}]}'

kubectl rollout restart deployment tekton-dashboard -n tekton-pipelines
```

### 9.3 Validation: Tekton Dashboard

```bash
kubectl get pods,deployment,svc -n tekton-pipelines | grep dashboard
```

Expected result: `tekton-dashboard` Pod and Deployment `1/1 Running`; Service `tekton-dashboard` on port `9097`.

### 9.4 Create an Air-Gapped CA `ClusterIssuer` for cert-manager

In an air-gapped environment, `cert-manager` cannot contact Let's Encrypt (`https://acme-v02.api.letsencrypt.org/directory`). Instead, create a Kubernetes TLS Secret in the `cert-manager` namespace using your internal Root CA (`ca.crt` and `ca.key`) and configure a CA `ClusterIssuer`.

> **Note:** Keep the ClusterIssuer name `letsencrypt-prod` (or `ca-issuer`) so it matches `issuerRef.name` in `tekton-dashboard-certificate.yaml`.

#### 9.4.1 Create the Internal CA Secret in `cert-manager`

```bash
kubectl create secret tls ca-key-pair \
  --cert=ca.crt \
  --key=ca.key \
  -n cert-manager
```

Validate that the Secret exists:

```bash
kubectl get secret ca-key-pair -n cert-manager
```

#### 9.4.2 Create the Air-Gap `ClusterIssuer`

File `cluster-issuer.yaml`:

| Important argument | Value |
|---|---|
| ClusterIssuer name | `letsencrypt-prod` (or `ca-issuer`) |
| Issuer type | `ca` (offline internal Certificate Authority) |
| `secretName` | `ca-key-pair` in namespace `cert-manager` |

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  ca:
    secretName: ca-key-pair
```

Apply the ClusterIssuer:

```bash
kubectl apply -f cluster-issuer.yaml
```

Check its status:

```bash
kubectl get clusterissuer letsencrypt-prod
```

Expected result: `READY=True`.

## 10. Istio Integration: Expose the Dashboard

All three resources are created in `tekton-pipelines`, the namespace of the `tekton-dashboard` Service, so no cross-namespace reference is needed. Apply them in this order: `Certificate`, `Gateway`, `HTTPRoute`.

### 10.1 TLS Certificate (cert-manager)

File `tekton-dashboard-certificate.yaml`:

| Important argument | Value |
|---|---|
| `issuerRef` | ClusterIssuer `letsencrypt-prod` (backed by internal CA from Section 9.4) |
| `dnsNames` | `tekton.k8slabs.com` |
| `secretName` | `tekton-dashboard-tls-cert` (created by cert-manager, used by the Gateway) |

```bash
kubectl apply -f tekton-dashboard-certificate.yaml
kubectl get certificate -n tekton-pipelines     # wait for READY=True
```

### 10.2 tekton-dashboard-gateway

File `tekton-dashboard-gateway.yaml`:

| Important argument | Value |
|---|---|
| `gatewayClassName` | `istio` |
| Listener | HTTPS, port 443, `tls.mode: Terminate` |
| `hostname` | `tekton.k8slabs.com` |
| `certificateRefs` | Secret `tekton-dashboard-tls-cert` (from Section 10.1) |
| `allowedRoutes` | Same namespace only |

```bash
kubectl get namespace tekton-pipelines --show-labels
kubectl apply -f tekton-dashboard-gateway.yaml
kubectl get gateway tekton-dashboard-gateway -n tekton-pipelines
```

Expected result: `PROGRAMMED=True` with an `ADDRESS`. Retrieve the Gateway IP address:

```bash
kubectl get gateway tekton-dashboard-gateway -n tekton-pipelines \
  -o jsonpath='{.status.addresses[0].value}{"\n"}'
```

### 10.3 HTTPRoute

File `tekton-dashboard-httproute.yaml`:

| Important argument | Value |
|---|---|
| `parentRefs` | Gateway `tekton-dashboard-gateway` |
| `hostnames` | `tekton.k8slabs.com` |
| `backendRefs` | Service `tekton-dashboard`, port `9097` |

```bash
kubectl apply -f tekton-dashboard-httproute.yaml
```

### 10.4 Validation: Istio Routing

```bash
kubectl get certificate,gateway,httproute -n tekton-pipelines
kubectl describe httproute tekton-dashboard-route -n tekton-pipelines
```

Expected result: Certificate `READY=True`, Gateway `PROGRAMMED=True`, and HTTPRoute conditions `Accepted=True` and `ResolvedRefs=True`.

## 11. Internal DNS Reconciliation

### 11.1 External DNS Rolling Restart or Static DNS Entry

If using External DNS with an internal DNS server:

```bash
kubectl rollout restart deployment external-dns -n tanzu-system-service-discovery
kubectl rollout status deployment external-dns -n tanzu-system-service-discovery

kubectl get pods -n tanzu-system-service-discovery
kubectl logs -n tanzu-system-service-discovery -l app=external-dns --tail=50
```

If managing DNS manually in your air-gapped lab DNS server (or testing from the jumpbox `/etc/hosts`), map the Gateway IP from Section 10.2 to `tekton.k8slabs.com`:

```bash
GATEWAY_IP=$(kubectl get gateway tekton-dashboard-gateway -n tekton-pipelines -o jsonpath='{.status.addresses[0].value}')
echo "${GATEWAY_IP} tekton.k8slabs.com" | sudo tee -a /etc/hosts
```

### 11.2 Verify the DNS Resolution

```bash
nslookup tekton.k8slabs.com
```

Expected result: the IP returned equals the Gateway address from Section 10.2.

## 12. Dashboard Access

Open the Dashboard in your browser (or verify via `curl` using your internal CA certificate):

```text
https://tekton.k8slabs.com
```

Verify from the jumpbox CLI:

```bash
curl -I --cacert ca.crt https://tekton.k8slabs.com
```

Expected result: `HTTP/2 200` and the browser displays the Tekton Dashboard with a valid certificate.

**Production:** The Dashboard can act on the cluster. Add authentication, RBAC, and network restrictions before wider use.

## 13. Create the `cicd` Namespace with Security Label and Harbor Credentials

All application-specific Tekton resources (`Task`, `TaskRun`, `Pipeline`, `PipelineRun`) run in the `cicd` namespace. Because this is an air-gapped environment, also configure `harbor-registry-secret` in `cicd` so Tasks can pull images from Harbor.

```bash
# 1. Check if namespace exists
kubectl get namespace cicd

# 2. Create namespace
kubectl create namespace cicd

# 3. Apply the privileged pod-security label immediately
kubectl label namespace cicd \
  pod-security.kubernetes.io/enforce=privileged \
  --overwrite

# 4. Create the Harbor pull secret in cicd
kubectl create secret docker-registry harbor-registry-secret \
  --docker-server=lab25-harbor.lab25.sunfire.lab \
  --docker-username='<HARBOR_USERNAME>' \
  --docker-password='<HARBOR_PASSWORD>' \
  -n cicd

# 5. Attach Harbor pull secret to the default ServiceAccount in cicd
kubectl patch serviceaccount default -n cicd \
  -p '{"imagePullSecrets":[{"name":"harbor-registry-secret"}]}'
```

### 13.1 Validation

```bash
kubectl get namespace cicd --show-labels
kubectl get namespace cicd -o jsonpath='{.metadata.labels.pod-security\.kubernetes\.io/enforce}{"\n"}'
kubectl get secret harbor-registry-secret -n cicd
```

Expected result: `Active`, `privileged`, and `harbor-registry-secret` present in `cicd`.

## 14. Smoke Test: Task and TaskRun (Air-Gapped)

This minimal test proves that Tekton Pipelines can schedule pods, pull step images from your local Harbor registry (`lab25-harbor.lab25.sunfire.lab`), and report logs to the Tekton Dashboard without internet access.

| File | Purpose | Air-Gap Setting |
|---|---|---|
| `test-task.yaml` | Task `hello` that prints a test message | Step `image:` set to `lab25-harbor.lab25.sunfire.lab/tekton/busybox:latest` (or mirrored `ubuntu` image in Harbor) |
| `taskrun.yaml` | TaskRun `hello-taskrun` that executes the Task | Runs in namespace `cicd` |

Verify `test-task.yaml` uses your Harbor image:

```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: hello
  namespace: cicd
spec:
  steps:
    - name: echo
      image: lab25-harbor.lab25.sunfire.lab/tekton/busybox:latest
      command:
        - echo
      args:
        - "Hello World from Air-Gapped Tekton on VKS!"
```

Verify `taskrun.yaml`:

```yaml
apiVersion: tekton.dev/v1
kind: TaskRun
metadata:
  name: hello-taskrun
  namespace: cicd
spec:
  taskRef:
    name: hello
```

Apply both manifests:

```bash
kubectl apply -f test-task.yaml
kubectl apply -f taskrun.yaml
```

### 14.1 Validation

```bash
kubectl get task,taskrun -n cicd
kubectl logs -l tekton.dev/taskRun=hello-taskrun -n cicd
```

Expected result: `hello-taskrun` shows `SUCCEEDED=True`, and the logs print `Hello World from Air-Gapped Tekton on VKS!`.

In the Tekton Dashboard (`https://tekton.k8slabs.com`): select Namespace → `cicd` and confirm both the `hello` Task and `hello-taskrun` TaskRun are visible and green.

## 15. Installation Validation and Final Verification

Run once at the end of the Tekton Pipelines and Tekton Dashboard installation. Every line must pass before moving on to Buildpacks, BuildKit, Cosign, and Triggers.

| Area | Command | Expected |
|---|---|---|
| Cluster | `kubectl get nodes -o wide` | All nodes `Ready` |
| Harbor Connectivity | `curl -I --cacert harbor-ca.crt https://lab25-harbor.lab25.sunfire.lab/v2/` | `HTTP/2 200` or `401` |
| Tekton Pods | `kubectl get pods -n tekton-pipelines` | Controller, webhook, resolvers, events, dashboard all `1/1 Running` |
| Tekton Images | `kubectl get pods -n tekton-pipelines -o jsonpath='{..image}'` | All images pulled from `lab25-harbor.lab25.sunfire.lab` |
| Tekton CRDs | `kubectl get crd \| grep tekton` | `tekton.dev` CRDs present |
| Namespace | `kubectl get namespace cicd --show-labels` | `Active`, `enforce=privileged` |
| Dashboard Service | `kubectl get pods,svc -n tekton-pipelines \| grep dashboard` | `1/1 Running`, port `9097` |
| Istio & TLS | `kubectl get certificate,gateway,httproute -n tekton-pipelines` | `Ready=True`, `Programmed=True`, `Accepted=True` |
| DNS | `nslookup tekton.k8slabs.com` | Resolves to Istio Gateway IP |
| Smoke Test | `kubectl get taskrun hello-taskrun -n cicd` | `SUCCEEDED=True` |
| User Access | Open `https://tekton.k8slabs.com` | Tekton Dashboard loads with valid TLS certificate |
