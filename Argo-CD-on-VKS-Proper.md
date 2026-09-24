# Argo CD on VKS

**VMware Argo CD Supervisor Service — Internet-Connected & Air-Gapped**

- **Argo CD Supervisor Service:** 1.2.0

---

## 1. Introduction

This document describes the implementation of the VMware Argo CD Supervisor Service for GitOps-based application delivery to VKS.

The VMware Argo CD Operator runs on the Supervisor and manages Argo CD instances deployed in vSphere Namespaces. The Argo CD instance is used to manage resources in registered VKS workload clusters.

In this version of the document, all Kubernetes YAML resources (the ArgoCD custom resource, Argo CD Applications and the application manifests) are deployed using **Helm charts** instead of `kubectl apply -f`. Steps that can only be done through the vSphere Client (Supervisor Service registration/installation, namespace creation, container registry configuration) are unchanged.

## 2. Why Argo CD on VKS?

VKS provides the Kubernetes runtime for application workloads. The Argo CD Supervisor Service provides the GitOps continuous delivery layer and integrates Argo CD with the VCF/VKS platform.

| Point | Description |
|---|---|
| Git as the source of truth | Desired application configuration is maintained in the GitOps repository. |
| VMware platform integration | The Argo CD lifecycle is managed by the VMware Argo CD Operator. |
| GitOps reconciliation | Argo CD compares desired state in Git with the live VKS state. |
| VKS cluster management | A VKS cluster can be registered as an Argo CD destination. |
| Controlled delivery | Application synchronization can follow the project's staging and production approval process. |

## 3. Benefits

- Declarative GitOps-based application deployment.
- VMware integration with Supervisor and VKS.
- Centralized management of registered VKS clusters.
- Internal Harbor and Git support for disconnected environments.
- Clear deployment status and drift visibility.
- Helm-packaged, versioned and parameterized deployments (values files per environment, `helm rollback` support).

## 4. Architecture

**VKS Platform with VMware Argo CD Supervisor Service**

```
Developer
  |
  v
Application Git Repository
  |
  | Webhook / Trigger
  v
Tekton CI Pipeline
  |
  +--> Buildpacks / BuildKit
  |
  +--> Push Image
  v
Harbor
  |
  | Update image tag / digest (Helm values file)
  v
GitOps Repository (Helm charts)
  |
  v
Supervisor
  |
  v
VMware Argo CD Operator
  |
  v
Argo CD Instance
(vSphere Namespace)
  |
  | Register / Manage
  v
VKS Workload Cluster
  |
  v
Application
```

The Application Git → Webhook/Trigger → Tekton flow belongs to CI. Argo CD consumes the GitOps repository and reconciles the application state to the registered VKS cluster.

## 5. Pre-Requisites

| Component | Version | Notes |
|---|---|---|
| VMware Cloud Foundation (VCF) | 9.0.0 | Project platform. |
| vSphere Kubernetes Service (VKS) | 3.6.x | Target workload cluster platform. |
| Kubernetes | Kubernetes v1.35.5+vmware.1 | Runtime for the VKS workload cluster. |
| Argo CD Supervisor Service | 1.1.0-25100889 | Approved Supervisor Service version for this implementation. |
| Argo CD package | 3.0.19+vmware.1-vks.1 | Supported range declared by Supervisor Service 1.2.0; verify exact package before deployment. |
| Supervisor | 9.0.0.0100 or later | Required Supervisor level documented for the Argo CD Supervisor Service. |
| Harbor (Supervisor Service) | 2.14.3 | Internal registry used by the project for application images, Helm OCI charts and disconnected service-image relocation. |
| VCF CLI | 9.0.x | VCF CLI for connecting to a VCF 9.0.x Supervisor. |
| Helm CLI | 3.14+ | Used to deploy the ArgoCD instance chart and the Argo CD Application chart. |
| Argo CD CLI | 3.0.19-vcf | Company-provided VMware-customized Linux AMD64 CLI. |
| Argo CD Supervisor Service YAML | 1.1.0-25100889 | Connected manifest: supervisor-service-argocd-legacy-1.1.0-25100889.yml. |
| imgpkg (Carvel) | 0.42+ | Used for Supervisor Service bundle relocation in the air-gapped environment. |
| Internal GitOps Repository | Approved internal Git | Must be reachable by the Argo CD instance. |
| Internal DNS | argocd.apps.company.local | Argo CD web UI hostname. |
| Internal LoadBalancer | Platform-provided | Argo CD server uses the LoadBalancer service provided by the Supervisor Service. |
| Required privileges | Project-approved | Supervisor Services management/install and vSphere Namespace permissions. |

