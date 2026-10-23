# Tekton Pipelines and Tekton Dashboard Installation on VKS (Air-Gapped Environment)
# Airgap Container Images

## Air-Gap Images colums 

| Column 1 | Column 2 | Column 3 | Column 4 | Column 5 | Column 6 |
| --- | --- | --- | --- | --- | --- |
| Container Images Required for Air-Gapped Installation |  |  |  |  |  |
|  |  |  |  |  |  |
| Images Required for Tekton Pipeline |  |  |  |  |  |
| Source file: tekton-pipeline-release.yaml |  |  |  |  |  |
| Sr. No. | Component / Task | Source Image | Version | Purpose | Air-Gap Action |
| 1 | Tekton Pipeline Controller | ghcr.io/tektoncd/pipeline/controller-10a3e32792f33651396d02b6855a6e36:v1.17.0 | v1.17.0 | Runs Tekton Pipelines | Mirror to Harbor |
| 2 | Tekton Pipeline Entrypoint | ghcr.io/tektoncd/pipeline/entrypoint-bff0a22da108bc2f16c818c97641a296:v1.17.0 | v1.17.0 | Executes Task steps | Mirror to Harbor |
| 3 | Tekton Pipeline Nop | ghcr.io/tektoncd/pipeline/nop-8eac7c133edad5df719dc37b36b62482:v1.17.0 | v1.17.0 | Handles no-op steps | Mirror to Harbor |
| 4 | Tekton Sidecar Log Results | ghcr.io/tektoncd/pipeline/sidecarlogresults-7501c6a20d741631510a448b48ab098f:v1.17.0 | v1.17.0 | Collects step results | Mirror to Harbor |
| 5 | Tekton Working Directory Init | ghcr.io/tektoncd/pipeline/workingdirinit-0c558922ec6a1b739e550e349f2d5fc1:v1.17.0 | v1.17.0 | Prepares Task working directories | Mirror to Harbor |
| 6 | Tekton Pipeline Webhook | ghcr.io/tektoncd/pipeline/webhook-d4749e605405422fd87700164e31b2d1:v1.17.0 | v1.17.0 | Validates and processes Pipeline resources | Mirror to Harbor |
| 7 | Tekton Shell | cgr.dev/chainguard/busybox | Digest-pinned in manifest | Runs shell commands in steps | Mirror to Harbor |
|  |  |  |  |  |  |
|  |  |  |  |  |  |
| Images Required for Tekton Dashboard |  |  |  |  |  |
| Source file: tekton-dashboard-release.yaml |  |  |  |  |  |
| Sr. No. | Component / Task | Source Image | Version | Purpose | Air-Gap Action |
| 1 | Tekton Dashboard | ghcr.io/tektoncd/dashboard/dashboard-9623576a202fe86c8b7d1bc489905f86:v0.73.0 | v0.73.0 | Provides the web UI for Tekton | Mirror to Harbor |
|  |  |  |  |  |  |
|  |  |  |  |  |  |
| Images Required for Tekton Triggers |  |  |  |  |  |
| Source files: tekton-triggers-release.yaml and the Core Interceptors manifest, if maintained separately. |  |  |  |  |  |
| Sr. No. | Component / Task | Source Image | Version | Purpose | Air-Gap Action |
| 1 | Tekton Triggers Controller | ghcr.io/tektoncd/triggers/controller-f656ca31de179ab913fa76abc255c315:v0.35.3 | v0.35.3 | Reconciles Trigger resources and creates PipelineRuns | Mirror to Harbor |
| 2 | Tekton EventListener Sink | ghcr.io/tektoncd/triggers/eventlistenersink-7ad1faa98cddbcb0c24990303b220bb8:v0.35.3 | v0.35.3 | Receives and forwards webhook events | Mirror to Harbor |
| 3 | Tekton Triggers Webhook | ghcr.io/tektoncd/triggers/webhook-dd1edc925ee1772a9f76e2c1bc291ef6:v0.35.3 | v0.35.3 | Validates and mutates Trigger resources | Mirror to Harbor |
| 4 | Tekton Core Interceptors | ghcr.io/tektoncd/triggers/interceptors-3176d6a3f314c3655b30bfd36e421dd5:v0.34.0 | v0.34.0 | Provides CEL, GitHub, GitLab, Bitbucket and Slack interceptors | Mirror to Harbor |
|  |  |  |  |  |  |
|  |  |  |  |  |  |
| Images Required for Pipeline Tasks |  |  |  |  |  |
| Source files: Your individual Task YAML files, such as smoke-test, build-type detection, Git clone, BuildKit, Buildpacks, SBOM, signing and Git update Tasks. |  |  |  |  |  |
| Sr. No. | Component / Task | Source Image | Version | Purpose | Air-Gap Action |
| 1 | Smoke Test Task | alpine:3.20 | 3.20 | Runs the initial Tekton smoke test | Mirror to Harbor |
| 2 | Detect Build Type Task | alpine:3.20 | 3.20 | Detects the application build type | Reuse the image from the Smoke Test Task |
| 3 | Git Clone Task | alpine/git:latest | latest | Clones source code | Mirror to Harbor and pin a version |
| 4 | BuildKit Task | moby/buildkit:latest | latest | Builds container images from Dockerfiles | Mirror to Harbor and pin a version |
| 5 | Buildpacks Task — Skopeo/JQ | quay.io/halkyonio/skopeo-jq:0.1.3 | 0.1.3 | Supports image operations and JSON processing | Mirror the digest-pinned image |
| 6 | Buildpacks Task — UBI Minimal | registry.access.redhat.com/ubi8/ubi-minimal | Digest-pinned in manifest | Provides a minimal UBI environment | Mirror the digest-pinned image |
| 7 | Buildpacks Task — Python | registry.access.redhat.com/ubi8/python-311 | Digest-pinned in manifest | Provides a Python runtime/build environment | Mirror the digest-pinned image |
| 8 | Buildpacks Builder | paketobuildpacks/builder-jammy-base | Pin the selected digest | Builds applications using Cloud Native Buildpacks | Mirror the builder and all required build/run images |
| 9 | SBOM Generation | anchore/syft:latest | latest | Generates a software bill of materials | Mirror to Harbor and pin a version |
| 10 | Image Signing / Attestation | ghcr.io/sigstore/cosign/cosign:v2.4.1 | v2.4.1 | Signs and attests built images | Mirror to Harbor |
| 11 | Sign Task Helper | alpine:3.20 | 3.20 | Reads image digests or runs helper commands | Reuse the image from the Smoke Test Task |
| 12 | Update Values Task | alpine/git:latest | latest | Updates the Git repository after a build | Reuse the Git Clone image; pin a version |
|  |  |  |  |  |  |
|  |  |  |  |  |  |
| Application Image Produced by the Pipeline |  |  |  |  |  |
| Sr. No. | Component / Task | Source Image | Version | Purpose | Air-Gap Action |
| 1 | Application Output | lab25-harbor.lab25.sunfire.lab/cicd/travelportal:latest | latest (build output tag) | Stores the built Travel Portal application image in Harbor | Build and push to the internal Harbor registry |
| This is an output image created by your pipeline, not an upstream image that needs to be mirrored before installation. |  |  |  |  |  |


