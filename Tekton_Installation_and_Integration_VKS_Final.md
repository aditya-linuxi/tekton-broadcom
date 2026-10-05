# Tekton Installation and Integration with Buildpacks, BuildKit, and Cosign on VKS

## 1. Introduction

**VKS (vSphere Kubernetes Service):** A Kubernetes service provided by VMware Cloud Foundation for creating and managing Kubernetes clusters on vSphere infrastructure.

**Tekton:** A Kubernetes-native CI/CD framework used to create and run automated pipelines as Kubernetes resources.

**Cloud Native Buildpacks (CNB):** A technology that converts application source code into container images without requiring a Dockerfile.

**BuildKit:** A modern container image build engine used to build images from Dockerfiles efficiently and supports advanced build features such as caching and parallel builds.

**Cosign:** A tool used to digitally sign and verify container images and other OCI artifacts.

**SBOM (Software Bill of Materials):** A detailed list of software components, libraries, and dependencies contained in a container image.

**Syft:** A tool used to generate an SBOM from a container image.

**Harbor:** A private container registry used to store and manage container images and OCI security artifacts produced by the CI pipeline.

**Tekton Triggers:** A Tekton component that listens for repository events and creates PipelineRuns automatically. The main objects used here are an EventListener, TriggerBinding, TriggerTemplate, and CEL interceptor.

**Webhook:** An HTTP request sent by the repository server to the Tekton EventListener when a repository event occurs.

---

## 2. Why Use BuildKit and Buildpacks with Tekton on VKS?

The pipeline supports two build paths:

- Dockerfile present -> BuildKit builds the image.
- Dockerfile absent -> Cloud Native Buildpacks builds the image.

A repository push is the developer action. Tekton then clones the source, detects the build type, builds the image, generates an SBOM, signs the immutable image digest, and updates the GitOps manifest repository.

The complete flow is:

```text
Application Source Repository
            |
            v
      Tekton Triggers
            |
            v
         PipelineRun
            |
            v
       Clone source
            |
            v
      Detect Dockerfile
         /         \\
        /           \\
   BuildKit      Buildpacks
        \           /
         \         /
          v       v
          Built OCI Image
                |
                v
            Generate SBOM
                |
                v
         Cosign sign/attest
                |
                v
             Harbor
                |
                v
       Update GitOps values.yaml
                |
                v
              ArgoCD
                |
                v
         Application on VKS
```

> **Branch convergence:** the Pipeline keeps the existing `runAfter` design. Tekton waits for both conditional build Tasks (`buildkit` and `buildpack`) to reach a terminal state before starting `sign`. Because `sign` reads the digest from the shared workspace rather than from a skipped Task result, the selected build path can converge safely into the same signing step.

---

## 3. Benefits

| Benefit | In simple words |
|---|---|
| Automatic build selection | Dockerfile present -> BuildKit; absent -> Buildpacks. |
| Less work for application teams | A Dockerfile is optional when Buildpacks can detect and build the application. |
| Fully automated CI | A code push starts the process; build, SBOM, signing, and GitOps update are automated. |
| Signed images | Cosign signs the immutable image digest. |
| SBOM for each build | Syft creates an SPDX JSON SBOM for the image. |
| One image registry | Harbor stores application images and related OCI artifacts. |
| Traceable deployments | GitOps records the image repository, tag, and immutable digest. |
| Dashboard visibility | Tekton Dashboard provides UI visibility into Tasks, TaskRuns, Pipelines, and PipelineRuns. |
| Automatic webhook trigger | Repository push -> EventListener -> PipelineRun. |
| No CI self-trigger loop | The `ci: update TravelPortal image digest` commit is rejected by the CEL filter. |
| Connected/disconnected capable | The pipeline can run offline after all required manifests, images, builders, credentials, and dependencies are mirrored locally. |

---

## 4. Architecture

All CI components run inside the VKS cluster. The source repository and GitOps repository may be separate repositories on the same internal Git service. The IP addresses and NodePorts shown in this document are the observed lab values and are not portable defaults.

### Application CI flow

```text
Developer
   |
   | git push origin main
   v
Application source repository
   |
   | webhook
   v
Tekton EventListener
   |
   +--> CEL filter
   |      +--> main branch only
   |      +--> reject CI digest-update commit
   |
   +--> TriggerBinding
   |
   +--> TriggerTemplate
   |
   v
PipelineRun
   |
   +--> git-clone-update
   |
   +--> detect-build-type
   |       |
   |       +--> Dockerfile -> buildkit-build
   |       |
   |       +--> No Dockerfile -> buildpacks-phases
   |
   +--> sign-image
   |       |
   |       +--> Syft SBOM
   |       +--> Cosign signature
   |       +--> Cosign SBOM attestation
   |
   +--> update-values
           |
           v
       GitOps repository
           |
           v
         ArgoCD
           |
           v
        VKS workload
```

### Webhook flow

```text
Repository server
       |
       | HTTP POST / JSON
       v
NodePort
10.12.92.3:31877   <- observed lab value; verify on every cluster
       |
       v
EventListener
       |
       +--> CEL
       +--> TriggerBinding
       +--> TriggerTemplate
       |
       v
PipelineRun
```

---

## 5. Prerequisites

### 5.1 Platform

| Component | Recommended baseline | Purpose |
|---|---|---|
| VMware Cloud Foundation | 9.x | Supervisor and workload cluster management |
| VKS | VCF 9.x bundled | Kubernetes workload cluster |
| Harbor | Internal/private Harbor | Image and OCI artifact storage |
| Kubernetes | 1.28+ | Tekton Pipelines prerequisite |
| Tekton Pipelines | v1.15.0 LTS baseline | CI pipeline engine |
| Tekton Dashboard | v0.71.0 LTS baseline | Pipeline UI; tested with Pipelines v1.15.x and Triggers v0.36.x |
| Tekton Triggers | v0.36.0 baseline for Dashboard compatibility | Repository webhook processing |
| Cosign | v3.0.2 in this configuration | Image signing and SBOM attestation |
| Syft | v1.54.0 current release at document update time | SBOM generation |

Tekton Pipelines v1.15.0 is an LTS release, and Tekton Dashboard v0.71.0 documents compatibility with Pipelines v1.15.x and Triggers v0.36.x. Tekton Triggers v0.37.0 is also an LTS release, but this document uses v0.36.0 to keep the documented Dashboard compatibility set aligned. See the official references at the end of this document.

> **Version note:** This document intentionally keeps the existing Tekton Triggers `v0.36.0` tag so the documented version set and names remain unchanged. Tekton Triggers `v0.36.0` reached end of life on August 26, 2026; for a new production rollout, use a currently supported release pair and retest Dashboard compatibility before deployment.

### 5.2 Required permissions

The account installing Tekton should have cluster-admin privileges.

The CI namespace will be `cicd`.

### 5.3 Tools on the administration workstation

```bash
kubectl version
kubectl cluster-info
kubectl get nodes -o wide

helm version
git --version
curl --version

docker --version    # optional; useful for image pull/login verification
cosign version      # optional; useful for offline verification from the workstation
```

---

# 6. Install Tekton Pipelines on Internet-Connected VKS

## 6.1 Verify the Kubernetes cluster

```bash
kubectl version
kubectl cluster-info
kubectl config current-context
kubectl get nodes -o wide
```

Expected:

```text
All VKS worker nodes -> Ready
```

## 6.2 Verify administrative permissions

```bash
kubectl auth can-i create customresourcedefinitions
kubectl auth can-i create clusterroles
kubectl auth can-i create namespaces
```

Expected:

```text
yes
yes
yes
```

## 6.3 Verify Internet connectivity from the workstation

A successful ping is not sufficient. Test the HTTPS endpoints used for installation.

```bash
curl -I https://infra.tekton.dev
curl -I https://registry-1.docker.io/v2/
curl -I https://ghcr.io/v2/
```

## 6.4 Verify connectivity from inside the cluster

```bash
kubectl run net-test \\
  --rm -it \\
  --restart=Never \\
  --image=curlimages/curl \\
  -- \\
  curl -I https://registry-1.docker.io/v2/
```

If the cluster cannot pull `curlimages/curl`, use an already available diagnostic image or an existing Pod that contains `curl`.

---

## 6.5 Install Tekton Pipelines v1.15.0

```bash
kubectl apply -f \\
  https://infra.tekton.dev/tekton-releases/pipeline/previous/v1.15.0/release.yaml
```

Check the namespace:

```bash
kubectl get namespace tekton-pipelines
```

Check the Pods:

```bash
kubectl get pods -n tekton-pipelines
```

Watch until all main components are ready:

```bash
kubectl get pods -n tekton-pipelines -w
```

Expected components include:

```text
tekton-pipelines-controller-xxxxx   1/1   Running
tekton-pipelines-webhook-xxxxx      1/1   Running
```

Check all resources:

```bash
kubectl get all -n tekton-pipelines
```

---

## 6.6 Verify Tekton CRDs

```bash
kubectl get crd | grep tekton
kubectl api-resources | grep tekton
```

Important resources include:

```text
tasks.tekton.dev
taskruns.tekton.dev
pipelines.tekton.dev
pipelineruns.tekton.dev
```

Conceptually:

```text
Task       -> TaskRun
Pipeline   -> PipelineRun
```

## 6.7 Verify the Tekton controller

```bash
kubectl get deployment -n tekton-pipelines
kubectl logs deployment/tekton-pipelines-controller \\
  -n tekton-pipelines \\
  --tail=50
```

---

# 7. Create the CI/CD Namespace

Use a separate namespace for application CI resources.

```bash
kubectl create namespace cicd --dry-run=client -o yaml | kubectl apply -f -
```

Verify:

```bash
kubectl get namespace cicd
```

For the current BuildKit/Buildpacks lab implementation, label the namespace:

```bash
kubectl label namespace cicd \\
  pod-security.kubernetes.io/enforce=privileged \\
  pod-security.kubernetes.io/audit=privileged \\
  pod-security.kubernetes.io/warn=privileged \\
  --overwrite
```

> **Security note:** The privileged Pod Security setting is required by this lab implementation because the BuildKit Task explicitly uses `privileged: true`, and the Buildpacks lifecycle task uses root/capabilities for extension handling. In production, use the least-privilege configuration supported by the selected builders and cluster security policy.

---

# 8. Install Tekton Dashboard

The official Dashboard tutorial uses `release-full.yaml` for read-write mode.

Install Dashboard v0.71.0:

```bash
kubectl apply -f \\
  https://infra.tekton.dev/tekton-releases/dashboard/previous/v0.71.0/release-full.yaml
```

Check the Dashboard Pod:

```bash
kubectl get pods -n tekton-pipelines
kubectl get deployment -n tekton-pipelines
```

Expected:

```text
tekton-dashboard    1/1
```

Check the Service:

```bash
kubectl get svc tekton-dashboard -n tekton-pipelines
```

The Dashboard is normally exposed as `ClusterIP`.

## 8.1 Change Dashboard Service to NodePort

For a lab or internal environment:

```bash
kubectl patch svc tekton-dashboard \\
  -n tekton-pipelines \\
  -p '{"spec":{"type":"NodePort"}}'
```