The VMware documentation states that the Argo CD Supervisor Service is installed on the Supervisor and the Argo CD instance is deployed in a vSphere Namespace. It also exposes the Argo CD server through a LoadBalancer and provides local account, RBAC and other settings through the ArgoCD custom resource.

## 6. Installing Argo CD Supervisor Service — Internet-Connected Environment

### 6.1 Download the Argo CD Supervisor Service YAML

1. Sign in to the Broadcom Support Portal: https://support.broadcom.com/
2. Open **Software > Enterprise Software > My Downloads**.
3. Search for **vSphere Supervisor Services**.
4. Open ArgoCD Service and select version 1.1.0-25100889
5. Agree to the Terms and Conditions and download the Argo CD Supervisor Service YAML.
6. Keep the downloaded service YAML as a versioned project artifact.

For this implementation, the company-provided 1.1.0-25100889 manifest is the artifact used for registration.

### 6.2 Register the Supervisor Service with vCenter

1. Open the vSphere Client.
2. Go to **Supervisor Management > Services**.
3. Select the target vCenter.
4. Upload the downloaded Argo CD Supervisor Service YAML in **Add New Service**.
5. Complete the registration and accept the EULA when prompted.

The service must be registered with vCenter before it is installed on a Supervisor.

### 6.3 Install the Supervisor Service on the Supervisor

1. Go to **Supervisor Management > Services**.
2. Select the registered ArgoCD Service 1.1.0-25100889.
3. Select **Actions > Install on Supervisors**.
4. Select the target Supervisor.
5. Review the compatibility pre-check results.
6. Complete the installation.

There is no kubectl or Helm command for installing the Supervisor Service itself. The VCF procedure installs the Supervisor Service through the vSphere Client and performs compatibility checks before installation.

### 6.4 Create the vSphere Namespace

1. In the vSphere Client, open **Supervisor Management > Namespaces**.
2. Select **Create Namespace**.
3. Select the target Supervisor.
4. Enter the approved namespace name: `argocd-instance-1`.
5. Configure the required namespace policies, storage : attach the storage policy:
lab-gold-storage-policy , network and other project settings.
6. Create the namespace and verify that it is available on the Supervisor.

The Argo CD instance is deployed in a vSphere Namespace.

### 6.5 Use the Existing Supervisor Context

```bash
# Check the existing VCF CLI contexts
vcf context list

# Use the existing Supervisor context
vcf context use sup-01

# Optional kubectl check
kubectl config get-contexts
kubectl config current-context

# Verify Helm is installed
helm version
```

In the shared lab, use the existing Supervisor context when one is already configured. Do not create a new context unnecessarily, because other lab users may share the same administration host. If no suitable Supervisor context exists, create one with the VCF CLI.

Observed lab context:
```
vcf context use sup-01
[ok] Token is still active. Skipped the token refresh for context "sup-01"
[ok] Successfully activated context 'sup-01' (Type: kubernetes)
```

### 6.6 Verify the Argo CD Package Version

```bash
kubectl explain argocd.spec.version
kubectl get packages -A | grep -i argocd
```

The CRD confirms the version field format. The package list below was used to verify the exact Argo CD package available in the lab.

Observed lab output:
```
svc-argocd-service-domain-c1014
argocd.kubernetes.vmware.com.3.0.19+vmware.1-vks.1
argocd.kubernetes.vmware.com 3.0.19+vmware.1-vks.1
vmware-system-supervisor-services
argocd-service.vsphere.vmware.com.1.1.0-25100889
argocd-service.vsphere.vmware.com 1.1.0-25100889
```

### 6.7 Deploy the Argo CD Instance

Instead of a raw `argocd-instance.yaml` applied with `kubectl`, the ArgoCD custom resource is packaged as a Helm chart named `argocd-instance`. Use the verified package version from Section 6.6 as the `version` value.

Chart structure:

```text
charts/argocd-instance/
├── Chart.yaml
├── values.yaml
└── templates/
    └── argocd.yaml
```

**charts/argocd-instance/Chart.yaml**

```yaml
apiVersion: v2
name: argocd-instance
description: VMware Argo CD Supervisor Service instance
type: application
version: 1.0.0
appVersion: "3.0.19+vmware.1-vks.1"
```

**charts/argocd-instance/values.yaml**

```yaml
name: argocd-1
namespace: argocd-instance-1
version: 3.0.19+vmware.1-vks.1
enableLoadBalancer: true
serverSideDiff: true
localAccounts: []
rbacPolicy: ""
```