## 1. Tekton Image Relocation Table (`imgpkg copy` Commands)

> **Important Note on Your Original Commands:** 
> 1. Your original commands had truncated SHA256 hashes ending in `…` (which will cause terminal errors) and included `:v1.17.0` at the end of `--to-repo` (which is invalid in `imgpkg`). 
> 2. In the table below, the **Cleaned & Ready-to-Use Command** column removes the invalid tag on `--to-repo` and uses the version tag so it can be run cleanly without failing on a cut-off SHA hash.

| # | Component Name | Version | What This Command & Image Does | Cleaned & Ready-to-Use Command (Jumpbox / Bastion) |
| :--- | :--- | :--- | :--- | :--- |
| **1** | **Tekton Entrypoint** | `v1.17.0` | Copies the `entrypoint` binary image. Tekton injects this into **every step** of a TaskPod to control step ordering (making Step 2 wait until Step 1 completes). | `imgpkg copy --cosign-signatures --include-non-distributable-layers -i ghcr.io/tektoncd/pipeline/entrypoint-bff0a22da108bc2f16c818c97641a296:v1.17.0 --to-repo harbor.sunfire.lab/tektoncd/pipeline/entrypoint-bff0a22da108bc2f16c818c97641a296 --registry-ca-cert-path /etc/docker/certs.d/harbor.sunfire.lab/ca.crt` |
| **2** | **Tekton NOP (No-Op)** | `v1.17.0` | Copies the `nop` image. Used by Tekton to gracefully stop sidecar containers once all main Task steps have finished running. | `imgpkg copy --cosign-signatures --include-non-distributable-layers -i ghcr.io/tektoncd/pipeline/nop-8eac7c133edad5df719dc37b36b62482:v1.17.0 --to-repo harbor.sunfire.lab/tektoncd/pipeline/nop-8eac7c133edad5df719dc37b36b62482 --registry-ca-cert-path /etc/docker/certs.d/harbor.sunfire.lab/ca.crt` |
| **3** | **Sidecar Log Results** | `v1.17.0` | Copies the `sidecarlogresults` helper image. Runs alongside Task pods to capture step outputs (`results`) and stream them back to the Tekton controller. | `imgpkg copy --cosign-signatures --include-non-distributable-layers -i ghcr.io/tektoncd/pipeline/sidecarlogresults-7501c6a20d741631510a448b48ab098f:v1.17.0 --to-repo harbor.sunfire.lab/tektoncd/pipeline/sidecarlogresults-7501c6a20d741631510a448b48ab098f --registry-ca-cert-path /etc/docker/certs.d/harbor.sunfire.lab/ca.crt` |
| **4** | **Working Dir Init** | `v1.17.0` | Copies the `workingdirinit` init-container image. Creates and sets up permissions for shared `/workspace` directories before Task steps start. | `imgpkg copy --cosign-signatures --include-non-distributable-layers -i ghcr.io/tektoncd/pipeline/workingdirinit-0c558922ec6a1b739e550e349f2d5fc1:v1.17.0 --to-repo harbor.sunfire.lab/tektoncd/pipeline/workingdirinit-0c558922ec6a1b739e550e349f2d5fc1 --registry-ca-cert-path /etc/docker/certs.d/harbor.sunfire.lab/ca.crt` |
| **5** | **Tekton Events** | `v1.17.0` | Copies the `events` controller image. Responsible for emitting Kubernetes events and CloudEvents when PipelineRuns or TaskRuns change status. | `imgpkg copy --cosign-signatures --include-non-distributable-layers -i ghcr.io/tektoncd/pipeline/events-a9042f7efb0cbade2a868a1ee5ddd52c:v1.17.0 --to-repo harbor.sunfire.lab/tektoncd/pipeline/events-a9042f7efb0cbade2a868a1ee5ddd52c --registry-ca-cert-path /etc/docker/certs.d/harbor.sunfire.lab/ca.crt` |
| **6** | **Tekton Resolvers** | `v1.17.0` | Copies the `resolvers` controller image. Allows Tekton to pull Task and Pipeline definitions remotely from Git, OCI bundles, or Cluster scopes. | `imgpkg copy --cosign-signatures --include-non-distributable-layers -i ghcr.io/tektoncd/pipeline/resolvers-ff86b24f130c42b88983d3c13993056d:v1.17.0 --to-repo harbor.sunfire.lab/tektoncd/pipeline/resolvers-ff86b24f130c42b88983d3c13993056d --registry-ca-cert-path /etc/docker/certs.d/harbor.sunfire.lab/ca.crt` |
| **7** | **Tekton Webhook** | `v1.17.0` | Copies the `webhook` admission controller image. Validates Tekton YAML syntax and sets default fields when you run `kubectl apply`. | `imgpkg copy --cosign-signatures --include-non-distributable-layers -i ghcr.io/tektoncd/pipeline/webhook-d4749e605405422fd87700164e31b2d1:v1.17.0 --to-repo harbor.sunfire.lab/tektoncd/pipeline/webhook-d4749e605405422fd87700164e31b2d1 --registry-ca-cert-path /etc/docker/certs.d/harbor.sunfire.lab/ca.crt` |
| **8** | **Tekton Dashboard** | `v0.73.0` | Copies the `dashboard` web UI image. Provides the graphical web dashboard to monitor Pipelines, Tasks, and logs inside your browser. | `imgpkg copy --cosign-signatures --include-non-distributable-layers -i ghcr.io/tektoncd/dashboard/dashboard-9623576a202fe86c8b7d1bc489905f86:v0.73.0 --to-repo harbor.sunfire.lab/tektoncd/dashboard/dashboard-9623576a202fe86c8b7d1bc489905f86 --registry-ca-cert-path /etc/docker/certs.d/harbor.sunfire.lab/ca.crt` |