Get the assigned NodePort:

```bash
kubectl get svc tekton-dashboard -n tekton-pipelines
```

Example:

```text
NAME               TYPE       CLUSTER-IP      PORT(S)
tekton-dashboard   NodePort   10.x.x.x        9097:30583/TCP
```

The NodePort is cluster-assigned. Do not assume `30583` on another cluster.

Get a VKS node IP:

```bash
kubectl get nodes -o wide
```

Open:

```text
http://<node-ip>:<node-port>
```

For production exposure, use an approved ingress/load balancer with HTTPS, DNS, authentication, RBAC, and network restrictions.

---

# 9. Validate Tekton with a Simple Task

## 9.1 Create `test-task.yaml`

```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: hello-task
  namespace: cicd
spec:
  steps:
    - name: hello
      image: alpine:3.20
      script: |
        #!/bin/sh
        echo "Hello from Tekton!"
        echo "Tekton Pipeline integration is working."
```

Apply:

```bash
kubectl apply -f test-task.yaml
```

Verify:

```bash
kubectl get task hello-task -n cicd
```

## 9.2 Create `hello-taskrun.yaml`

```yaml
apiVersion: tekton.dev/v1
kind: TaskRun
metadata:
  name: hello-taskrun
  namespace: cicd
spec:
  taskRef:
    name: hello-task
```

Apply:

```bash
kubectl apply -f hello-taskrun.yaml
```

Check:

```bash
kubectl get taskrun -n cicd
kubectl get taskrun -n cicd -w
```

Get logs:

```bash
kubectl logs -l tekton.dev/taskRun=hello-taskrun -n cicd
```

Expected:

```text
Hello from Tekton!
Tekton Pipeline integration is working.
```

Open Dashboard and select namespace `cicd` to confirm the TaskRun is visible.

---

# 10. Build Pipeline Tasks

The following Tasks are used by the main Pipeline:

```text
1. git-clone-update
2. detect-build-type
3. buildkit-build
4. buildpacks-phases
5. sign-image
6. update-values
```

---

## 10.1 Git Clone Task

Create `git-clone-update-task.yaml`:

```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: git-clone-update
  namespace: cicd
spec:
  description: Clone a Git repository

  params:
    - name: url
      type: string
      description: Git repository URL

    - name: revision
      type: string
      description: Git branch or tag
      default: main

    - name: deleteExisting
      type: string
      description: Delete existing workspace contents
      default: "true"

  steps:
    - name: clone
      image: alpine/git:latest
      computeResources: {}
      script: |
        #!/bin/sh
        set -eu

        WORKSPACE="$(workspaces.output.path)"

        echo "========================================"
        echo "Git Clone"
        echo "========================================"
        echo "Workspace: ${WORKSPACE}"
        echo "Repository: $(params.url)"
        echo "Revision:   $(params.revision)"

        if [ "$(params.deleteExisting)" = "true" ]; then
          echo "Cleaning workspace..."
          rm -rf "${WORKSPACE}"/*
          rm -rf "${WORKSPACE}"/.[!.]*
          rm -rf "${WORKSPACE}"/..?*
        fi

        if [ "$(workspaces.git-credentials.bound)" = "true" ]; then
          GIT_USERNAME="$(cat $(workspaces.git-credentials.path)/username)"
          GIT_TOKEN="$(cat $(workspaces.git-credentials.path)/token)"
          git config --global credential.helper '!f() { if [ "$1" = get ]; then printf "protocol=http\\nusername=%s\\npassword=%s\\n\\n" "$GIT_USERNAME" "$GIT_TOKEN"; fi; }; f'
          echo "Repository credentials configured."
        fi

        echo "Cloning repository..."
        git clone \
          --branch "$(params.revision)" \
          --depth 1 \
          "$(params.url)" \
          "${WORKSPACE}"

        git config --global --add safe.directory "${WORKSPACE}"

        cd "${WORKSPACE}"
        git status
        git branch --show-current
        git log -1 --oneline

        echo "Repository cloned successfully."

  workspaces:
    - name: output
      description: Workspace where the Git repository will be cloned

    - name: git-credentials
      description: Optional Git credentials containing username and token
      optional: true
```

Apply and verify:

```bash
kubectl apply -f git-clone-update-task.yaml
kubectl get task git-clone-update -n cicd
```

---

## 10.2 Detect Build Type Task

Create `detect-build-type.yaml`:

```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: detect-build-type
  namespace: cicd
spec:
  workspaces:
    - name: source

  results:
    - name: BUILD_TYPE
      description: "Build type: buildkit or buildpack"

  steps:
    - name: detect
      image: alpine:3.20
      script: |
        #!/bin/sh
        set -eu

        cd "$(workspaces.source.path)"

        if [ -f Dockerfile ]; then
          echo "Dockerfile found. Using BuildKit."
          printf 'buildkit' > "$(results.BUILD_TYPE.path)"
        else
          echo "Dockerfile not found. Using Buildpacks."
          printf 'buildpack' > "$(results.BUILD_TYPE.path)"
        fi
```

Apply and verify:

```bash
kubectl apply -f detect-build-type.yaml
kubectl get task detect-build-type -n cicd
```

---

# 11. Buildpacks Task

The Buildpacks Task below runs the Cloud Native Buildpacks lifecycle phases separately. It uses the configured builder image and exports the application image to Harbor.

Create `buildpacks-phases.yaml`:

```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: buildpacks-phases
  namespace: cicd
  labels:
    app.kubernetes.io/version: "0.3"
  annotations:
    tekton.dev/categories: Image Build, Security
    tekton.dev/pipelines.minVersion: "0.62.0"
    tekton.dev/tags: image-build
    tekton.dev/displayName: "Buildpacks phases"
    tekton.dev/platforms: "linux/amd64"
spec:
  description: >-
    Build application source into a container image using Cloud Native Buildpacks
    lifecycle phases and push the resulting image to the configured registry.

  workspaces:
    - name: source
      description: Directory where application source is located.
    - name: cache
      description: Directory where cache is stored.
      optional: true
    - name: dockerconfig
      description: Docker config for registry authentication.
      optional: true

  params:
    - name: CNB_BUILD_IMAGE
      description: Reference to the current build image in an OCI registry.
      default: ""
    - name: CNB_BUILDER_IMAGE
      description: Builder image containing lifecycle, buildpacks and metadata.
    - name: CNB_CACHE_IMAGE
      description: Cache image reference when a cache image is used.
      default: ""
    - name: CNB_ENV_VARS
      type: array
      description: Environment variables to set during build.
      default: []
    - name: CNB_EXPERIMENTAL_MODE
      description: Lifecycle experimental mode.
      default: silent
    - name: CNB_GROUP_ID
      description: Group ID of the builder image user.
      default: ""
    - name: CNB_INSECURE_REGISTRIES
      description: Comma-separated registries where TLS verification is skipped.
      default: ""
    - name: CNB_LAYERS_DIR
      description: Path to layers directory.
      default: /layers
    - name: CNB_LOG_LEVEL
      description: Log level.
      default: "info"
    - name: CNB_PLATFORM_API_SUPPORTED
      description: Buildpacks Platform API supported by this Task.
      default: "0.13"
    - name: CNB_PLATFORM_API
      description: User selected Buildpacks Platform API.
      default: ""
    - name: CNB_PLATFORM_DIR
      description: Platform directory.
      default: /platform
    - name: CNB_PROCESS_TYPE
      description: Default process type.
      default: ""
    - name: CNB_RUN_IMAGE
      description: Runtime image.
      default: ""
    - name: CNB_SKIP_LAYERS
      description: Skip restore SBOM layer.
      default: false
    - name: CNB_USER_ID
      description: User ID of the builder image user.
      default: ""
    - name: APP_IMAGE
      description: Application image.
    - name: SOURCE_SUBPATH
      description: Source subpath inside source workspace.
      default: ""
    - name: TAGS
      description: Additional image tags.
      default: ""
    - name: USER_HOME
      description: Absolute path to user home.
      default: /tekton/home
    - name: INSPECT_TOOLS_IMAGE
      description: Image containing skopeo and jq.
      default: quay.io/halkyonio/skopeo-jq:0.1.3@sha256:1b3d21ad541227dc9d3e793d18cef9eb00a969c0c01eb09cab88997bc63680c6

  results:
    - name: APP_IMAGE_DIGEST
      description: Digest of the built APP_IMAGE.

  stepTemplate:
    env:
      - name: CNB_EXPERIMENTAL_MODE
        value: $(params.CNB_EXPERIMENTAL_MODE)
      - name: HOME
        value: $(params.USER_HOME)
      - name: DOCKER_CONFIG
        value: /tekton/home/.docker

  steps:
    - name: get-labels-and-env
      image: $(params.INSPECT_TOOLS_IMAGE)
      onError: stopAndFail
      env:
        - name: PARAM_VERBOSE
          value: $(params.CNB_LOG_LEVEL)
        - name: PARAM_BUILDER_IMAGE
          value: $(params.CNB_BUILDER_IMAGE)
        - name: PARAM_CNB_PLATFORM_API
          value: $(params.CNB_PLATFORM_API)
        - name: PARAM_CNB_PLATFORM_API_SUPPORTED
          value: $(params.CNB_PLATFORM_API_SUPPORTED)
      results:
        - name: UID
          description: UID of the user in the builder image.
        - name: GID
          description: GID of the user in the builder image.
        - name: EXTENSION_LABELS
          description: Buildpack extension labels.
        - name: CNB_PLATFORM_API
          description: Selected Platform API.
      script: |
        #!/bin/sh
        set -eu

        if [ "${PARAM_VERBOSE}" = "debug" ]; then
          set -x
        fi

        mkdir -p /tekton/home/.docker

        if [ -f "$(workspaces.dockerconfig.path)/.dockerconfigjson" ]; then
          cp "$(workspaces.dockerconfig.path)/.dockerconfigjson" /tekton/home/.docker/config.json
        elif [ -f "$(workspaces.dockerconfig.path)/config.json" ]; then
          cp "$(workspaces.dockerconfig.path)/config.json" /tekton/home/.docker/config.json
        elif [ -f "$(workspaces.source.path)/$(params.SOURCE_SUBPATH)/.docker/config.json" ]; then
          cp "$(workspaces.source.path)/$(params.SOURCE_SUBPATH)/.docker/config.json" /tekton/home/.docker/config.json
        fi

        if [ ! -f "$HOME/.docker/config.json" ]; then
          echo "ERROR: registry credentials were not found. Harbor authentication is required."
          exit 1
        fi

        if [ -f /etc/harbor-ca/ca.crt ]; then
          if [ -f /etc/ssl/certs/ca-certificates.crt ]; then
            cat /etc/ssl/certs/ca-certificates.crt /etc/harbor-ca/ca.crt > /tmp/cnb-ca-bundle.crt
          elif [ -f /etc/pki/tls/certs/ca-bundle.crt ]; then
            cat /etc/pki/tls/certs/ca-bundle.crt /etc/harbor-ca/ca.crt > /tmp/cnb-ca-bundle.crt
          else
            cp /etc/harbor-ca/ca.crt /tmp/cnb-ca-bundle.crt
          fi
          export SSL_CERT_FILE=/tmp/cnb-ca-bundle.crt
        fi

        CLEANED_IMAGE="${PARAM_BUILDER_IMAGE%@*}"

        IMG_MANIFEST=$(skopeo inspect --authfile "$HOME/.docker/config.json" "docker://${CLEANED_IMAGE}")
        IMG_LABELS=$(printf '%s' "$IMG_MANIFEST" | jq -c '.Labels // {}')

        BUILDER_LABEL='io.buildpacks.builder.metadata'
        EXT_LABEL='io.buildpacks.extension.layers'
        BUILDER_LABEL_JSON=$(printf '%s' "$IMG_LABELS" | jq -c --arg key "$BUILDER_LABEL" '.[$key] // empty')

        CNB_PLATFORM_API="${PARAM_CNB_PLATFORM_API:-$PARAM_CNB_PLATFORM_API_SUPPORTED}"

        if [ -n "$BUILDER_LABEL_JSON" ] && [ "$BUILDER_LABEL_JSON" != "null" ]; then
          PLATFORM_APIS=$(printf '%s' "$BUILDER_LABEL_JSON" | jq -r '.lifecycle.apis.platform.supported[]')
          if printf '%s\n' "$PLATFORM_APIS" | grep -Fxq "$CNB_PLATFORM_API"; then
            printf '%s' "$CNB_PLATFORM_API" > "$(step.results.CNB_PLATFORM_API.path)"
          else
            echo "Selected platform API is not supported by the builder image: ${CNB_PLATFORM_API}"
            echo "Supported platform APIs:"
            printf '%s\n' "$PLATFORM_APIS"
            exit 1
          fi
        else
          printf '%s' "$PARAM_CNB_PLATFORM_API_SUPPORTED" > "$(step.results.CNB_PLATFORM_API.path)"
        fi

        EXTENSION_LABELS_JSON=$(printf '%s' "$IMG_LABELS" | jq -c --arg key "$EXT_LABEL" '.[$key] // empty')
        if [ -n "$EXTENSION_LABELS_JSON" ] && [ "$EXTENSION_LABELS_JSON" != "null" ]; then
          printf '%s' "$EXTENSION_LABELS_JSON" > "$(step.results.EXTENSION_LABELS.path)"
        else
          printf '%s' "empty" > "$(step.results.EXTENSION_LABELS.path)"
        fi

        CNB_USER_ID=$(printf '%s' "$IMG_MANIFEST" | jq -r '.Env[]? | select(startswith("CNB_USER_ID=")) | sub("^CNB_USER_ID="; "")' | head -n 1)
        CNB_GROUP_ID=$(printf '%s' "$IMG_MANIFEST" | jq -r '.Env[]? | select(startswith("CNB_GROUP_ID=")) | sub("^CNB_GROUP_ID="; "")' | head -n 1)

        if [ -z "$CNB_USER_ID" ] || [ -z "$CNB_GROUP_ID" ]; then
          echo "ERROR: Builder image does not expose CNB_USER_ID/CNB_GROUP_ID."
          exit 1
        fi

        printf '%s' "$CNB_USER_ID" > "$(step.results.UID.path)"
        printf '%s' "$CNB_GROUP_ID" > "$(step.results.GID.path)"
      volumeMounts:
        - name: harbor-ca
          mountPath: /etc/harbor-ca
          readOnly: true

    - name: prepare
      image: registry.access.redhat.com/ubi8/ubi-minimal@sha256:b2a1bec3dfbc7a14a1d84d98934dfe8fdde6eb822a211286601cf109cbccb075
      args:
        - "--env-vars"
        - "$(params.CNB_ENV_VARS[*])"
      env:
        - name: CNB_USER_ID
          value: $(steps.get-labels-and-env.results.UID)
        - name: CNB_GROUP_ID
          value: $(steps.get-labels-and-env.results.GID)
      script: |
        #!/bin/sh
        set -eu

        if [ "$(workspaces.cache.bound)" = "true" ]; then
          chown -R "$CNB_USER_ID:$CNB_GROUP_ID" "$(workspaces.cache.path)"
        fi

        mkdir -p /tekton/home/.docker /platform/env

        for path in "/tekton/home" "/tekton/home/.docker" "/tekton/creds" "/layers" "$(workspaces.source.path)"; do
          if [ -e "$path" ]; then
            chown -R "$CNB_USER_ID:$CNB_GROUP_ID" "$path"
          fi
        done

        parsing_envs="false"
        for arg in "$@"; do
          if [ "$arg" = "--env-vars" ]; then
            parsing_envs="true"
            continue
          fi

          if [ "$parsing_envs" = "true" ]; then
            key=${arg%%=*}
            value=${arg#*=}
            if [ -n "$key" ] && [ "$key" != "$arg" ]; then
              printf '%s' "$value" > "/platform/env/${key}"
            fi
          fi
        done

    - name: analyze
      image: $(params.CNB_BUILDER_IMAGE)
      imagePullPolicy: Always
      env:
        - name: CNB_PLATFORM_API
          value: $(steps.get-labels-and-env.results.CNB_PLATFORM_API)
      script: |
        #!/bin/sh
        set -eu

        if [ -f /etc/harbor-ca/ca.crt ]; then
          if [ -f /etc/ssl/certs/ca-certificates.crt ]; then
            cat /etc/ssl/certs/ca-certificates.crt /etc/harbor-ca/ca.crt > /tmp/cnb-ca-bundle.crt
          elif [ -f /etc/pki/tls/certs/ca-bundle.crt ]; then
            cat /etc/pki/tls/certs/ca-bundle.crt /etc/harbor-ca/ca.crt > /tmp/cnb-ca-bundle.crt
          else
            cp /etc/harbor-ca/ca.crt /tmp/cnb-ca-bundle.crt
          fi
          export SSL_CERT_FILE=/tmp/cnb-ca-bundle.crt
        fi

        exec /cnb/lifecycle/analyzer "$@"
      args:
        - "-log-level=$(params.CNB_LOG_LEVEL)"
        - "-layers=$(params.CNB_LAYERS_DIR)"
        - "-cache-image=$(params.CNB_CACHE_IMAGE)"
        - "-uid=$(steps.get-labels-and-env.results.UID)"
        - "-gid=$(steps.get-labels-and-env.results.GID)"
        - "-insecure-registry=$(params.CNB_INSECURE_REGISTRIES)"
        - "-tag=$(params.TAGS)"
        - "-skip-layers=$(params.CNB_SKIP_LAYERS)"
        - "$(params.APP_IMAGE)"
      volumeMounts:
        - name: harbor-ca
          mountPath: /etc/harbor-ca
          readOnly: true
        - name: layers-dir
          mountPath: /layers

    - name: detect
      image: $(params.CNB_BUILDER_IMAGE)
      imagePullPolicy: Always
      env:
        - name: CNB_PLATFORM_API
          value: $(steps.get-labels-and-env.results.CNB_PLATFORM_API)
      script: |
        #!/bin/sh
        set -eu

        if [ -f /etc/harbor-ca/ca.crt ]; then
          if [ -f /etc/ssl/certs/ca-certificates.crt ]; then
            cat /etc/ssl/certs/ca-certificates.crt /etc/harbor-ca/ca.crt > /tmp/cnb-ca-bundle.crt
          elif [ -f /etc/pki/tls/certs/ca-bundle.crt ]; then
            cat /etc/pki/tls/certs/ca-bundle.crt /etc/harbor-ca/ca.crt > /tmp/cnb-ca-bundle.crt
          else
            cp /etc/harbor-ca/ca.crt /tmp/cnb-ca-bundle.crt
          fi
          export SSL_CERT_FILE=/tmp/cnb-ca-bundle.crt
        fi

        exec /cnb/lifecycle/detector "$@"
      args:
        - "-log-level=$(params.CNB_LOG_LEVEL)"
        - "-app=$(workspaces.source.path)/$(params.SOURCE_SUBPATH)"
        - "-group=/layers/group.toml"
        - "-plan=/layers/plan.toml"
        - "-layers=$(params.CNB_LAYERS_DIR)"
        - "-platform=$(params.CNB_PLATFORM_DIR)"
      volumeMounts:
        - name: harbor-ca
          mountPath: /etc/harbor-ca
          readOnly: true
        - name: layers-dir
          mountPath: /layers
        - name: platform-dir
          mountPath: $(params.CNB_PLATFORM_DIR)
        - name: tekton-home-dir
          mountPath: /tekton/home

    - name: restore
      image: $(params.CNB_BUILDER_IMAGE)
      imagePullPolicy: Always
      env:
        - name: UID
          value: $(steps.get-labels-and-env.results.UID)
        - name: GID
          value: $(steps.get-labels-and-env.results.GID)
        - name: CNB_LOG_LEVEL
          value: $(params.CNB_LOG_LEVEL)
        - name: CNB_BUILD_IMAGE
          value: $(params.CNB_BUILD_IMAGE)
        - name: CNB_BUILDER_IMAGE
          value: $(params.CNB_BUILDER_IMAGE)
        - name: CNB_CACHE_IMAGE
          value: $(params.CNB_CACHE_IMAGE)
        - name: CNB_INSECURE_REGISTRIES
          value: $(params.CNB_INSECURE_REGISTRIES)
        - name: CNB_SKIP_LAYERS
          value: $(params.CNB_SKIP_LAYERS)
        - name: CNB_PLATFORM_API
          value: $(steps.get-labels-and-env.results.CNB_PLATFORM_API)
        - name: CNB_LAYERS_DIR
          value: $(params.CNB_LAYERS_DIR)
      script: |
        #!/usr/bin/env bash
        set -eu
        export BUILD_IMAGE=${CNB_BUILD_IMAGE:-${CNB_BUILDER_IMAGE}}
        /cnb/lifecycle/restorer \
          -log-level=${CNB_LOG_LEVEL} \
          -build-image=${BUILD_IMAGE} \
          -group=/layers/group.toml \
          -layers=${CNB_LAYERS_DIR} \
          -cache-dir=$(workspaces.cache.path) \
          -cache-image=${CNB_CACHE_IMAGE} \
          -uid=${UID} \
          -gid=${GID} \
          -insecure-registry=${CNB_INSECURE_REGISTRIES} \
          -skip-layers=${CNB_SKIP_LAYERS}
      volumeMounts:
        - name: harbor-ca
          mountPath: /etc/harbor-ca
          readOnly: true
        - name: layers-dir
          mountPath: /layers

    - name: extender
      when:
        - input: $(steps.get-labels-and-env.results.EXTENSION_LABELS)
          operator: notin
          values: ["empty"]
      image: $(params.CNB_BUILDER_IMAGE)
      imagePullPolicy: Always
      env:
        - name: CNB_PLATFORM_API
          value: $(steps.get-labels-and-env.results.CNB_PLATFORM_API)
      script: |
        #!/bin/sh
        set -eu

        if [ -f /etc/harbor-ca/ca.crt ]; then
          if [ -f /etc/ssl/certs/ca-certificates.crt ]; then
            cat /etc/ssl/certs/ca-certificates.crt /etc/harbor-ca/ca.crt > /tmp/cnb-ca-bundle.crt
          elif [ -f /etc/pki/tls/certs/ca-bundle.crt ]; then
            cat /etc/pki/tls/certs/ca-bundle.crt /etc/harbor-ca/ca.crt > /tmp/cnb-ca-bundle.crt
          else
            cp /etc/harbor-ca/ca.crt /tmp/cnb-ca-bundle.crt
          fi
          export SSL_CERT_FILE=/tmp/cnb-ca-bundle.crt
        fi

        exec /cnb/lifecycle/extender "$@"
      args:
        - "-log-level=$(params.CNB_LOG_LEVEL)"
        - "-app=$(workspaces.source.path)/$(params.SOURCE_SUBPATH)"
        - "-generated=/layers/generated"
        - "-uid=$(steps.get-labels-and-env.results.UID)"
        - "-gid=$(steps.get-labels-and-env.results.GID)"
        - "-platform=$(params.CNB_PLATFORM_DIR)"
      securityContext:
        runAsUser: 0
        runAsGroup: 0
        capabilities:
          add:
            - SYS_ADMIN
            - SETFCAP
      volumeMounts:
        - name: harbor-ca
          mountPath: /etc/harbor-ca
          readOnly: true
        - name: layers-dir
          mountPath: /layers
        - name: tekton-home-dir
          mountPath: /tekton/home
        - name: platform-dir
          mountPath: $(params.CNB_PLATFORM_DIR)

    - name: build
      when:
        - input: $(steps.get-labels-and-env.results.EXTENSION_LABELS)
          operator: in
          values: ["empty"]
      image: $(params.CNB_BUILDER_IMAGE)
      imagePullPolicy: Always
      env:
        - name: CNB_PLATFORM_API
          value: $(steps.get-labels-and-env.results.CNB_PLATFORM_API)
      script: |
        #!/bin/sh
        set -eu

        if [ -f /etc/harbor-ca/ca.crt ]; then
          if [ -f /etc/ssl/certs/ca-certificates.crt ]; then
            cat /etc/ssl/certs/ca-certificates.crt /etc/harbor-ca/ca.crt > /tmp/cnb-ca-bundle.crt
          elif [ -f /etc/pki/tls/certs/ca-bundle.crt ]; then
            cat /etc/pki/tls/certs/ca-bundle.crt /etc/harbor-ca/ca.crt > /tmp/cnb-ca-bundle.crt
          else
            cp /etc/harbor-ca/ca.crt /tmp/cnb-ca-bundle.crt
          fi
          export SSL_CERT_FILE=/tmp/cnb-ca-bundle.crt
        fi

        exec /cnb/lifecycle/builder "$@"
      args:
        - "-log-level=$(params.CNB_LOG_LEVEL)"
        - "-app=$(workspaces.source.path)/$(params.SOURCE_SUBPATH)"
        - "-layers=$(params.CNB_LAYERS_DIR)"
        - "-group=/layers/group.toml"
        - "-plan=/layers/plan.toml"
        - "-platform=$(params.CNB_PLATFORM_DIR)"
      volumeMounts:
        - name: harbor-ca
          mountPath: /etc/harbor-ca
          readOnly: true
        - name: layers-dir
          mountPath: /layers
        - name: platform-dir
          mountPath: $(params.CNB_PLATFORM_DIR)
        - name: tekton-home-dir
          mountPath: /tekton/home

    - name: export
      image: $(params.CNB_BUILDER_IMAGE)
      imagePullPolicy: Always
      env:
        - name: CNB_PLATFORM_API
          value: $(steps.get-labels-and-env.results.CNB_PLATFORM_API)
      script: |
        #!/bin/sh
        set -eu

        if [ -f /etc/harbor-ca/ca.crt ]; then
          if [ -f /etc/ssl/certs/ca-certificates.crt ]; then
            cat /etc/ssl/certs/ca-certificates.crt /etc/harbor-ca/ca.crt > /tmp/cnb-ca-bundle.crt
          elif [ -f /etc/pki/tls/certs/ca-bundle.crt ]; then
            cat /etc/pki/tls/certs/ca-bundle.crt /etc/harbor-ca/ca.crt > /tmp/cnb-ca-bundle.crt
          else
            cp /etc/harbor-ca/ca.crt /tmp/cnb-ca-bundle.crt
          fi
          export SSL_CERT_FILE=/tmp/cnb-ca-bundle.crt
        fi

        exec /cnb/lifecycle/exporter "$@"
      args:
        - "-log-level=$(params.CNB_LOG_LEVEL)"
        - "-app=$(workspaces.source.path)/$(params.SOURCE_SUBPATH)"
        - "-layers=$(params.CNB_LAYERS_DIR)"
        - "-group=/layers/group.toml"
        - "-cache-dir=$(workspaces.cache.path)"
        - "-cache-image=$(params.CNB_CACHE_IMAGE)"
        - "-report=/layers/report.toml"
        - "-process-type=$(params.CNB_PROCESS_TYPE)"
        - "-uid=$(steps.get-labels-and-env.results.UID)"
        - "-gid=$(steps.get-labels-and-env.results.GID)"
        - "-insecure-registry=$(params.CNB_INSECURE_REGISTRIES)"
        - "$(params.APP_IMAGE)"
      volumeMounts:
        - name: harbor-ca
          mountPath: /etc/harbor-ca
          readOnly: true
        - name: layers-dir
          mountPath: /layers

    - name: results
      image: registry.access.redhat.com/ubi8/python-311@sha256:43605cb2491ef2297a7acf4b4bf0b7f54f0c91b96daf12ae41c49cc7f192b153
      script: |
        #!/usr/bin/env python3
        import tomllib

        with open("/layers/report.toml", "rb") as f:
            data = tomllib.load(f)

        img_data = data.get("image", {})
        digest = img_data.get("digest")

        print(f"tags: {img_data.get('tags')}")
        print(f"Digest: {digest}")

        if not digest or not str(digest).startswith("sha256:"):
            raise SystemExit(f"Invalid image digest: {digest}")

        with open("$(results.APP_IMAGE_DIGEST.path)", "w") as f:
            f.write(digest)

        with open("$(workspaces.source.path)/image-digest", "w") as f:
            f.write(digest)
      volumeMounts:
        - name: layers-dir
          mountPath: /layers

  volumes:
    - name: harbor-ca
      configMap:
        name: harbor-ca-cert
    - name: tekton-home-dir
      emptyDir: {}
    - name: layers-dir
      emptyDir: {}
    - name: platform-dir
      emptyDir: {}
```