**charts/argocd-instance/templates/argocd.yaml**

```yaml
apiVersion: argocd-service.vsphere.vmware.com/v1alpha1
kind: ArgoCD
metadata:
  name: {{ .Values.name }}
  namespace: {{ .Values.namespace }}
spec:
  version: {{ .Values.version }}
  enableLoadBalancer: {{ .Values.enableLoadBalancer }}
  serverSideDiff: {{ .Values.serverSideDiff }}
  {{- with .Values.localAccounts }}
  localAccounts:
    {{- toYaml . | nindent 4 }}
  {{- end }}
  {{- if .Values.rbacPolicy }}
  rbac:
    policy: |
{{ .Values.rbacPolicy | indent 6 }}
  {{- end }}
```

Validate and deploy:

```bash
helm lint charts/argocd-instance
helm template argocd-instance charts/argocd-instance -n argocd-instance-1

helm upgrade --install argocd-instance charts/argocd-instance \
  -n argocd-instance-1
```

The Supervisor Service Operator creates the underlying Argo CD resources. The `enableLoadBalancer` setting enables the Argo CD server LoadBalancer, and `serverSideDiff: true` is the documented setting for VKS lifecycle management.

### 6.8 Verify the Argo CD Instance

```bash
helm list -n argocd-instance-1
kubectl get pods -n argocd-instance-1
kubectl get svc -n argocd-instance-1
kubectl get deploy -n argocd-instance-1
```

Observed lab output confirms the Argo CD server, repository server, application controller and Redis components are running. The Redis secret-init pod is expected to show Completed after initialization.

Observed service output:
```
argocd-redis        ClusterIP      10.96.1.192   <none>        6379/TCP
argocd-repo-server   ClusterIP      10.96.0.255   <none>        8081/TCP
argocd-server        LoadBalancer   10.96.0.102   10.12.90.54   80:30100/TCP,443:32074/TCP
```

Observed pod output:
```
argocd-application-controller-0      1/1     Running     0
argocd-redis-76566bbccc-cc4rh        1/1     Running     0
argocd-redis-secret-init-zj79n       0/1     Completed   0
argocd-repo-server-67bb8d5cb-5pfzz   1/1     Running     0
argocd-server-7ff9fd6994-65jkw       1/1     Running     0
```

### 6.9 Access the Argo CD Web UI

```bash
kubectl get svc -n argocd-instance-1 argocd-server

# Open the Argo CD UI using the observed LoadBalancer IP:
https://10.12.90.54
```

The deployed lab instance exposes the Argo CD server through LoadBalancer IP 10.12.90.54.

> Open the UI at https://10.12.90.54 (the lab browser showed the Argo CD Applications page).

The Argo CD web UI is accessed through the LoadBalancer endpoint exposed by the argocd-server service.

### 6.10 Retrieve the Initial Admin Password

```bash
kubectl get secret -n argocd-instance-1 argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```

### 6.11 Install the VMware-Customized Argo CD CLI

Use the company-provided VMware-customized Linux Argo CD CLI: `argocd-cli-linux-amd64-v3.0.19-vcf.gz`.

1. Save `argocd-cli-linux-amd64-v3.0.19-vcf.gz` on the administration host.
2. Decompress the archive and make the binary executable.

```bash
gunzip argocd-cli-linux-amd64-v3.0.19-vcf.gz
chmod +x argocd-cli-linux-amd64-v3.0.19-vcf
sudo install -m 555 argocd-cli-linux-amd64-v3.0.19-vcf /usr/local/bin/argocd
```

3. Verify the CLI.

```bash
argocd version
```
Observed lab output:
```
argocd: v3.0.19+d67e6eb90-vcf
BuildDate: 2025-12-02T08:11:08Z
GitTag: v3.0.19+d67e6eb90-vcf
Platform: linux/amd64
```
Use the company-provided VMware-customized Linux CLI `argocd-cli-linux-amd64-v3.0.19-vcf.gz`.

### 6.12 Login and Change the Admin Password

```bash
argocd login 10.12.90.54
```
Observed login output:
```
'admin:login' logged in successfully
Context '10.12.90.54' updated
```
Verify the active Argo CD CLI context:
```
argocd context
```
Observed lab output:
```
CURRENT  NAME         SERVER
*        10.12.90.54  10.12.90.54
```
Change the initial admin password:
```
argocd account update-password
```
Enter the current password, new password and confirmation when prompted.
Observed output:
```
Password updated
Context '10.12.90.54' updated
```

## 7. Create Argo CD Local User

### 7.1 Define the Local Account