---

## 2. `imgpkg` Flag-by-Flag Reference Table

This table explains what each argument inside your `imgpkg copy` command does and why it is required for an air-gapped Harbor transfer.

| Flag / Argument | Example Value | What It Does & Why You Need It |
| :--- | :--- | :--- |
| **`imgpkg copy`** | `imgpkg copy` | Carvel tool command that copies container images and OCI bundles between registries or tar files while preserving exact digests (`@sha256`). |
| **`--cosign-signatures`** | *(Boolean Flag)* | Searches for and copies **Cosign cryptographic signatures** alongside the container image so offline security tools (like Kyverno) can verify the image wasn't tampered with. |
| **`--include-non-distributable-layers`** | *(Boolean Flag)* | Forces the copy of restricted/foreign image layers that tools normally skip. Mandatory for air-gapped registries so pulls don't fail due to missing base layers. |
| **`-i`** | `ghcr.io/tektoncd/...:v1.17.0` | **Source Image:** Defines the public upstream image on GitHub Container Registry (`ghcr.io`) to pull from. |
| **`--to-repo`** | `harbor.sunfire.lab/tektoncd/...` | **Destination Repository:** Defines the target repository path inside your internal Harbor registry where the image will be pushed. *(Do not put `:tag` here).* |
| **`--registry-ca-cert-path`** | `/etc/docker/certs.d/harbor.sunfire.lab/ca.crt` | **Custom Root CA Path:** Tells `imgpkg` to trust your private Harbor registry's self-signed or internal SSL certificate during the HTTPS push. |