Apply:

```bash
kubectl apply -f buildpacks-phases.yaml
kubectl get task buildpacks-phases -n cicd
```

---

# 12. BuildKit Task

Create `buildkit-build-task.yaml`:

```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: buildkit-build
  namespace: cicd
spec:
  params:
    - name: IMAGE
      type: string

    - name: DOCKERFILE
      type: string
      default: Dockerfile

    - name: CONTEXT
      type: string
      default: .

  results:
    - name: IMAGE_DIGEST
      description: Digest of the pushed image
      type: string

  workspaces:
    - name: source
    - name: dockerconfig

  steps:
    - name: build
      image: moby/buildkit:latest
      env:
        - name: DOCKER_CONFIG
          value: /tekton/home/.docker
      securityContext:
        privileged: true
      volumeMounts:
        - name: harbor-ca
          mountPath: /etc/buildkit/certs
          readOnly: true
      script: |
        #!/bin/sh
        set -eu

        echo "Starting BuildKit build"
        echo "IMAGE: $(params.IMAGE)"
        echo "DOCKERFILE: $(params.DOCKERFILE)"
        echo "CONTEXT: $(params.CONTEXT)"

        test -f "$(workspaces.source.path)/$(params.DOCKERFILE)"

        mkdir -p "${DOCKER_CONFIG}"

        if [ -f "$(workspaces.dockerconfig.path)/.dockerconfigjson" ]; then
          cp "$(workspaces.dockerconfig.path)/.dockerconfigjson" "${DOCKER_CONFIG}/config.json"
        elif [ -f "$(workspaces.dockerconfig.path)/config.json" ]; then
          cp "$(workspaces.dockerconfig.path)/config.json" "${DOCKER_CONFIG}/config.json"
        else
          echo "ERROR: Harbor registry credentials were not mounted."
          exit 1
        fi

        cat > /tmp/buildkitd.toml <<EOF
        [registry."lab25-harbor.lab25.sunfire.lab"]
          ca = ["/etc/buildkit/certs/ca.crt"]
        EOF

        cat /tmp/buildkitd.toml

        cd "$(workspaces.source.path)"

        BUILDKITD_FLAGS="--config /tmp/buildkitd.toml" \
        buildctl-daemonless.sh build \
          --frontend dockerfile.v0 \
          --local context="$(workspaces.source.path)/$(params.CONTEXT)" \
          --local dockerfile="$(workspaces.source.path)" \
          --opt filename="$(params.DOCKERFILE)" \
          --output type=image,name="$(params.IMAGE)",push=true,name-canonical=true \
          --metadata-file=/tmp/build-metadata.json

        cat /tmp/build-metadata.json

        IMAGE_DIGEST=$(grep '"containerimage.digest"' /tmp/build-metadata.json \
          | sed 's/.*"containerimage.digest"[[:space:]]*:[[:space:]]*"\\([^" ]*\\)".*/\\1/')

        if [ -z "${IMAGE_DIGEST}" ]; then
          echo "ERROR: BuildKit did not return an image digest."
          exit 1
        fi

        case "${IMAGE_DIGEST}" in
          sha256:*) ;;
          *)
            echo "ERROR: Invalid image digest: ${IMAGE_DIGEST}"
            exit 1
            ;;
        esac

        printf '%s' "${IMAGE_DIGEST}" > "$(results.IMAGE_DIGEST.path)"
        printf '%s' "${IMAGE_DIGEST}" > "$(workspaces.source.path)/image-digest"

        echo "Image digest: ${IMAGE_DIGEST}"

```

Apply:

```bash
kubectl apply -f buildkit-build-task.yaml
kubectl get task buildkit-build -n cicd
kubectl describe task buildkit-build -n cicd
```

> The BuildKit Task uses the Harbor CA and does not enable `registry.insecure=true` for the Harbor HTTPS registry.

---

# 13. Cosign / SBOM Task

This Task:

1. Reads the immutable digest from the shared workspace.
2. Generates an SPDX JSON SBOM using Syft.
3. Signs the immutable image digest using Cosign.
4. Attaches the SBOM as a Cosign attestation.

`COSIGN_TLOG_UPLOAD=false` is used for the private/air-gapped lab flow. Enable transparency-log upload only when the environment is explicitly allowed to reach the approved Sigstore services.

Create `sign-image-task.yaml`:

```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: sign-image
  namespace: cicd
spec:
  params:
    - name: IMAGE
      type: string

    - name: COSIGN_TLOG_UPLOAD
      type: string
      default: "false"
      description: Upload the signature/attestation to the Sigstore transparency log.

  results:
    - name: IMAGE_DIGEST
      description: Digest of the image that was signed
      type: string

  workspaces:
    - name: dockerconfig
    - name: cosign
    - name: source

  steps:
    - name: image-digest
      image: alpine:3.20
      script: |
        #!/bin/sh
        set -eu

        mkdir -p /tekton/home/.docker

        if [ -f "$(workspaces.dockerconfig.path)/.dockerconfigjson" ]; then
          cp "$(workspaces.dockerconfig.path)/.dockerconfigjson" /tekton/home/.docker/config.json
        elif [ -f "$(workspaces.dockerconfig.path)/config.json" ]; then
          cp "$(workspaces.dockerconfig.path)/config.json" /tekton/home/.docker/config.json
        else
          echo "ERROR: Harbor registry credentials were not mounted."
          exit 1
        fi

        DIGEST_FILE="$(workspaces.source.path)/image-digest"

        if [ ! -f "${DIGEST_FILE}" ]; then
          echo "ERROR: Image digest file not found: ${DIGEST_FILE}"
          exit 1
        fi

        IMAGE_DIGEST="$(cat "${DIGEST_FILE}" | tr -d '[:space:]')"

        case "${IMAGE_DIGEST}" in
          sha256:*) ;;
          *)
            echo "ERROR: Invalid image digest: ${IMAGE_DIGEST}"
            exit 1
            ;;
        esac

        printf '%s' "${IMAGE_DIGEST}" > "$(results.IMAGE_DIGEST.path)"
        printf '%s' "${IMAGE_DIGEST}" > "${DIGEST_FILE}"

        echo "Immutable image reference:"
        echo "$(params.IMAGE)@${IMAGE_DIGEST}"

    - name: sbom
      image: anchore/syft:v1.54.0
      env:
        - name: DOCKER_CONFIG
          value: /tekton/home/.docker
        - name: SSL_CERT_FILE
          value: /etc/harbor-ca/ca.crt
      volumeMounts:
        - name: harbor-ca
          mountPath: /etc/harbor-ca
          readOnly: true
      script: |
        #!/bin/sh
        set -eu

        IMAGE_DIGEST="$(cat "$(workspaces.source.path)/image-digest" | tr -d '[:space:]')"
        IMAGE_REF="$(params.IMAGE)@${IMAGE_DIGEST}"

        /syft scan "${IMAGE_REF}" \
          -o spdx-json="$(workspaces.source.path)/sbom.spdx.json"

        test -s "$(workspaces.source.path)/sbom.spdx.json"

    - name: sign
      image: ghcr.io/sigstore/cosign/cosign:v3.0.2
      env:
        - name: DOCKER_CONFIG
          value: /tekton/home/.docker
        - name: SSL_CERT_FILE
          value: /etc/harbor-ca/ca.crt
        - name: COSIGN_PASSWORD
          valueFrom:
            secretKeyRef:
              name: cosign-password
              key: password
      volumeMounts:
        - name: harbor-ca
          mountPath: /etc/harbor-ca
          readOnly: true
      script: |
        #!/bin/sh
        set -eu

        IMAGE_DIGEST="$(cat "$(workspaces.source.path)/image-digest" | tr -d '[:space:]')"
        IMAGE_REF="$(params.IMAGE)@${IMAGE_DIGEST}"

        /ko-app/cosign sign --yes \
          --tlog-upload="$(params.COSIGN_TLOG_UPLOAD)" \
          --key "$(workspaces.cosign.path)/cosign.key" \
          "${IMAGE_REF}"

    - name: attest-sbom
      image: ghcr.io/sigstore/cosign/cosign:v3.0.2
      env:
        - name: DOCKER_CONFIG
          value: /tekton/home/.docker
        - name: SSL_CERT_FILE
          value: /etc/harbor-ca/ca.crt
        - name: COSIGN_PASSWORD
          valueFrom:
            secretKeyRef:
              name: cosign-password
              key: password
      volumeMounts:
        - name: harbor-ca
          mountPath: /etc/harbor-ca
          readOnly: true
      script: |
        #!/bin/sh
        set -eu

        IMAGE_DIGEST="$(cat "$(workspaces.source.path)/image-digest" | tr -d '[:space:]')"
        IMAGE_REF="$(params.IMAGE)@${IMAGE_DIGEST}"

        /ko-app/cosign attest --yes \
          --tlog-upload="$(params.COSIGN_TLOG_UPLOAD)" \
          --key "$(workspaces.cosign.path)/cosign.key" \
          --type spdxjson \
          --predicate "$(workspaces.source.path)/sbom.spdx.json" \
          "${IMAGE_REF}"

```

Apply:

```bash
kubectl apply -f sign-image-task.yaml
kubectl get task sign-image -n cicd
kubectl describe task sign-image -n cicd
```

---

# 14. GitOps `update-values` Task

The source repository and GitOps repository are treated as separate repositories.

The Pipeline therefore uses two parameters:

- `REPO_URL` -> application source repository.
- `MANIFEST_REPO_URL` -> GitOps/Helm manifest repository.

This avoids accidentally cloning the application repository when the final Task is supposed to update `values.yaml` in the GitOps repository.

Create `update-values.yaml`:

```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: update-values
  namespace: cicd
spec:
  description: >-
    Clone the GitOps manifest repository, update Helm values.yaml with the newly
    built Harbor image digest, commit the change, and push it back to the repository.

  params:
    - name: REPO_URL
      type: string
      description: GitOps manifest repository URL
      default: http://10.12.90.62/admin/manifest-helm.git

    - name: IMAGE
      type: string
      description: Full container image including tag
      default: lab25-harbor.lab25.sunfire.lab/cicd/travelportal:latest

    - name: IMAGE_DIGEST
      type: string
      description: SHA256 digest of the pushed image

    - name: VALUES_FILE
      type: string
      description: Helm values file relative to repository root
      default: helm-charts/values.yaml

    - name: GIT_BRANCH
      type: string
      description: Git branch
      default: main

    - name: COMMIT_MESSAGE
      type: string
      description: Git commit message
      default: "ci: update TravelPortal image digest"

  workspaces:
    - name: source

  steps:
    - name: update-and-push
      image: alpine/git:latest
      env:
        - name: GIT_USERNAME
          valueFrom:
            secretKeyRef:
              name: repo-git-credentials
              key: username

        - name: GIT_TOKEN
          valueFrom:
            secretKeyRef:
              name: repo-git-credentials
              key: token

      script: |
        #!/bin/sh
        set -eu

        WORKDIR="$(workspaces.source.path)"
        REPO_URL="$(params.REPO_URL)"
        GIT_BRANCH="$(params.GIT_BRANCH)"
        VALUES_FILE="$(params.VALUES_FILE)"
        IMAGE="$(params.IMAGE)"
        IMAGE_DIGEST="$(params.IMAGE_DIGEST)"

        echo "========================================"
        echo "Update Helm values.yaml"
        echo "========================================"
        echo "Repository: ${REPO_URL}"
        echo "Branch:     ${GIT_BRANCH}"
        echo "Image:      ${IMAGE}"
        echo "Digest:     ${IMAGE_DIGEST}"
        echo "Values:     ${VALUES_FILE}"

        rm -rf "${WORKDIR:?}"/*
        rm -rf "${WORKDIR}"/.[!.]*
        rm -rf "${WORKDIR}"/..?*

        git config --global user.name "Tekton CI"
        git config --global user.email "tekton-ci@local"
        git config --global init.defaultBranch main

        # Do not embed the token in the repository URL.
        # Use Git's credential helper protocol instead.
        git config --global credential.helper '!f() { if [ "$1" = get ]; then printf "protocol=http\\nusername=%s\\npassword=%s\\n\\n" "$GIT_USERNAME" "$GIT_TOKEN"; fi; }; f'

        echo "Cloning GitOps repository..."
        git clone \
          --branch "${GIT_BRANCH}" \
          "${REPO_URL}" \
          "${WORKDIR}"

        git config --global --add safe.directory "${WORKDIR}"
        cd "${WORKDIR}"

        git status
        git branch --show-current
        git log --oneline --decorate -5

        git fetch origin "${GIT_BRANCH}"
        git checkout "${GIT_BRANCH}"
        git reset --hard "origin/${GIT_BRANCH}"

        VALUES_FILE_PATH="${WORKDIR}/${VALUES_FILE}"

        if [ ! -f "${VALUES_FILE_PATH}" ]; then
          echo "ERROR: values.yaml not found: ${VALUES_FILE_PATH}"
          find "${WORKDIR}" -type f -name "values.yaml" -print
          exit 1
        fi

        echo "Current values.yaml:"
        cat "${VALUES_FILE_PATH}"

        IMAGE_REPOSITORY="${IMAGE%:*}"
        IMAGE_TAG="${IMAGE##*:}"

        echo "Image repository: ${IMAGE_REPOSITORY}"
        echo "Image tag:        ${IMAGE_TAG}"
        echo "Image digest:     ${IMAGE_DIGEST}"

        case "${IMAGE_DIGEST}" in
          sha256:*)
            echo "Valid SHA256 digest."
            ;;
          *)
            echo "ERROR: IMAGE_DIGEST does not start with sha256:"
            echo "${IMAGE_DIGEST}"
            exit 1
            ;;
        esac

        TMP_VALUES_FILE="${VALUES_FILE_PATH}.tmp"

        awk \
          -v repo="${IMAGE_REPOSITORY}" \
          -v tag="${IMAGE_TAG}" \
          -v digest="${IMAGE_DIGEST}" \
          '
          BEGIN { in_image=0; found_repo=0; found_tag=0; found_digest=0 }
          /^image:[[:space:]]*$/ { in_image=1; print; next }
          in_image && /^[^[:space:]]/ {
            if (!found_repo) { print "  repository: " repo; found_repo=1 }
            if (!found_tag) { print "  tag: \"" tag "\""; found_tag=1 }
            if (!found_digest) { print "  digest: \"" digest "\""; found_digest=1 }
            in_image=0
          }
          in_image && /^  repository:/ { print "  repository: " repo; found_repo=1; next }
          in_image && /^  tag:/ { print "  tag: \"" tag "\""; found_tag=1; next }
          in_image && /^  digest:/ { print "  digest: \"" digest "\""; found_digest=1; next }
          { print }
          END {
            if (in_image) {
              if (!found_repo) { print "  repository: " repo; found_repo=1 }
              if (!found_tag) { print "  tag: \"" tag "\""; found_tag=1 }
              if (!found_digest) { print "  digest: \"" digest "\""; found_digest=1 }
            }
            if (!found_repo || !found_tag || !found_digest) exit 1
          }
          ''' \
          "${VALUES_FILE_PATH}" > "${TMP_VALUES_FILE}"

        mv "${TMP_VALUES_FILE}" "${VALUES_FILE_PATH}"

        echo "Updated values.yaml:"
        cat "${VALUES_FILE_PATH}"

        echo "Git diff:"
        git diff -- "${VALUES_FILE}"

        if git diff --quiet -- "${VALUES_FILE}"; then
          echo "No changes detected in values.yaml."
          exit 0
        fi

        git add "${VALUES_FILE}"

        git commit \
          -m "$(params.COMMIT_MESSAGE): ${IMAGE}"

        PUSH_SUCCESS="false"

        for ATTEMPT in 1 2 3; do
          echo "Push attempt ${ATTEMPT}/3"

          if git push origin "${GIT_BRANCH}"; then
            PUSH_SUCCESS="true"
            echo "Git push successful."
            break
          fi

          echo "Push rejected. Fetching latest remote branch..."
          git fetch origin "${GIT_BRANCH}"
          if ! git rebase "origin/${GIT_BRANCH}"; then
            git rebase --abort || true
            echo "ERROR: Git rebase failed after a concurrent remote update."
            exit 1
          fi
        done

        if [ "${PUSH_SUCCESS}" != "true" ]; then
          echo "ERROR: Git push failed after 3 attempts."
          exit 1
        fi

        echo "GitOps values.yaml updated successfully."
        git log -1 --oneline
```