Configure the local account through the Helm values of the `argocd-instance` chart (`spec.localAccounts` and `spec.rbac` are rendered by the chart template). Do not edit `argocd-cm` directly for this setting.

**charts/argocd-instance/values.yaml** (updated)

```yaml
name: argocd-1
namespace: argocd-instance-1
version: 3.0.19+vmware.1-vks.1
enableLoadBalancer: true
serverSideDiff: true
localAccounts:
  - <USERNAME>
rbacPolicy: |
  g, <USERNAME>, role:admin
```

```bash
helm upgrade --install argocd-instance charts/argocd-instance \
  -n argocd-instance-1
```

Alternatively, pass the values at deploy time without editing the file:

```bash
helm upgrade --install argocd-instance charts/argocd-instance \
  -n argocd-instance-1 \
  --set localAccounts[0]=<USERNAME> \
  --set-string rbacPolicy="g, <USERNAME>, role:admin"
```

The Supervisor Service exposes `spec.localAccounts` and `spec.rbac` on the ArgoCD custom resource.

### 7.2 Set the Local User Password

```bash
argocd account update-password --account <USERNAME>
```

Passwords are stored by Argo CD as bcrypt hashes, so the password is set with the Argo CD CLI and not through Helm.

Use the project's approved RBAC role. The `role:admin` example above should be replaced with a least-privilege role when applicable.

## 8. Register the VKS Workload Cluster

Retrieve the VKS kubeconfig and register the VKS cluster with the Argo CD instance.

Before registering the VKS cluster, ensure that the existing Supervisor context is being used, as described in Section 6.5. If a suitable Supervisor context is not already available, create and configure one using the VCF CLI before continuing.

Retrieve the VKS kubeconfig and identify the available context:

```bash
kubectl --kubeconfig <VKS_KUBECONFIG> config current-context

argocd cluster add <VKS_CONTEXT> --kubeconfig <VKS_KUBECONFIG>
```
Observed lab output:
```
ServiceAccount "argocd-manager" created
ClusterRole "argocd-manager-role" created
ClusterRoleBinding "argocd-manager-role-binding" created
Bearer token Secret created
Cluster 'https://10.12.90.55:6443' added
```
The registered cluster can be verified using:
```
argocd cluster list
```
output:
```
SERVER                 NAME                     STATUS
https://10.12.90.55:6443  supns3:vks-lab-12:vks-lab-12  Unknown
```
The VMware documentation shows the same approach for a guest/VKS cluster: obtain the VKS kubeconfig, identify the context, and add the cluster with the customized Argo CD CLI.

Cluster registration is performed with the Argo CD CLI, which creates the `argocd-manager` ServiceAccount, ClusterRole, ClusterRoleBinding and token on the VKS cluster and stores the cluster credentials in Argo CD. No YAML or Helm chart is needed for this step.

Important: The Supervisor context and the VKS workload-cluster context serve different purposes. The Supervisor context is used to manage the Argo CD Supervisor Service and Argo CD instance, while the VKS context is the target cluster that Argo CD manages.

## 9. GitOps and CI/CD Integration

### 9.1 GitLab GitOps Repository

Use a dedicated private GitLab repository as the source of truth for Kubernetes desired state. Keep application source code and deployment manifests logically separated so that CI produces artifacts while CD consumes the GitOps repository.

The GitOps repository stores **Helm charts** instead of plain manifests and Kustomize overlays. Environment differences (staging/production) are expressed with Helm values files.

Example repository:

```text
gitops-repo/
├── charts/
│   ├── argocd-instance/
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/
│   │       └── argocd.yaml
│   ├── argocd-apps/
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/
│   │       ├── repo-secret.yaml
│   │       ├── appproject.yaml
│   │       └── applications.yaml
│   └── sample-java/
│       ├── Chart.yaml
│       ├── values.yaml
│       ├── values-staging.yaml
│       ├── values-production.yaml
│       └── templates/
│           ├── deployment.yaml
│           └── service.yaml
```

### 9.2 CI to GitOps Flow

1. Developer pushes application code to the application GitLab repository.
2. GitLab webhook/trigger starts the Tekton pipeline.
3. Tekton builds, tests and publishes the container image to Harbor.
4. Tekton updates the approved image tag or immutable image digest in the Helm values file (`values-staging.yaml`) of the GitLab GitOps repository.
5. Tekton commits and pushes the GitOps change.
6. The GitOps commit becomes the new desired state for Argo CD.

### 9.3 Private GitLab Repository Access from Argo CD