---

## 3. Kubernetes Proxy & Custom CA YAML Reference Table (`external-dns`)

This table explains each section of the Kubernetes Pod specification used to allow `external-dns` to communicate through a corporate proxy and trust internal TLS certificates.

| YAML Field / Section | Configured Value | What It Is Used For |
| :--- | :--- | :--- |
| **`containers.name`** | `external-dns` | Defines the container running ExternalDNS (which syncs Kubernetes Ingresses/Services to your DNS server). |
| **`env: HTTP_PROXY`** | `http://proxy.example.com:8080` | Routes all outbound unencrypted HTTP requests from the pod through your corporate forward proxy. |
| **`env: HTTPS_PROXY`** | `http://proxy.example.com:8080` | Routes all outbound encrypted HTTPS requests (e.g., to external DNS APIs) through your corporate forward proxy. |
| **`env: NO_PROXY`** | `10.0.0.0/8,kubernetes.default.svc` | **Bypass List:** Prevents the pod from sending internal cluster traffic (`10.0.0.0/8`) or Kubernetes API calls (`kubernetes.default.svc`) to the external corporate proxy. |
| **`volumeMounts.name`** | `ca-bundle` | Attaches the certificate volume defined in `volumes` into the running container. |
| **`volumeMounts.mountPath`** | `/etc/ssl/certs` | Mounts the certificates directly into the Linux system trust store path so the container trusts your proxy's SSL inspection certificate. |
| **`volumeMounts.readOnly`** | `true` | Protects the mounted certificate files from being modified or deleted by the container. |
| **`volumes.secret.secretName`** | `ca-proxy` | References the Kubernetes Secret (`ca-proxy`) in the same namespace that stores your corporate CA certificate bundle. |



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