Apply and verify:

```bash
kubectl apply -f update-values.yaml
kubectl get task update-values -n cicd
kubectl describe task update-values -n cicd
```

---

# 15. Pipeline

The Pipeline uses two repository parameters:

```text
REPO_URL             -> Application source repository
MANIFEST_REPO_URL    -> GitOps manifest repository
```

Create `travelportal-pipeline-values-update.yaml`:

```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: travelportal-pipeline-values-update
  namespace: cicd
spec:
  params:
    - name: REPO_URL
      type: string
      default: http://10.12.90.62/admin/travelPortal-test-buildpack.git

    - name: MANIFEST_REPO_URL
      type: string
      default: http://10.12.90.62/admin/manifest-helm.git

    - name: REVISION
      type: string
      default: main

    - name: IMAGE
      type: string
      default: lab25-harbor.lab25.sunfire.lab/cicd/travelportal:latest

    - name: BUILDER_IMAGE
      type: string
      default: paketobuildpacks/builder-jammy-base

  workspaces:
    - name: source
    - name: dockerconfig
    - name: cosign-key
    - name: cache
    - name: git-credentials

  tasks:
    - name: clone
      taskRef:
        name: git-clone-update
      params:
        - name: url
          value: "$(params.REPO_URL)"
        - name: revision
          value: "$(params.REVISION)"
        - name: deleteExisting
          value: "true"
      workspaces:
        - name: output
          workspace: source
        - name: git-credentials
          workspace: git-credentials

    - name: detect
      runAfter:
        - clone
      taskRef:
        name: detect-build-type
      workspaces:
        - name: source
          workspace: source

    - name: buildkit
      runAfter:
        - detect
      when:
        - input: "$(tasks.detect.results.BUILD_TYPE)"
          operator: in
          values:
            - buildkit
      taskRef:
        name: buildkit-build
      params:
        - name: IMAGE
          value: "$(params.IMAGE)"
        - name: DOCKERFILE
          value: Dockerfile
        - name: CONTEXT
          value: "."
      workspaces:
        - name: source
          workspace: source
        - name: dockerconfig
          workspace: dockerconfig

    - name: buildpack
      runAfter:
        - detect
      when:
        - input: "$(tasks.detect.results.BUILD_TYPE)"
          operator: in
          values:
            - buildpack
      taskRef:
        name: buildpacks-phases
      params:
        - name: CNB_BUILDER_IMAGE
          value: "$(params.BUILDER_IMAGE)"
        - name: APP_IMAGE
          value: "$(params.IMAGE)"
        - name: SOURCE_SUBPATH
          value: ""
      workspaces:
        - name: source
          workspace: source
        - name: dockerconfig
          workspace: dockerconfig
        - name: cache
          workspace: cache

    - name: sign
      runAfter:
        - buildkit
        - buildpack
      taskRef:
        name: sign-image
      params:
        - name: IMAGE
          value: "$(params.IMAGE)"
        - name: COSIGN_TLOG_UPLOAD
          value: "false"
      workspaces:
        - name: dockerconfig
          workspace: dockerconfig
        - name: cosign
          workspace: cosign-key
        - name: source
          workspace: source

    - name: update-values
      runAfter:
        - sign
      taskRef:
        name: update-values
      params:
        - name: REPO_URL
          value: "$(params.MANIFEST_REPO_URL)"
        - name: IMAGE
          value: "$(params.IMAGE)"
        - name: IMAGE_DIGEST
          value: "$(tasks.sign.results.IMAGE_DIGEST)"
        - name: VALUES_FILE
          value: helm-charts/values.yaml
        - name: GIT_BRANCH
          value: "$(params.REVISION)"
        - name: COMMIT_MESSAGE
          value: "ci: update TravelPortal image digest"
      workspaces:
        - name: source
          workspace: source
```

Apply:

```bash
kubectl apply -f travelportal-pipeline-values-update.yaml
kubectl get pipeline travelportal-pipeline-values-update -n cicd
```

> The pipeline keeps the original conditional BuildKit/Buildpacks -> Sign dependency unchanged by request. This is the one known pipeline-graph item to fix later.

---

# 16. Pipeline Dependencies

The PipelineRun needs the following resources:

```text
Namespace
   |
   +-- StorageClass: lab-gold-storage-policy
   |
   +-- ConfigMap: harbor-ca-cert
   |
   +-- Secret: harbor-registry-secret
   +-- Secret: repo-git-credentials
   +-- Secret: cosign-password
   +-- Secret: cosign-key
   |
   +-- ServiceAccount: tekton-pipeline-sa
   |
   +-- Role / RoleBinding
```

---

## 16.1 Verify StorageClass

```bash
kubectl get storageclass lab-gold-storage-policy
```

If your VKS environment uses another StorageClass name, replace it in the PipelineRun and TriggerTemplate.

---

## 16.2 Harbor CA ConfigMap

Create `harbor-ca-cert.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: harbor-ca-cert
  namespace: cicd
data:
  ca.crt: |
    -----BEGIN CERTIFICATE-----
    <PASTE-YOUR-HARBOR-OR-VCENTER-CA-CERTIFICATE-HERE>
    -----END CERTIFICATE-----
```

Apply:

```bash
kubectl apply -f harbor-ca-cert.yaml
kubectl get configmap harbor-ca-cert -n cicd
```

The certificate must contain the actual CA chain needed to validate:

```text
lab25-harbor.lab25.sunfire.lab
```

---

## 16.3 Create repository credentials

The Git source repository and GitOps manifest repository use the same internal Git service in this lab.

```bash
kubectl create secret generic repo-git-credentials \\
  -n cicd \\
  --from-literal=username=admin \\
  --from-literal=token='<REPO_PERSONAL_ACCESS_TOKEN>' \\
  --dry-run=client -o yaml | kubectl apply -f -
```

Verify:

```bash
kubectl get secret repo-git-credentials -n cicd
```

---

## 16.4 Create Harbor registry Secret

Recommended command:

```bash
kubectl create secret docker-registry harbor-registry-secret \\
  -n cicd \\
  --docker-server=lab25-harbor.lab25.sunfire.lab \\
  --docker-username=admin \\
  --docker-password='<HARBOR_PASSWORD>' \\
  --dry-run=client -o yaml | kubectl apply -f -
```

Verify:

```bash
kubectl get secret harbor-registry-secret -n cicd
```

This creates the Docker configuration used by BuildKit, Buildpacks, Syft, and Cosign.

---

## 16.5 Create Cosign password Secret

```bash
kubectl create secret generic cosign-password \\
  -n cicd \\
  --from-literal=password='<COSIGN_KEY_PASSWORD>' \\
  --dry-run=client -o yaml | kubectl apply -f -
```

---

## 16.6 Create Cosign key Secret

Create `cosign-key-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cosign-key
  namespace: cicd
type: Opaque
stringData:
  cosign.key: |
    -----BEGIN ENCRYPTED COSIGN PRIVATE KEY-----
    <PASTE-COSIGN-PRIVATE-KEY>
    -----END ENCRYPTED COSIGN PRIVATE KEY-----
  cosign.pub: |
    -----BEGIN COSIGN PUBLIC KEY-----
    <PASTE-COSIGN-PUBLIC-KEY>
    -----END COSIGN PUBLIC KEY-----
```

Apply:

```bash
kubectl apply -f cosign-key-secret.yaml
kubectl get secret cosign-key -n cicd
```

---

# 17. Pipeline ServiceAccount and RBAC

Create `tekton-pipeline-sa.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tekton-pipeline-sa
  namespace: cicd
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tekton-pipeline-role
  namespace: cicd
rules:
  - apiGroups: [""]
    resources:
      - secrets
      - configmaps
    verbs:
      - get
      - list
      - watch

  - apiGroups: [""]
    resources:
      - persistentvolumeclaims
    verbs:
      - get
      - list
      - watch
      - update
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tekton-pipeline-rolebinding
  namespace: cicd
subjects:
  - kind: ServiceAccount
    name: tekton-pipeline-sa
    namespace: cicd
roleRef:
  kind: Role
  name: tekton-pipeline-role
  apiGroup: rbac.authorization.k8s.io
```

Apply:

```bash
kubectl apply -f tekton-pipeline-sa.yaml
```

Verify:

```bash
kubectl get sa tekton-pipeline-sa -n cicd
kubectl get role tekton-pipeline-role -n cicd
kubectl get rolebinding tekton-pipeline-rolebinding -n cicd
```

---

# 18. Manual PipelineRun

The PipelineRun below creates a new source PVC and cache PVC for every PipelineRun. This avoids multiple concurrent runs overwriting the same source workspace.

Create `travelportal-pipelinerun-values-update.yaml`:

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  generateName: travelportal-build-
  namespace: cicd
spec:
  pipelineRef:
    name: travelportal-pipeline-values-update

  params:
    - name: REPO_URL
      value: http://10.12.90.62/admin/travelPortal-test-buildpack.git

    - name: MANIFEST_REPO_URL
      value: http://10.12.90.62/admin/manifest-helm.git

    - name: REVISION
      value: main

    - name: IMAGE
      value: lab25-harbor.lab25.sunfire.lab/cicd/travelportal:latest

    - name: BUILDER_IMAGE
      value: paketobuildpacks/builder-jammy-base

  taskRunTemplate:
    serviceAccountName: tekton-pipeline-sa

  workspaces:
    - name: source
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          storageClassName: lab-gold-storage-policy
          resources:
            requests:
              storage: 5Gi

    - name: dockerconfig
      secret:
        secretName: harbor-registry-secret

    - name: cosign-key
      secret:
        secretName: cosign-key

    - name: cache
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          storageClassName: lab-gold-storage-policy
          resources:
            requests:
              storage: 5Gi

    - name: git-credentials
      secret:
        secretName: repo-git-credentials

  taskRunSpecs:
    - pipelineTaskName: buildkit
      podTemplate:
        volumes:
          - name: harbor-ca
            configMap:
              name: harbor-ca-cert

    - pipelineTaskName: sign
      podTemplate:
        volumes:
          - name: harbor-ca
            configMap:
              name: harbor-ca-cert