The GitLab GitOps repository is private. Configure an Argo CD repository credential using the project-approved GitLab authentication method, such as an access token over HTTPS or an SSH deploy key. Store credentials in Argo CD repository configuration/Secrets and do not commit them to the GitOps repository.

With Helm, the repository Secret is rendered by the `argocd-apps` chart (`templates/repo-secret.yaml`) and the token is passed at install time with `--set`, never stored in `values.yaml`.

**charts/argocd-apps/templates/repo-secret.yaml**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-gitops-repo
  namespace: {{ .Values.namespace }}
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: {{ .Values.gitops.repoURL }}
  username: {{ .Values.gitops.username }}
  password: {{ .Values.gitops.token }}
```

Recommended access model: Argo CD has read access to the GitOps repository; CI retains the write capability required to update image references.

### 9.4 GitLab Webhook to Argo CD

Configure a GitLab Push Event webhook so a GitOps repository commit can notify Argo CD immediately. Example endpoint:

```text
https://argocd.apps.company.local/api/webhook
```

Configure the webhook secret according to the project security standard. Keep repository polling/reconciliation enabled as the fallback mechanism.

### 9.5 Argo CD Application and Automated Sync

The Argo CD Application points to the GitLab GitOps repository, selects the required branch/revision and the Helm chart path, and targets the registered VKS cluster. For environments approved for automated deployment, use automated sync with self-healing and pruning as defined by the project deployment policy.

The sync policy is a value in the `argocd-apps` chart:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

### 9.6 Example Image Tag / Digest Update

Before CI update (`charts/sample-java/values-staging.yaml`):

```yaml
image:
  repository: harbor.company.local/platform/sample-java
  tag: 1.0.4
```

After Tekton builds and pushes the approved image:

```yaml
image:
  repository: harbor.company.local/platform/sample-java
  tag: 1.0.5
```

Tekton edits the values file, for example:

```bash
yq -i '.image.tag = "1.0.5"' charts/sample-java/values-staging.yaml
git commit -am "sample-java staging -> 1.0.5"
git push origin main
```

Argo CD detects the new Git revision, renders the Helm chart with the environment values file, compares desired and live state, and synchronizes the VKS workload. For stronger immutability, the GitOps repository may record the image digest (`image.digest: sha256:...`) instead of a mutable tag when required by the project.

### 9.7 Deployment Responsibility

Tekton is responsible for CI activities and updating the GitOps repository. Argo CD is responsible for CD and reconciliation from Git to Kubernetes. CI should not bypass GitOps by running `helm install` or applying the application manifests directly to the VKS cluster.

## 10. Complete Workflow

```text
Developer
   |
   v
Application GitLab Repository
   |
   | Webhook / Trigger
   v
Tekton CI
   |
   v
Build / Test / Push Image
   |
   v
Harbor
   |
   | Update image tag / digest (Helm values file)
   v
GitLab GitOps Repository (Helm charts)
   |
   | Git revision / Webhook
   v
Argo CD Supervisor Service
   |
   v
Argo CD Instance
   |
   | Sync (helm template + apply)
   v
Registered VKS Cluster
   |
   v
Application
```

1. Developer commits application changes.
2. A GitLab webhook/trigger starts the Tekton pipeline.
3. Tekton builds and publishes the application image to Harbor.
4. Tekton updates the Helm values file in the GitLab GitOps repository with the approved image tag or digest and pushes the Git commit.
5. Argo CD detects the GitLab Git revision (webhook or reconciliation).
6. Argo CD renders the Helm chart, compares desired and live state.
7. Argo CD synchronizes the application to the registered VKS cluster.
8. Application deployment is validated.

## 11. GitOps Repository Layout

```text
gitops/
└── charts/
    ├── argocd-instance/
    ├── argocd-apps/
    └── sample-java/
        ├── Chart.yaml
        ├── values.yaml
        ├── values-staging.yaml
        ├── values-production.yaml
        └── templates/
```

**charts/sample-java/Chart.yaml**

```yaml
apiVersion: v2
name: sample-java
description: Sample Java application
type: application
version: 1.0.0
appVersion: "1.0.5"
```

**charts/sample-java/values.yaml**

```yaml
replicaCount: 1
image:
  repository: harbor.company.local/platform/sample-java
  tag: latest
  digest: ""
service:
  port: 8080
```

**charts/sample-java/values-staging.yaml** (updated by CI)

```yaml
replicaCount: 1
image:
  tag: 1.0.5
```

**charts/sample-java/values-production.yaml**

```yaml
replicaCount: 3
image:
  tag: 1.0.4
```

**charts/sample-java/templates/deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}{{ if .Values.image.digest }}@{{ .Values.image.digest }}{{ else }}:{{ .Values.image.tag }}{{ end }}"
          ports:
            - containerPort: {{ .Values.service.port }}
```

**charts/sample-java/templates/service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
spec:
  selector:
    app: {{ .Release.Name }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.port }}
```

## 12. Argo CD Application

The Argo CD Application objects, the AppProject and the repository Secret are deployed with the `argocd-apps` Helm chart instead of applying an Application YAML manually. The Application itself deploys the `sample-java` Helm chart to the VKS cluster.

**charts/argocd-apps/Chart.yaml**

```yaml
apiVersion: v2
name: argocd-apps
description: Argo CD repository, project and applications
type: application
version: 1.0.0
```

**charts/argocd-apps/values.yaml**

```yaml
namespace: argocd-instance-1

gitops:
  repoURL: https://gitlab.company.local/platform/gitops-repo.git
  revision: main
  username: argocd-read
  token: ""              # pass with --set at install time; never commit

destination:
  server: https://<VKS_API_SERVER>

project: platform

applications:
  - name: sample-java-staging
    path: charts/sample-java
    valueFile: values-staging.yaml
    namespace: sample-app
  # - name: sample-java-production
  #   path: charts/sample-java
  #   valueFile: values-production.yaml
  #   namespace: sample-app-prod

syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

**charts/argocd-apps/templates/appproject.yaml**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: {{ .Values.project }}
  namespace: {{ .Values.namespace }}
spec:
  sourceRepos:
    - {{ .Values.gitops.repoURL }}
  destinations:
    - server: {{ .Values.destination.server }}
      namespace: "*"
  clusterResourceWhitelist:
    - group: "*"
      kind: "*"
```

**charts/argocd-apps/templates/applications.yaml**

```yaml
{{- range .Values.applications }}
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: {{ .name }}
  namespace: {{ $.Values.namespace }}
spec:
  project: {{ $.Values.project }}
  source:
    repoURL: {{ $.Values.gitops.repoURL }}
    targetRevision: {{ $.Values.gitops.revision }}
    path: {{ .path }}
      helm:
        valueFiles:
        - {{ .valueFile }}
  destination:
    server: {{ $.Values.destination.server }}
    namespace: {{ .namespace }}
  syncPolicy:
    {{- toYaml $.Values.syncPolicy | nindent 4 }}
    syncOptions:
      - CreateNamespace=true
{{- end }}
```

Deploy the chart (Supervisor context):

```bash
vcf context use sup-01

helm lint charts/argocd-apps
helm upgrade --install argocd-apps charts/argocd-apps \
  -n argocd-instance-1 \
  --set gitops.token=$GITLAB_TOKEN \
  --set destination.server=https://10.12.90.55:6443
```

## 13. Application Validation

```bash
helm list -n argocd-instance-1

argocd app get <APP_NAME>
argocd app diff <APP_NAME>
argocd app sync <APP_NAME>
argocd app wait <APP_NAME>
```

Confirm the application reaches the expected `Synced` and `Healthy` state.

## 14. Installing Argo CD Supervisor Service — Air-Gapped Environment

The air-gapped deployment keeps the same VMware Argo CD Supervisor Service model. The service definition and its OCI image bundle are prepared in the Internet-connected environment, transferred through the approved air-gap process, imported to the internal Harbor registry, and then registered and installed on the Supervisor.

### 14.1 Install imgpkg on the Staging Host

```bash
wget -O- https://carvel.dev/install.sh > install.sh
sudo bash install.sh
imgpkg version
```

### 14.2 Obtain the Argo CD Supervisor Service YAML

1. Sign in to the Broadcom Support Portal.
2. Go to **Software > Enterprise Software > My Downloads**.
3. Search for **vSphere Supervisor Services**.
4. Open ArgoCD Service and select the approved 1.1.0 version.
5. Download the service definition YAML.

Use the company-provided Argo CD Supervisor Service 1.1.0 manifest for the target VCF/Supervisor release. For the air-gapped/restricted environment, use the depot variant of the Argo CD Supervisor Service manifest.

The depot variant references the internal Supervisor depot registry instead of the external Broadcom registry.

### 14.3 Identify the Argo CD Service Bundle

For the air-gapped environment, use the company-provided Argo CD Supervisor Service 1.1.0 manifest. Its bundle source is the internal `depot.kube-system.svc` registry path below.

```
depot.kube-system.svc/vcf/vcf-service-argocd/ga/1.1.0/argocd-service:v1.1.0_vmware.1
```

For the staging/connected host, create the offline tar from the Broadcom registry bundle in the company-provided connected-environment manifest. After transfer, use the company-provided air-gapped manifest with its internal `depot.kube-system.svc` bundle source.

### 14.4 Create the Offline imgpkg Bundle

```bash
imgpkg copy -b projects.packages.broadcom.com/vsphere/supervisor/argocd-service/1.1.0/argocd-service:v1.1.0_vmware.1 \
  --to-tar argocd-service-v1.1.0_vmware.1.tar \
  --cosign-signatures