```

Apply:

```bash
kubectl apply -f travelportal-pipelinerun-values-update.yaml
```

Verify:

```bash
kubectl get pipelinerun -n cicd
kubectl get taskrun -n cicd
kubectl get pvc -n cicd
```

Watch:

```bash
kubectl get pipelineruns -n cicd -w
```

Inspect a failed run:

```bash
kubectl describe pipelinerun <PIPELINERUN_NAME> -n cicd
kubectl get taskrun -n cicd
kubectl describe taskrun <TASKRUN_NAME> -n cicd
```

---

# 19. Validate the Built Image in Harbor

From a system that can reach Harbor:

```bash
docker login lab25-harbor.lab25.sunfire.lab
```

Pull the image:

```bash
docker pull \\
  lab25-harbor.lab25.sunfire.lab/cicd/travelportal:latest
```

List the image:

```bash
docker images
```

The digest is the immutable identifier used for Cosign verification and GitOps deployment.

---

# 20. Verify the Cosign Signature

Set the image and digest:

```bash
IMAGE=lab25-harbor.lab25.sunfire.lab/cicd/travelportal
IMAGE_DIGEST=sha256:<ACTUAL_IMAGE_DIGEST>
```

Verify offline with the public key:

```bash
cosign verify \\
  --offline \\
  --key cosign.pub \\
  "${IMAGE}@${IMAGE_DIGEST}"
```

---

# 21. Verify the SBOM Attestation

```bash
cosign verify-attestation \\
  --offline \\
  --key cosign.pub \\
  --type spdxjson \\
  "${IMAGE}@${IMAGE_DIGEST}"
```

This verifies that the SPDX SBOM attestation is associated with the specified image digest.

---

# 22. Install Tekton Triggers

Tekton Triggers is used to start PipelineRuns automatically when the repository sends a webhook.

For this documented compatibility set, install Triggers v0.36.0 including the interceptor manifests.

```bash
kubectl apply -f \\
  https://storage.googleapis.com/tekton-releases/triggers/previous/v0.36.0/release.yaml

kubectl apply -f \\
  https://storage.googleapis.com/tekton-releases/triggers/previous/v0.36.0/interceptors.yaml
```

Watch the Pods:

```bash
kubectl get pods -n tekton-pipelines -w
```

Expected Triggers components include:

```text
tekton-triggers-controller
tekton-triggers-webhook
tekton-triggers-core-interceptors
```

Verify CRDs:

```bash
kubectl get crd | grep triggers.tekton.dev
```

Important resources include:

```text
clusterinterceptors.triggers.tekton.dev
clustertriggerbindings.triggers.tekton.dev
eventlisteners.triggers.tekton.dev
interceptors.triggers.tekton.dev
triggerbindings.triggers.tekton.dev
triggers.triggers.tekton.dev
triggertemplates.triggers.tekton.dev
```

Verify the CEL interceptor:

```bash
kubectl get clusterinterceptor cel -o yaml
```

---

# 23. Tekton Triggers RBAC

The EventListener uses:

```text
repository-trigger-sa
```

Create `webhook-sa.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: repository-trigger-sa
  namespace: cicd
```

Apply:

```bash
kubectl apply -f webhook-sa.yaml
```

Create `repository-trigger-rbac.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: repository-trigger-role
  namespace: cicd
rules:
  - apiGroups: ["triggers.tekton.dev"]
    resources:
      - eventlisteners
      - triggerbindings
      - triggertemplates
      - triggers
      - interceptors
    verbs:
      - get
      - list
      - watch

  - apiGroups: ["tekton.dev"]
    resources:
      - pipelineruns
    verbs:
      - create
      - get
      - list
      - watch
```

Apply:

```bash
kubectl apply -f repository-trigger-rbac.yaml
```

Create `repository-trigger-rolebinding.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: repository-trigger-rolebinding
  namespace: cicd
subjects:
  - kind: ServiceAccount
    name: repository-trigger-sa
    namespace: cicd
roleRef:
  kind: Role
  name: repository-trigger-role
  apiGroup: rbac.authorization.k8s.io
```

Apply:

```bash
kubectl apply -f repository-trigger-rolebinding.yaml
```

Create `repository-trigger-cluster-role.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: repository-trigger-cluster-role
rules:
  - apiGroups: ["triggers.tekton.dev"]
    resources:
      - clusterinterceptors
      - clustertriggerbindings
    verbs:
      - get
      - list
      - watch
```

Apply:

```bash
kubectl apply -f repository-trigger-cluster-role.yaml
```

Create `repository-trigger-cluster-rolebinding.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: repository-trigger-cluster-rolebinding
subjects:
  - kind: ServiceAccount
    name: repository-trigger-sa
    namespace: cicd
roleRef:
  kind: ClusterRole
  name: repository-trigger-cluster-role
  apiGroup: rbac.authorization.k8s.io
```

Apply:

```bash
kubectl apply -f repository-trigger-cluster-rolebinding.yaml
```

Verify:

```bash
kubectl get sa repository-trigger-sa -n cicd
kubectl get role repository-trigger-role -n cicd
kubectl get rolebinding repository-trigger-rolebinding -n cicd
kubectl get clusterrole repository-trigger-cluster-role
kubectl get clusterrolebinding repository-trigger-cluster-rolebinding
```

Verify permissions:

```bash
kubectl auth can-i list triggerbindings.triggers.tekton.dev \\
  --as=system:serviceaccount:cicd:repository-trigger-sa -n cicd

kubectl auth can-i list triggertemplates.triggers.tekton.dev \\
  --as=system:serviceaccount:cicd:repository-trigger-sa -n cicd

kubectl auth can-i list eventlisteners.triggers.tekton.dev \\
  --as=system:serviceaccount:cicd:repository-trigger-sa -n cicd

kubectl auth can-i list clusterinterceptors.triggers.tekton.dev \\
  --as=system:serviceaccount:cicd:repository-trigger-sa

kubectl auth can-i list clustertriggerbindings.triggers.tekton.dev \\
  --as=system:serviceaccount:cicd:repository-trigger-sa

kubectl auth can-i create pipelineruns.tekton.dev \\
  --as=system:serviceaccount:cicd:repository-trigger-sa -n cicd
```

Expected:

```text
yes
```

for each command.

---

# 24. TriggerBinding

The TriggerBinding supplies the application repository and GitOps repository used by the Pipeline.

Create `repository-trigger-binding.yaml`:

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
  name: travelportal-repository-binding
  namespace: cicd
spec:
  params:
    - name: REPO_URL
      value: http://10.12.90.62/admin/travelPortal-test-buildpack.git

    - name: MANIFEST_REPO_URL
      value: http://10.12.90.62/admin/manifest-helm.git

    - name: REVISION
      value: main
```

Apply:

```bash
kubectl apply -f repository-trigger-binding.yaml
kubectl get triggerbinding travelportal-repository-binding -n cicd -o yaml
```

---

# 25. TriggerTemplate

The TriggerTemplate creates a new PipelineRun automatically.

Create `repository-trigger-template.yaml`:

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: travelportal-repository-template
  namespace: cicd
spec:
  params:
    - name: REPO_URL
    - name: MANIFEST_REPO_URL
    - name: REVISION
      default: main

  resourcetemplates:
    - apiVersion: tekton.dev/v1
      kind: PipelineRun
      metadata:
        generateName: travelportal-build-
      spec:
        pipelineRef:
          name: travelportal-pipeline-values-update

        params:
          - name: REPO_URL
            value: $(tt.params.REPO_URL)
          - name: MANIFEST_REPO_URL
            value: $(tt.params.MANIFEST_REPO_URL)
          - name: REVISION
            value: $(tt.params.REVISION)
          - name: IMAGE
            value: lab25-harbor.lab25.sunfire.lab/cicd/travelportal:latest
          - name: BUILDER_IMAGE
            value: paketobuildpacks/builder-jammy-base

        taskRunTemplate:
          serviceAccountName: tekton-pipeline-sa

        taskRunSpecs:
          - pipelineTaskName: buildkit
            podTemplate:
              volumes:
                - name: harbor-ca
                  configMap:
                    name: harbor-ca-cert

          - pipelineTaskName: sign
            podTemplate:
              volumes:
                - name: harbor-ca
                  configMap:
                    name: harbor-ca-cert

        timeouts:
          pipeline: 1h0m0s

        workspaces:
          - name: source
            volumeClaimTemplate:
              spec:
                accessModes:
                  - ReadWriteOnce
                storageClassName: lab-gold-storage-policy
                resources:
                  requests:
                    storage: 5Gi

          - name: dockerconfig
            secret:
              secretName: harbor-registry-secret

          - name: cosign-key
            secret:
              secretName: cosign-key

          - name: cache
            volumeClaimTemplate:
              spec:
                accessModes:
                  - ReadWriteOnce
                storageClassName: lab-gold-storage-policy
                resources:
                  requests:
                    storage: 5Gi

          - name: git-credentials
            secret:
              secretName: repo-git-credentials
```

Apply:

```bash
kubectl apply -f repository-trigger-template.yaml
kubectl get triggertemplate travelportal-repository-template -n cicd -o yaml
```

---

# 26. EventListener

The EventListener receives the repository webhook.

Create `repository-eventlistener.yaml`:

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: travelportal-repository-listener
  namespace: cicd
spec:
  serviceAccountName: repository-trigger-sa

  resources:
    kubernetesResource:
      serviceType: NodePort

  triggers:
    - name: travelportal-push

      interceptors:
        - ref:
            apiVersion: triggers.tekton.dev
            kind: ClusterInterceptor
            name: cel
          params:
            - name: filter
              value: >-
                body.ref == 'refs/heads/main' &&
                !body.head_commit.message.startsWith('ci: update TravelPortal image digest')

      bindings:
        - ref: travelportal-repository-binding

      template:
        ref: travelportal-repository-template
```

Apply:

```bash
kubectl apply -f repository-eventlistener.yaml
```

Verify:

```bash
kubectl get eventlistener travelportal-repository-listener -n cicd
```

Expected:

```text
READY=True
AVAILABLE=True
```

Verify the CEL interceptor:

```bash
kubectl get clusterinterceptor cel -o yaml
```

---

# 27. Verify the EventListener Service

```bash
kubectl get svc el-travelportal-repository-listener -n cicd
```

Example:

```text
NAME                                  TYPE       CLUSTER-IP    PORT(S)
el-travelportal-repository-listener  NodePort   10.x.x.x      8080:31877/TCP
```

The actual NodePort is cluster-specific. Do not hard-code `31877` unless it is the current value shown by the Service.

Get the endpoints:

```bash
kubectl get endpoints el-travelportal-repository-listener -n cicd -o wide
```

Get the EventListener Pods:

```bash
kubectl get pods -n cicd \\
  -l eventlistener=travelportal-repository-listener \\
  -o wide
```

---

# 28. Test the EventListener Before Configuring the Webhook

Assume the observed lab values are:

```text
Node:        10.12.92.3
NodePort:    31877
Webhook URL: http://10.12.92.3:31877
```

Verify the actual NodePort first:

```bash
kubectl get svc el-travelportal-repository-listener -n cicd
```

Then test:

```bash
curl -v http://10.12.92.3:31877
```

Create a webhook payload:

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
```

Send it:

```bash
curl -v \\
  -H 'Content-Type: application/json' \\
  --data-binary @/tmp/test-event.json \\
  http://10.12.92.3:31877