```

The air-gap workflow uses `imgpkg copy` to export the Supervisor Service 1.2.0 bundle to a tar file from the connected environment. Keep `--cosign-signatures` to preserve the bundle signatures.

Also package the Helm charts on the connected host so they can be transferred together with the bundle:

```bash
helm package charts/argocd-instance charts/argocd-apps charts/sample-java
# Produces: argocd-instance-1.0.0.tgz, argocd-apps-1.0.0.tgz, sample-java-1.0.0.tgz
```

### 14.5 Transfer the Air-Gapped Artifacts

```
supervisor-service-argocd-depot-1.1.0-25100889.yml
argocd-service-v1.1.0_vmware.1.tar
argocd-cli-linux-amd64-v3.0.19-vcf.gz
argocd-instance-1.0.0.tgz
argocd-apps-1.0.0.tgz
sample-java-1.0.0.tgz
```

Transfer these artifacts through the organization's approved offline transfer process.

### 14.6 Import the Bundle into Internal Harbor

```bash
docker login <HARBOR_FQDN>

imgpkg copy \
  --tar argocd-service-v1.2.0_vmware.1.tar \
  --to-repo <HARBOR_FQDN>/argocd \
  --cosign-signatures
```

If Harbor uses a private CA, supply the approved CA option to imgpkg. VMware's private-registry guidance requires the Supervisor to trust and access the private registry as appropriate.

Push the Helm charts to Harbor as OCI artifacts:

```bash
helm registry login <HARBOR_FQDN>

helm push argocd-instance-1.0.0.tgz oci://<HARBOR_FQDN>/charts
helm push argocd-apps-1.0.0.tgz oci://<HARBOR_FQDN>/charts
helm push sample-java-1.0.0.tgz oci://<HARBOR_FQDN>/charts
```

Use `--ca-file <CA_FILE>` with the Helm commands if Harbor uses a private CA.

### 14.7 Update the Supervisor Service YAML

Replace the public bundle reference in the service YAML with the internal Harbor bundle reference returned by the `imgpkg copy` operation.

```yaml
template:
  spec:
    fetch:
      - imgpkgBundle:
          image: <HARBOR_FQDN>/argocd/<BUNDLE>:<TAG>
```

Keep the service YAML and the relocated bundle version aligned.

### 14.8 Configure Internal Harbor on the Supervisor

1. Open the vSphere Client.
2. Go to **Supervisor Management > Supervisors**.
3. Select the target Supervisor.
4. Open **Configure > Container Registries**.
5. Add the internal Harbor registry and configure the required CA and credentials.

VMware documents this registry configuration for private Supervisor Service image pulls.

### 14.9 Register and Install the Air-Gapped Supervisor Service

1. Go to **Supervisor Management > Services**.
2. Select the target vCenter.
3. Upload/register the modified Argo CD Supervisor Service YAML. : `supervisor-service-argocd-depot-1.1.0-25100889.yml`
4. Choose **Actions > Install on Supervisors**.
5. Select the target Supervisor.
6. Complete the compatibility checks and installation.

The Supervisor Service installation process is the same UI-driven process after the service definition has been changed to reference the private registry.

After installation, verify the Argo CD Supervisor Service package:
```
kubectl get packages -A | grep -i argocd
```

### 14.10 Create the Argo CD Instance

Deploy the `argocd-instance` Helm chart from the internal Harbor OCI registry. The ArgoCD custom resource rendered by this chart is the only resource required to create the minimum Argo CD instance; the Supervisor Service Operator creates the underlying Argo CD resources.

```bash
helm upgrade --install argocd-instance \
  oci://<HARBOR_FQDN>/charts/argocd-instance \
  --version 1.0.0 \
  -n argocd-instance-1
```

Default values used by the chart (`values.yaml`):

```yaml
name: argocd-1
namespace: argocd-instance-1
version: 3.0.19+vmware.1-vks.1
enableLoadBalancer: true
serverSideDiff: true
```

If the local `.tgz` file is used instead of the OCI registry:

```bash
helm upgrade --install argocd-instance argocd-instance-1.0.0.tgz \
  -n argocd-instance-1
```

### 14.11 Verify the Air-Gapped Argo CD Instance

```bash
helm list -n argocd-instance-1
kubectl get pods -n argocd-instance-1
kubectl get svc -n argocd-instance-1
```

- Argo CD pods are Running/Ready.
- Argo CD images resolve through internal Harbor.
- `argocd-server` has an internal LoadBalancer endpoint.
- No public registry is required during the Argo CD instance deployment.

### 14.12 Configure the Internal DNS

```
argocd.apps.company.local -> <INTERNAL_ARGOCD_LOADBALANCER_IP>

https://argocd.apps.company.local
```

### 14.13 Install the VMware-Customized Argo CD CLI in the Air-Gapped Environment

1. Transfer the approved VMware-customized Linux Argo CD CLI from the connected environment. : `argocd-cli-linux-amd64-v3.0.19-vcf.gz`
2. Copy the binary to the administration host in the air-gapped environment.
3. Make the binary executable and install it.

```bash
gunzip argocd-cli-linux-amd64-v3.0.19-vcf.gz
chmod +x argocd-cli-linux-amd64-v3.0.19-vcf
sudo install -m 555 argocd-cli-linux-amd64-v3.0.19-vcf /usr/local/bin/argocd
argocd version --client
```

Use the same VMware-customized CLI build supplied for the selected Argo CD Supervisor Service version.

### 14.14 Configure the Internal GitOps Repository

Configure the Argo CD instance to use the approved internal Git/GitOps repository. The air-gapped deployment must not depend on public GitHub for application synchronization.

Deploy the `argocd-apps` chart from Harbor, pointing `gitops.repoURL` to the internal GitLab repository:

```bash
helm upgrade --install argocd-apps \
  oci://<HARBOR_FQDN>/charts/argocd-apps \
  --version 1.0.0 \
  -n argocd-instance-1 \
  --set gitops.repoURL=https://gitlab.company.local/platform/gitops-repo.git \
  --set gitops.token=$GITLAB_TOKEN \
  --set destination.server=https://<VKS_API_SERVER>
```

Any third-party Helm chart and its container images must also be mirrored to internal Harbor, with `image.repository` overridden in the values file.

## 15. Rollback

Application rollback is performed by reverting the GitOps change (the Helm values file) and allowing Argo CD to reconcile the previous desired state.

```bash
git revert <BAD_COMMIT>
git push origin main

argocd app get <APP_NAME>
argocd app sync <APP_NAME>
argocd app wait <APP_NAME>
```

For the Helm-deployed Argo CD instance and Argo CD Application charts, use Helm rollback:

```bash
helm history argocd-instance -n argocd-instance-1
helm rollback argocd-instance <REVISION> -n argocd-instance-1

helm history argocd-apps -n argocd-instance-1
helm rollback argocd-apps <REVISION> -n argocd-instance-1
```

For a Supervisor Service upgrade, follow the Supervisor Service upgrade procedure and update `version` in the `argocd-instance` chart values (rendered to `ArgoCD.spec.version`) to the approved version supported by the installed service, then run `helm upgrade`.

## 16. Conclusion

The VMware Argo CD Supervisor Service provides the GitOps continuous delivery layer for VKS workloads. The implementation uses the VMware Argo CD Operator, an ArgoCD custom resource in a vSphere Namespace (deployed with a Helm chart), an internal LoadBalancer endpoint, internal DNS, the VMware-customized Argo CD CLI, Helm charts for the Argo CD Applications and the sample application, and the project's GitOps repository.

For the air-gapped environment, the Argo CD Supervisor Service bundle is relocated with imgpkg, the Helm charts are stored as OCI artifacts in internal Harbor, and the service is referenced by the local service YAML and installed on the Supervisor without requiring public registry access.

## Official Documentation References

- VMware by Broadcom — vSphere Supervisor 9.0 Services and Standalone Components: Using the Argo CD Supervisor Service.
- VMware by Broadcom — vSphere Supervisor 9.0 Services and Standalone Components: Manage Resources in VKS Clusters by Using Argo CD.
- VMware by Broadcom — Argo CD Custom Resource Reference.
- Supervisor Services catalog and installation guidance: https://vsphere-tmm.github.io/Supervisor-Services/
- Broadcom Support Portal: https://support.broadcom.com/
- VMware vSphere Supervisor air-gapped guide: https://github.com/vmware/vsphere-supervisor/blob/main/airgapped/air-gapped-vcf91.md
- Carvel imgpkg: https://carvel.dev/imgpkg/
- Helm: https://helm.sh/docs/