```

A successful EventListener path normally returns `HTTP 202 Accepted` with an event ID.

---

# 29. Configure the Repository Webhook

On the repository server:

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
| Method | POST |
| Content type | application/json |
| Event | Push events |
| Branch | main |

> The exact webhook UI depends on the Git server. The important requirement is that the Git server can reach the EventListener NodePort over the internal network.

If the Git server enforces an outbound HTTP allow-list, ensure the EventListener host and port are allowed. Otherwise the server may reject the webhook before it leaves the Git service.

---

# 30. Automatic Pipeline Execution

Once the webhook is configured, the developer no longer needs to create a PipelineRun manually.

Developer action:

```bash
git add .
git commit -m "developer change"
git push origin main
```

Watch the EventListener:

```bash
kubectl logs -f \\
  deployment/el-travelportal-repository-listener \\
  -n cicd
```

Watch PipelineRuns:

```bash
kubectl get pipelineruns -n cicd -w
```

You should see:

```text
travelportal-build-xxxxx
```

---

# 31. CI Self-Trigger Protection

The GitOps update Task commits:

```text
ci: update TravelPortal image digest: <image>
```

The EventListener CEL filter checks:

```text
body.ref == 'refs/heads/main'
```

and ignores commits whose message begins with:

```text
ci: update TravelPortal image digest
```

Therefore:

```text
Developer commit
    |
    v
Pipeline starts
    |
    v
update-values commits values.yaml
    |
    v
Repository sends webhook again
    |
    v
CEL sees CI commit prefix
    |
    v
Event rejected
    |
    v
No second PipelineRun
```

---

# 32. Troubleshooting Commands

## PipelineRun failed

```bash
kubectl get pipelineruns -n cicd
kubectl describe pipelinerun <RUN> -n cicd
kubectl get taskruns -n cicd
kubectl describe taskrun <TASKRUN> -n cicd
```

## BuildKit failed

```bash
kubectl logs <BUILDKIT-POD> -n cicd -c step-build
```

Check:

```text
Harbor DNS
Harbor CA
Docker credentials
Dockerfile
BuildKit privileged Pod policy
```

## Buildpacks failed

Check:

```text
Builder image pull
Builder image platform API
Harbor credentials
Cache PVC
Source workspace
```

## Syft failed

Check:

```text
Harbor CA
DOCKER_CONFIG
Harbor credentials
Image digest
```

## Cosign failed

Check:

```text
cosign.key
cosign-password
Harbor CA
Harbor credentials
COSIGN_TLOG_UPLOAD
```

For the air-gapped/private-registry mode, keep:

```text
COSIGN_TLOG_UPLOAD=false
```

## GitOps update failed

Check:

```bash
kubectl logs <UPDATE_VALUES_POD> -n cicd -c step-update-and-push
```

Verify:

```text
repo-git-credentials
MANIFEST_REPO_URL
helm-charts/values.yaml
main branch
repository write permissions
```

## EventListener not ready

```bash
kubectl describe eventlistener travelportal-repository-listener -n cicd
kubectl get pods -n cicd -l eventlistener=travelportal-repository-listener -o wide
kubectl logs deployment/el-travelportal-repository-listener -n cicd
```

## Webhook reaches Git server but no PipelineRun appears

Check:

```text
EventListener Service
NodePort
network routing/firewall
CEL filter
TriggerBinding
TriggerTemplate
repository payload format
```

---

# 33. Final Validation Checklist

Run:

```bash
kubectl get nodes
kubectl get namespace tekton-pipelines
kubectl get namespace cicd

kubectl get pods -n tekton-pipelines
kubectl get deployment -n tekton-pipelines
kubectl get crd | grep tekton

kubectl get task -n cicd
kubectl get pipeline -n cicd
kubectl get pipelineruns -n cicd
kubectl get taskruns -n cicd
kubectl get pvc -n cicd

kubectl get secret repo-git-credentials -n cicd
kubectl get secret harbor-registry-secret -n cicd
kubectl get secret cosign-password -n cicd
kubectl get secret cosign-key -n cicd
kubectl get configmap harbor-ca-cert -n cicd

kubectl get crd | grep triggers.tekton.dev
kubectl get pods -n tekton-pipelines | grep triggers
kubectl get sa repository-trigger-sa -n cicd
kubectl get eventlistener travelportal-repository-listener -n cicd
kubectl get svc el-travelportal-repository-listener -n cicd
kubectl get endpoints el-travelportal-repository-listener -n cicd
kubectl get triggerbinding travelportal-repository-binding -n cicd
kubectl get triggertemplate travelportal-repository-template -n cicd
```

Expected high-level state:

| Component | Expected state |
|---|---|
| VKS nodes | Ready |
| `tekton-pipelines` namespace | Active |
| `cicd` namespace | Active |
| Tekton controller | Running |
| Tekton webhook | Running |
| Tekton Dashboard | Running |
| Triggers controller/webhook/interceptors | Running |
| Tasks | Present |
| Pipeline | Present |
| PipelineRun | Succeeded for a working build path |
| Harbor image | Present |
| Image digest | `sha256:...` |
| Cosign signature | Verified with `cosign.pub` |
| SBOM attestation | Verified as SPDX JSON |
| EventListener | Ready/Available |
| Webhook | HTTP 202 from direct test |
| GitOps values.yaml | Updated with image digest |
| CI self-trigger | Rejected by CEL filter |

---

# 34. Disconnected / Air-Gapped Deployment Notes

The architecture can be used in disconnected environments, but Internet access must not be assumed at runtime.

Before disconnecting the environment, mirror all required artifacts into an approved internal registry or repository, including at minimum:

```text
Tekton Pipelines manifests
Tekton Dashboard manifests
Tekton Triggers manifests + interceptors
BuildKit image
Buildpacks builder image
Buildpacks run image if required
Syft image
Cosign image
Alpine / Git helper images
Skopeo/JQ image
UBI lifecycle helper images
Required CA certificates
Git repository access
Harbor registry access
```

For production/offline use:

- Pin image versions.
- Prefer image digests instead of floating `latest` tags.
- Mirror all dependencies before disconnecting the environment.
- Store the trusted CA chain in the cluster.
- Keep Cosign transparency-log upload disabled unless an approved internal/accessible service is available.

---

# 35. Security Notes

The current lab implementation uses privileged execution for BuildKit and root/capabilities for some Buildpacks lifecycle operations. This is not equivalent to a completely rootless build architecture.

Recommended production controls include:

- Separate `cicd` namespace.
- Least-privilege ServiceAccounts and RBAC.
- Private Harbor registry.
- Trusted Harbor CA rather than TLS bypass.
- Image signing by digest.
- SBOM attestation.
- Version-pinned CI images.
- Restricted access to Tekton Dashboard.
- HTTPS for externally exposed webhook/dashboard endpoints where applicable.
- GitOps deployment based on immutable image digests.
- Network restrictions between Git service, Tekton, Harbor, and VKS workloads.

---

# 36. Complete File Order

Create and apply the files in this order:

```text
01. test-task.yaml
02. hello-taskrun.yaml
03. git-clone-update-task.yaml
04. detect-build-type.yaml
05. buildpacks-phases.yaml
06. buildkit-build-task.yaml
07. sign-image-task.yaml
08. update-values.yaml
09. travelportal-pipeline-values-update.yaml
10. harbor-ca-cert.yaml
11. cosign-key-secret.yaml
12. tekton-pipeline-sa.yaml
13. travelportal-pipelinerun-values-update.yaml
14. webhook-sa.yaml
15. repository-trigger-rbac.yaml
16. repository-trigger-rolebinding.yaml
17. repository-trigger-cluster-role.yaml
18. repository-trigger-cluster-rolebinding.yaml
19. repository-trigger-binding.yaml
20. repository-trigger-template.yaml
21. repository-eventlistener.yaml
```

Required Secrets are created before the PipelineRun:

```text
repo-git-credentials
harbor-registry-secret
cosign-password
cosign-key
```

Required configuration:

```text
harbor-ca-cert
```

---

# 37. End-to-End Final Flow

```text
1. Developer pushes to main
          |
          v
2. Repository sends webhook
          |
          v
3. Tekton EventListener receives it
          |
          v
4. CEL checks branch and CI commit prefix
          |
          v
5. TriggerBinding supplies source + manifest repository information
          |
          v
6. TriggerTemplate creates PipelineRun
          |
          v
7. git-clone-update clones application source
          |
          v
8. detect-build-type checks for Dockerfile
          |
          +-----------------------------+
          |                             |
          v                             v
9a. BuildKit                    9b. Buildpacks
    builds image                    builds image
          |                             |
          +-------------+---------------+
                        v
10. image-digest is available in shared workspace
                        |
                        v
11. sign-image
      |
      +--> Syft generates sbom.spdx.json
      +--> Cosign signs image digest
      +--> Cosign attaches SBOM attestation
                        |
                        v
12. update-values clones GitOps repository
                        |
                        v
13. values.yaml is updated with repository/tag/digest
                        |
                        v
14. Git commit + push
                        |
                        v
15. Repository sends webhook for CI commit
                        |
                        v
16. CEL rejects CI digest-update commit
                        |
                        v
17. ArgoCD detects GitOps change
                        |
                        v
18. ArgoCD deploys the immutable image digest to VKS
```

> Step 10 -> Step 11 uses the existing conditional branch structure intentionally kept unchanged in this document. That branch-convergence issue remains the one item to correct separately.

---

# 38. Conclusion

This implementation provides a Kubernetes-native CI workflow on VKS using Tekton, BuildKit, Buildpacks, Syft, Cosign, and Harbor.

The pipeline automatically selects BuildKit or Buildpacks based on the application source, generates an SBOM, signs the immutable image digest, and stores the image and OCI security artifacts in Harbor.

The application source repository and GitOps manifest repository are separated, providing a controlled workflow between application builds and deployment configuration.

The GitOps update records the immutable image digest in `values.yaml`, allowing ArgoCD to deploy a traceable image version.

The same architecture can be used in connected or disconnected VKS environments after the required images, manifests, CA certificates, credentials, and registry artifacts have been made available locally.

---

# 39. Official References

- Tekton Pipelines installation: https://tekton.dev/docs/installation/
- Tekton Pipelines v1.15.0 release: https://github.com/tektoncd/pipeline/releases/tag/v1.15.0
- Tekton Dashboard installation: https://tekton.dev/docs/dashboard/tutorial/
- Tekton Dashboard v0.71.0 release: https://github.com/tektoncd/dashboard/releases/tag/v0.71.0
- Tekton Triggers installation: https://tekton.dev/docs/installation/triggers/
- Tekton Triggers overview: https://tekton.dev/docs/triggers/
- Tekton Triggers v0.36.0 release: https://github.com/tektoncd/triggers/releases/tag/v0.36.0
- Cosign documentation: https://docs.sigstore.dev/cosign/system_config/installation/
- Syft releases: https://github.com/anchore/syft/releases
- Tekton Dashboard release compatibility information: https://github.com/tektoncd/dashboard/releases
