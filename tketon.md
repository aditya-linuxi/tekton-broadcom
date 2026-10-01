# Tektone installation and integration with buildpack, buildkit and cosign on vks

## Introduction

- **VKS (vSphere Kubernetes Service):** A Kubernetes service provided by VMware Cloud Foundation for creating and managing Kubernetes clusters on vSphere infrastructure.

- **Tekton:** A Kubernetes-native CI/CD framework used to create and run automated pipelines as Kubernetes resources.

- **Cloud Native Buildpacks (CNB):** A technology that converts application source code into container images without requiring the developer to write a Dockerfile.

- **BuildKit:** A modern container image build engine used to build images from Dockerfiles efficiently and supports advanced build features such as caching and parallel builds.

- **Cosign:** A tool used to digitally sign and verify container images and other OCI artifacts. It helps confirm that an image came from a trusted source and has not been tampered with.

- **SBOM (Software Bill of Materials):** A detailed list of the software components, libraries, and dependencies contained in a container image. It helps with vulnerability tracking and software supply-chain security.

- **Harbor:** A private container registry used to store and manage the container images produced by the Tekton pipeline.

- **Tekton Triggers:** A Tekton component that listens for events (such as a repository webhook) and automatically creates PipelineRuns. It is made of an EventListener, a TriggerBinding and a TriggerTemplate.

- **Webhook:** An HTTP call the repository sends to the Tekton EventListener every time someone pushes code.

## Why use BuildKit and Buildpacks with Tekton on VKS?

In plain terms, here's why this combination makes sense:

- **One automatic process for everyone.** Whether or not a team wrote a Dockerfile, their code still gets built the same reliable way — Tekton decides which tool to use, so nobody has to remember or configure it manually.

- **Nothing gets built by hand.** A person pushing code is the only manual step. Everything after that — building, checking for security issues, signing, publishing — happens on its own.

- **Trust is built in, not bolted on.** Every image gets signed and gets an ingredients list (SBOM) the moment it's built — not added later as an afterthought. Harbor can then refuse to hand out any image that isn't signed.

- **It all runs on the same platform.** VKS already gives us the cluster, the storage, the networking, Harbor (image store), and ArgoCD (deployment) — Tekton just plugs into what's already there instead of needing its own separate servers.

- **Works with no internet too.** Everything described here — BuildKit, Buildpacks, Tekton, and Cosign — can run fully offline, which matters for secure/air-gapped environments.

- **One dashboard to watch it all.** Tekton's Dashboard shows every build, whether it succeeded or failed, and why — in a web page, not just log files.

- **A push starts the build by itself.** The repository sends a webhook to Tekton Triggers, which creates the PipelineRun. Nobody has to start it manually, and commits made by the pipeline itself never start a second build.

Application Source Code → Tekton → Buildpacks / BuildKit → Image + SBOM → Cosign Signing → Harbor

## Benefits

| Benefit | In simple words |
|---|---|
| No guesswork on how to build | Tekton always knows: Dockerfile present → BuildKit, absent → Buildpacks. |
| Less work for app teams | Teams don't need to write or maintain a Dockerfile if they don't want to — Buildpacks handles it for them. |
| Fully automatic | A code push is the only human action; the build, sign, and publish steps run by themselves. |
| Every image is signed | Cosign signs the image and its ingredients list (SBOM) right after it's built, so nothing unverified reaches production. |
| Safer builds | BuildKit and Buildpacks both run without needing root/admin access inside the cluster. |
| One place for images | Every built image lands in Harbor, where it's scanned, verified, and stored. |
| Same steps everywhere | The exact same pipeline works whether the cluster has internet access or is completely offline. |
| You can always prove what's running | Images are tracked by a fixed ID (a digest), not a name that can change, and each one has a matching signed ingredients list. |
| One dashboard for visibility | Anyone can open the Tekton Dashboard and see the status of every build, without needing cluster access. |
| Builds start on their own | A push to the main branch sends a webhook to Tekton Triggers, which creates the PipelineRun — no manual PipelineRun needed. |
| No endless build loops | The commit the pipeline makes to update values.yaml is filtered out, so it does not trigger another build. |

## Architecture

All of this — Tekton, BuildKit, Buildpacks, and Cosign — runs inside the VKS cluster itself. Nothing extra needs to be installed outside it. Here's the whole journey from "developer pushes code" to "signed image is deployed" — everything after step 1 happens automatically, with no manual steps:

```
1. Developer pushes code
             │
             ▼
 2. Tekton wakes up (triggered by the push)
             │
             ▼
 3. Tekton checks: does this repo have a Dockerfile?
             │
    ┌────────┴────────┐
   Yes                 No
    │                   │
    ▼                   ▼
 4a. BuildKit        4b. Buildpacks
 builds the image     builds the image
 (using the           (using paketo
  Dockerfile)          builder-jammy-base)
    │                   │
    └────────┬──────────┘
             ▼
 5. An ingredients list (SBOM) is created for the image
             │
             ▼
 6. The image AND the ingredients list are digitally
    signed with Cosign
             │
             ▼
 7. Everything is pushed to Harbor
    (Harbor scans it and checks the signature)
             │
             ▼
 8. Harbor stores the image with a fixed ID (digest)
             │
             ▼
 9. GitOps updates the deployment files, and ArgoCD
    deploys the new, signed image onto VKS
```

| Step | What it means in plain words |
|---|---|
| Developer pushes code | The only thing a person actually has to do |
| Tekton | The automation engine — watches for pushes and runs the whole pipeline |
| Dockerfile check | The one decision point: which tool builds the image |
| BuildKit / Buildpacks | The two ways an image actually gets built |
| SBOM (ingredients list) | A record of everything that went into the image, for security and audit |
| Cosign signing | Proves the image really came from our pipeline and hasn't been tampered with |
| Harbor | Where every image is stored, scanned, and checked for a valid signature |
| ArgoCD | Takes the newly signed image and rolls it out to the running application |
| Webhook | The repository's call to Tekton on every push (step 1 → step 2) |
| EventListener | Receives the webhook, filters it (main branch only, ignores the pipeline's own commits) and creates the PipelineRun |
| Self-trigger filter | Stops the values.yaml commit from step 9 from starting the pipeline again |

How the push reaches Tekton (steps 1 and 2 in detail):

```
Developer pushes code
        │
        ▼
Repository sends a webhook (POST)
        │
        ▼
EventListener (NodePort)
        │
        ├── CEL filter: main branch only, ignore CI commits
        ├── TriggerBinding: reads repo URL, branch, commit
        └── TriggerTemplate: creates the PipelineRun
        │
        ▼
PipelineRun starts → step 3
```

## Prerequisites

### Platform

| Component | Version | What it's for |
|---|---|---|
| VMware Cloud Foundation (VCF) | 9.x | Sets up and manages the Supervisor and workload clusters |
| vSphere Kubernetes Service (VKS) | VCF 9.x bundled | Runs the actual Kubernetes workload cluster where builds happen |
| Regional Harbor | Installed via VCF Automation / Supervisor Service | Stores, scans, and signs images |
| Tekton Pipelines | Latest version supported by your cluster | Runs the CI pipeline — the automation engine described above |
| Tekton Dashboard | Same release train as Pipelines | The web page for watching build status |
| Cosign | Latest stable | Signs images and their ingredients lists (SBOMs) |
| Syft | Latest stable | Generates the ingredients list (SBOM) for each image |
| Tekton Triggers | Same release train as Pipelines | Receives the repository webhook and starts the pipeline automatically |

### Tools needed on your build machine

```
kubectl version
kubectl cluster-info
kubectl get nodes -o wide

docker --version    # optional, only needed for local testing
helm version
git --version
curl --version
```

## Installation of Tekton on VKS – Internet Connected VKS

Checked the current Tekton documentation. The current stable/LTS line is Tekton Pipelines v1.15.0, and Tekton's prerequisites include Kubernetes 1.28+, kubectl, and cluster-admin privileges.

First check your Kubernetes cluster

```
kubectl version #tekton version should compatible with k8s version
kubectl cluster-info
```

Check which cluster you are connected to

```
kubectl config current-context #to verify the cluster and its state
kubectl get nodes
```

Check your permissions

```
kubectl auth can-i create customresourcedefinitions
```

Expected: yes

```
kubectl auth can-i create clusterroles
```

Expected: yes

```
kubectl auth can-i create namespaces
```

Expected: yes

Check Internet connectivity from your workstation

```
ping www.google.com
```

Check Internet connectivity from the VKS nodes

```
kubectl get nodes -o wide
```

Make sure the workers have the expected network connectivity.

Later, when we install Buildpacks/BuildKit, the worker nodes will need to pull images such as:

```
paketobuildpacks/builder-jammy-base
moby/buildkit
```

Install Tekton Pipelines

```
kubectl apply --filename https://infra.tekton.dev/tekton-releases/pipeline/latest/release.yaml
```

You should see many resources being created, for example:

```
customresourcedefinition.apiextensions.k8s.io/... created
namespace/tekton-pipelines created
serviceaccount/... created
clusterrole.rbac.authorization.k8s.io/... created
deployment.apps/tekton-pipelines-controller created
deployment.apps/tekton-pipelines-webhook created
```

Immediately check the namespace

```
kubectl get namespace tekton-pipelines
```

Expected:

```
NAME                STATUS   AGE
tekton-pipelines    Active   ...
```

Check Tekton pods

```
kubectl get pods -n tekton-pipelines
```

Initially you may see:

```
NAME                                      READY   STATUS
tekton-pipelines-controller-xxxxx        0/1     ContainerCreating
tekton-pipelines-webhook-xxxxx            0/1     ContainerCreating
```

Wait a little.

Run again:

```
kubectl get pods -n tekton-pipelines
```

Eventually you want:

```
NAME                                      READY   STATUS
tekton-pipelines-controller-xxxxx        1/1     Running
tekton-pipelines-webhook-xxxxx            1/1     Running
```

The official documentation says the installation is complete when the components show 1/1 under READY

Check all Tekton resources

```
kubectl get all -n tekton-pipelines
```

You should see:

```
pods
services
deployments
replicasets
```

For example:

```
NAME                                           READY
pod/tekton-pipelines-controller-xxxxx         1/1
pod/tekton-pipelines-webhook-xxxxx             1/1
```

Check Tekton API resources Check Tekton CRDs

Tekton creates Kubernetes Custom Resource Definitions.

Run:

```
kubectl get crd | grep tekton
```

You should see resources related to:

```
tasks.tekton.dev
taskruns.tekton.dev
pipelines.tekton.dev
pipelineruns.tekton.dev
```

These are the important building blocks.

Conceptually:

```
Task
  ↓
TaskRun

Pipeline
  ↓
PipelineRun
```

```
kubectl api-resources | grep tekton
```

You should see resources similar to:

```
tasks
taskruns
pipelines
pipelineruns
```

This confirms Kubernetes recognizes Tekton's API resources.

Check Tekton controller

Run:

```
kubectl get deployment -n tekton-pipelines
```

Expected:

```
NAME                          READY   UP-TO-DATE   AVAILABLE
tekton-pipelines-controller   1/1     1            1
tekton-pipelines-webhook      1/1     1            1
```

Check controller logs

If everything is Running, you can still verify the controller logs:

```
kubectl logs deployment/tekton-pipelines-controller -n tekton-pipelines --tail=50
```

Look for errors such as:

```
ERROR
failed
panic
connection refused
```

Normal startup messages are fine.

Check services

Run:

```
kubectl get svc -n tekton-pipelines
```

You should see services associated with Tekton components.

Final installation validation

Run these commands one by one:

```
kubectl get nodes
kubectl get namespace tekton-pipelines
kubectl get pods -n tekton-pipelines
kubectl get deployments -n tekton-pipelines
kubectl get crd | grep tekton
kubectl api-resources | grep tekton
```

The important result is:

```
VKS nodes             → Ready
tekton-pipelines      → Active
Tekton controller     → 1/1 Running
Tekton webhook        → 1/1 Running
Tekton CRDs           → Present
```

## Integration for Pipeline creation + Tekton Dashboard.

For your Internet-connected VKS environment, I recommend this setup:

```
VKS Cluster
│
├── tekton-pipelines
│     ├── Tekton Controller
│     └── Tekton Webhook
│
├── tekton-dashboard
│     └── Tekton Dashboard
│
└── cicd
      ├── Tasks
      ├── Pipelines
      └── PipelineRuns
```

Verify Tekton Pipelines first

Run:

```
kubectl get pods -n tekton-pipelines
kubectl get crd | grep tekton
```

Create a namespace for your CI/CD objects

Don't put your application Pipelines directly into tekton-pipelines.

Create a separate namespace:

```
kubectl create namespace cicd
```

Verify:

```
kubectl get namespace cicd
```

Expected:

```
NAME    STATUS   AGE
cicd    Active   ...
```

Your architecture is now:

```
tekton-pipelines
        |
        | Tekton engine
        v
      cicd
        |
        +-- Tasks
        +-- Pipelines
        +-- PipelineRuns
```

Install Tekton Dashboard

Since your cluster has Internet access, use the official Tekton Dashboard release manifest.

Run:

```
kubectl apply -f https://infra.tekton.dev/tekton-releases/dashboard/latest/release.yaml
```

Tekton documents this as the installation method for Dashboard.

Check Dashboard namespace

Run:

```
kubectl get namespace tekton-pipelines
```

The Dashboard is normally installed into the Tekton namespace.

Now check:

```
kubectl get pods -n tekton-pipelines
```

You should see something similar to:

```
NAME                                      READY   STATUS
tekton-pipelines-controller-xxxxx        1/1     Running
tekton-pipelines-webhook-xxxxx            1/1     Running
tekton-dashboard-xxxxx                    1/1     Running
```

The exact pod names will be different.

Check Dashboard deployment

Run:

```
kubectl get deployment -n tekton-pipelines
```

You should see:

```
NAME                          READY
tekton-pipelines-controller   1/1
tekton-pipelines-webhook      1/1
tekton-dashboard              1/1
```

If tekton-dashboard isn't ready, don't continue to Pipeline testing yet.

Check Dashboard service

Run:

```
kubectl get svc -n tekton-pipelines
```

You should see a service similar to:

```
NAME               TYPE        CLUSTER-IP      PORT(S)
tekton-dashboard   ClusterIP   10.x.x.x        9097/TCP
```

For your initial implementation, don't expose it externally yet.

change svc type to NodePort to access the dashboard ui

Run:

```
kubectl edit svc tekton-dashboard  -n tekton-pipelines
```

change svc type to NodePort

```
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"app":"tekton-dashboard","app.kubernetes.io/component":"dashboard","app.kubernetes.io/instance":"default","app.kubernetes.io/name":"dashboard","app.kubernetes.io/part-of":"tekton-dashboard","app.kubernetes.io/version":"v0.72.0","dashboard.tekton.dev/release":"v0.72.0","version":"v0.72.0"},"name":"tekton-dashboard","namespace":"tekton-pipelines"},"spec":{"ports":[{"name":"http","port":9097,"protocol":"TCP","targetPort":9097}],"selector":{"app.kubernetes.io/component":"dashboard","app.kubernetes.io/instance":"default","app.kubernetes.io/name":"dashboard","app.kubernetes.io/part-of":"tekton-dashboard"}}}
  creationTimestamp: "2026-09-17T14:19:59Z"
  labels:
    app: tekton-dashboard
    app.kubernetes.io/component: dashboard
    app.kubernetes.io/instance: default
    app.kubernetes.io/name: dashboard
    app.kubernetes.io/part-of: tekton-dashboard
    app.kubernetes.io/version: v0.72.0
    dashboard.tekton.dev/release: v0.72.0
    version: v0.72.0
  name: tekton-dashboard
  namespace: tekton-pipelines
  resourceVersion: "229775"
  uid: 155bd1de-e1d9-40e0-bf55-8e25bf0fae37
spec:
  clusterIP: 198.62.134.202
  clusterIPs:
  - 198.62.134.202
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - name: http
    nodePort: 30583
    port: 9097
    protocol: TCP
    targetPort: 9097
  selector:
    app.kubernetes.io/component: dashboard
    app.kubernetes.io/instance: default
    app.kubernetes.io/name: dashboard
    app.kubernetes.io/part-of: tekton-dashboard
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```

**save**

**ACCESS the dashboard using the \<node ip : nodeport\>**

Open:

```
http://nodeip:9097
```

You should now see: Tekton Dashboard

This is your first integration validation.

production environment, we can expose it through:

```
Internet/User
      |
      v
Ingress / LoadBalancer
      |
      v
Tekton Dashboard
```

and configure:

- HTTPS

- DNS

- authentication

- RBAC

- network restrictions

Create a simple test Task

Before creating your actual Buildpacks/BuildKit Pipeline, we should verify that Tekton can execute a Task.

Create a file: test-task.yaml

Put:

```
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: hello-task
  namespace: cicd
spec:
  steps:
    - name: hello
      image: alpine:3.20
      script: |        #!/bin/sh
        echo "Hello from Tekton!"
        echo "Tekton Pipeline integration is working."
```

Apply the Task

Run:

```
kubectl apply -f test-task.yaml
```

Expected:

```
task.tekton.dev/hello-task created
```

Verify:

```
kubectl get tasks -n cicd
```

Expected:

```
NAME          AGE
hello-task    ...
```

Create TaskRun

A Task is the definition. A TaskRun actually executes it.

Create: hello-taskrun.yaml

```
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

```
kubectl apply -f hello-taskrun.yaml
```

Check TaskRun

Run:

```
kubectl get taskrun -n cicd
```

You should eventually see:

```
NAME            SUCCEEDED   REASON
hello-taskrun   True        Succeeded
```

You can also watch it:

```
kubectl get taskrun -n cicd -w
```

Check Task logs

Run:

```
kubectl logs -l tekton.dev/taskRun=hello-taskrun -n cicd
```

You should see:

```
Hello from Tekton!
Tekton Pipeline integration is working.
```

If that works, the basic Tekton execution engine is working.

See the TaskRun in Dashboard

Go back to:

```
http://localhost:9097
```

Select: Namespace → cicd

You should be able to see the Task/TaskRun.

This confirms:

```
Browser
   |
   v
Tekton Dashboard
   |
   v
Tekton API
   |
   v
TaskRun
   |
   v
Kubernetes Pod
```

## Tekton pipeline setup with buildpack, buildkit and cosign

Final architecture

```
            GitHub
                           |
                           v
              +-----------------------+
              | Tekton Pipeline       |
              | namespace: cicd       |
              +-----------+-----------+
                          |
                          v
                   Clone repository
                          |
                          v
                  Check Dockerfile
                     /          \
                    /            \
             Dockerfile          No Dockerfile
                 exists               |
                    |                 |
                    v                 v
               BuildKit          Buildpacks
                    |                 |
                    +--------+--------+
                             |
                             v
                       OCI Image
                             |
                             v
                         SBOM
                             |
                             v
                    +----------------+
                    |    Cosign      |
                    |                |
                    | Sign image     |
                    | Sign/attest    |
                    | SBOM           |
                    +-------+--------+
                            |
                            v
                    Local Harbor
                            |
                            v
              Signed Image + SBOM
```

Prerequisites

VKS cluster Tekton Pipelines Tekton Dashboard kubectl Internet connectivity

First check:

```
kubectl get nodes
```

Then:

```
kubectl get pods -n tekton-pipelines
```

You want Tekton components to be Running.

Create a namespace

Recommend using your own namespace rather than default.

Run:

```
kubectl create namespace cicd
```

If it already exists:

```
Error from server (AlreadyExists)
```

that's fine.

Verify:

```
kubectl get namespace cicd
```

Label the namespace:
 
```
kubectl label namespace cicd pod-security.kubernetes.io/enforce=privileged --overwrite
```
 # Tasks
 
 # Task creation 

### Git-Clone update task 
git-clone-update is a Tekton Task used to clone source code from a Git repository into a shared workspace.
It cleans the workspace, clones the required branch/revision, and validates the Git repository.
The cloned source code is then used by BuildKit or Buildpacks to build the container image.

### create git-clone-update-task.yaml
```
apiVersion: tekton.dev/v1
kind: Task

metadata:
  name: git-clone-update
  namespace: cicd

spec:
  description: Clone a Git repository

  params:
    - description: Git repository URL
      name: url
      type: string

    - default: main
      description: Git branch, tag, or commit
      name: revision
      type: string

    - default: "true"
      description: Delete existing workspace contents
      name: deleteExisting
      type: string

  steps:
    - computeResources: {}
      image: alpine/git:latest
      name: clone

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
        echo ""

        echo "Cleaning workspace..."
        rm -rf "${WORKSPACE}"/*
        rm -rf "${WORKSPACE}"/.[!.]*
        rm -rf "${WORKSPACE}"/..?*

        echo "Cloning repository..."
        git clone \
          --branch "$(params.revision)" \
          --depth 1 \
          "$(params.url)" \
          "${WORKSPACE}"

        echo "========================================"
        echo "Repository cloned successfully"
        echo "========================================"

        echo "Repository contents:"
        ls -la "${WORKSPACE}"

        echo ""
        echo "========================================"
        echo "Configuring Git safe.directory"
        echo "========================================"

        git config --global --add safe.directory "${WORKSPACE}"

        echo "safe.directory configured:"
        git config --global --get-all safe.directory

        echo ""
        echo "========================================"
        echo "Git status"
        echo "========================================"

        cd "${WORKSPACE}"
        git status

        echo ""
        echo "========================================"
        echo "Git Clone Completed"
        echo "========================================"

  workspaces:
    - description: Workspace where the Git repository will be cloned
      name: output
```
apply:
```
kubectl apply -f git-clone-update-task.yaml
```
verify:
```
kubectl get task git-clone-update -n cicd
```

### detect-build-type task 

detect-build-type is a Tekton Task that checks whether the source repository contains a Dockerfile.
It selects BuildKit when a Dockerfile exists, otherwise it selects Cloud Native Buildpacks.
The selected build type is stored as a Tekton result for the next pipeline task to use.

### detect-build-type

Create detect-build-type.yaml
```
piVersion: tekton.dev/v1
kind: Task

metadata:
  name: detect-build-type
  namespace: cicd

spec:
  workspaces:
    - name: source

  results:
    - name: BUILD_TYPE
      description: Build type: buildkit or buildpack

  steps:
    - name: detect
      image: alpine:3.20

      script: |
        #!/bin/sh
        set -eu

        echo "Checking source repository..."

        cd "$(workspaces.source.path)"

        if [ -f Dockerfile ]; then
          echo "Dockerfile found."
          echo "Using BuildKit."

          printf "buildkit" > "$(results.BUILD_TYPE.path)"
        else
          echo "Dockerfile not found."
          echo "Using Buildpacks."

          printf "buildpack" > "$(results.BUILD_TYPE.path)"
        fi
```
apply:

```
kubectl apply -f detect-build-type-task.yaml
```
verify:
```
kubectl get task detect-build-type -n cicd
```
 
## Buildpacks Task 

Buildpacks:  Phases Task	A ready-made Tekton Task that builds an image from source code.	Builds the image when there is no Dockerfile.

### Buildpacks-phases
  
Create buildpacks-phases.yaml
```
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: buildpacks-phases
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
    The Buildpacks-Phases task builds source into a container image and pushes it to
    a registry, using Cloud Native Buildpacks - https://buildpacks.io/. This task separately calls the aspects of the
    Cloud Native Buildpacks lifecycle, to provide increased security via container isolation.
 
    When the builder image includes extensions (= Dockerfiles), then this task will execute them.
    That allows to by example install packages, rpm, etc and to customize the build process according to your needs.
 
    This task supports the Platform spec 0.13: https://github.com/buildpacks/spec/blob/platform/v0.13/platform.md
 
  workspaces:
    - name: source
      description: Directory where application source is located.
    - name: cache
      description: Directory where cache is stored (when no cache image is provided).
      optional: true
    - name: dockerconfig
      description: Docker config for registry authentication.   # ✅ add this
      optional: true
 
  params:
    - name: CNB_BUILD_IMAGE
      description: Reference to the current build image in an OCI registry (if used <kaniko-dir> must be provided)
      default: ""
    - name: CNB_BUILDER_IMAGE
      description: The Builder image which includes the lifecycle tool, the buildpacks and metadata.
    - name: CNB_CACHE_IMAGE
      description: Reference to a cache image in an OCI registry (if no cache workspace is provided).
      default: ""
    - name: CNB_ENV_VARS
      type: array
      description: Environment variables to set during _build-time_.
      default: []
    - name: CNB_EXPERIMENTAL_MODE
      description: Control the lifecycle's execution according to the mode silent, warn, error for the experimental features.
      default: silent
    - name: CNB_GROUP_ID
      description: The group ID of the builder image user.
      default: ""
    - name: CNB_INSECURE_REGISTRIES
      description: List of registries separated by a comma having a self-signed certificate where TLS verification will be skipped.
      default: ""
    - name: CNB_LAYERS_DIR
      description: Path to layers directory
      default: "/layers"
    - name: CNB_LOG_LEVEL
      description: Logging level values info, warning, error, debug
      default: "info"
    - name: CNB_PLATFORM_API_SUPPORTED
      description: Buildpack Platform API supported by the Tekton task
      default: "0.13"
    - name: CNB_PLATFORM_API
      description: User's Buildpack Platform API
      default: ""
    - name: CNB_PLATFORM_DIR
      description: Path to the platform directory
      default: "/platform"
    - name: CNB_PROCESS_TYPE
      description: Default process type to set in the exported image
      # making it emppty so that buildpack pack can assign web
      default: ""
    - name: CNB_RUN_IMAGE
      description: Reference to an image which is packaging the application runtime to be launched.
      default: ""
    - name: CNB_SKIP_LAYERS
      description: Do not restore SBOM layer from previous image
      default: false
    # DEPRECATED: It does not make sense to support such an env variable as mounting the unix docker socket part of a pod from a host volume
    # will never happen for security reason
    # - name: CNB_USE_DAEMON
    #  description: Analyze image from docker daemon
    #  default: false
    - name: CNB_USER_ID
      description: The user ID of the builder image user.
      default: ""
 
    - name: APP_IMAGE
      description: The name of the container image for your application.
    - name: SOURCE_SUBPATH
      description: A subpath within the `source` input where the source to build is located.
      default: ""
    - name: TAGS
      description: Additional tag to apply to the exported image
      default: ""
    - name: USER_HOME
      description: Absolute path to the user's home directory.
      default: /tekton/home
    - name: INSPECT_TOOLS_IMAGE
      description: Image packaging tools like skopeo and jq to inspect the builder images
      default: quay.io/halkyonio/skopeo-jq:0.1.3@sha256:1b3d21ad541227dc9d3e793d18cef9eb00a969c0c01eb09cab88997bc63680c6
 
  results:
    - name: APP_IMAGE_DIGEST
      description: The digest of the built `APP_IMAGE`.
 
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
          description: UID of the user specified in the Builder image
        - name: GID
          description: GID of the user specified in the Builder image
        - name: EXTENSION_LABELS
          description: "Extensions labels: io.buildpacks.extension.layers defined in the Builder image"
        - name: CNB_PLATFORM_API
          description: The CNB_PLATFORM_API to be used by lifecycle and verified against the one supported by this task
      script: |
        #!/usr/bin/env bash
        set -eu
 
        if [ "${PARAM_VERBOSE}" = "debug" ] ; then
          set -x
        fi
        echo "Creating the path for docker.."
        mkdir -p /tekton/home/.docker
        echo "--> Copying dockerconfig credentials"
        ls -lrt "$(workspaces.source.path)/$(params.SOURCE_SUBPATH)/.docker/"
        if [[ -f "$(workspaces.source.path)/$(params.SOURCE_SUBPATH)/.docker/config.json" ]]; then
           cp "$(workspaces.source.path)/$(params.SOURCE_SUBPATH)/.docker/config.json" "/tekton/home/.docker/config.json"
           echo "Copied .dockerconfigjson to config.json"
        fi
        echo # Check if registry creds docker file has been mounted from a secret"
        if [[ -f "$HOME/.docker/config.json" ]]; then
          printf %"s\n" "The docker config.json file exists !"
        else
          printf %"s\n" "!!!!! Warning: No registry credentials file exist. So it could be possible that the task will fail due to docker rate limit, etc !!!"
        fi
 
        printf %"s\n" "Remove the @sha from the image as not supported by skopeo to inspect an image"
        CLEANED_IMAGE="${PARAM_BUILDER_IMAGE%@*}"
 
        EXT_LABEL_1="io.buildpacks.extension.layers"
        EXT_LABEL_2="io.buildpacks.buildpack.order-extensions"
        BUILDER_LABEL="io.buildpacks.builder.metadata"
 
        IMG_MANIFEST=$(skopeo inspect --authfile $HOME/.docker/config.json "docker://${CLEANED_IMAGE}")
 
        #
        # The following test should be reviewed as :
        #
        # 1) we get from non ubi images a {} value as you can see hereafter
        #   "io.buildpacks.extension.layers": "{}",
        #
        # 2) Do we have to check the content of this label too ?
        #    "io.buildpacks.buildpack.order-extensions": "null",
        #
 
        IMG_LABELS=$(echo $IMG_MANIFEST | jq -e '.Labels')
 
        if [[ $(echo "$IMG_LABELS" | jq -r '.["'${BUILDER_LABEL}'"]') != "{}" ]] > /dev/null; then
          printf %"s\n" "## The builder image ${PARAM_BUILDER_IMAGE} includes the label: \"${BUILDER_LABEL}\" :"
 
          builderLabel=$(echo -n "$IMG_LABELS" | jq -r '.["'${BUILDER_LABEL}'"]')
          platforms=($(echo $builderLabel | jq -r '.lifecycle.apis.platform.supported'))
          printf %"s\n" "Lifecycle platforms API supported: ${platforms[@]}"
 
          CNB_PLATFORM_API=${PARAM_CNB_PLATFORM_API:-$PARAM_CNB_PLATFORM_API_SUPPORTED}
          echo "Platform API selected: $CNB_PLATFORM_API"
          printf %"s\n" "Platform API supported by this task: $PARAM_CNB_PLATFORM_API_SUPPORTED"
 
          if [[ "${platforms[@]}" =~ "$CNB_PLATFORM_API" && "$CNB_PLATFORM_API" == "$PARAM_CNB_PLATFORM_API_SUPPORTED" ]]; then
              echo -n "$CNB_PLATFORM_API" > "$(step.results.CNB_PLATFORM_API.path)"
              printf %"s\n" "$CNB_PLATFORM_API is in the list of the platform supported by lifecycle like also this Tekton task :-)"
          else
              echo "$PARAM_CNB_PLATFORM_API is not in the list of the supported platform by lifecycle or is not supported by this tekton task: ${PARAM_CNB_PLATFORM_API_SUPPORTED} !"
              exit 1
          fi
        fi
 
        if [[ $(echo "$IMG_LABELS" | jq -r '.["'${EXT_LABEL_1}'"]') != "{}" ]] > /dev/null; then
          echo "## The builder image ${PARAM_BUILDER_IMAGE} includes some extensions as the extension label \"${EXT_LABEL_1}\" is NOT empty:"
          echo -n "$IMG_LABELS" | jq -r '.["'${EXT_LABEL_1}'"]' | tee "$(step.results.EXTENSION_LABELS.path)"
          echo ""
        else
          echo "## The builder image ${PARAM_BUILDER_IMAGE} dot not include extensions as the extension label \"${EXT_LABEL_1}\" is empty !"
          echo -n "empty" | tee "$(step.results.EXTENSION_LABELS.path)"
        fi
 
        CNB_USER_ID=$(echo $IMG_MANIFEST | jq -r '.Env' | jq -r '.[] | select(test("^CNB_USER_ID="))'  | cut -d '=' -f 2)
        CNB_GROUP_ID=$(echo $IMG_MANIFEST | jq -r '.Env' | jq -r '.[] | select(test("^CNB_GROUP_ID="))' | cut -d '=' -f 2)
 
        echo "## The CNB_USER_ID & CNB_GROUP_ID defined within the builder image: ${PARAM_BUILDER_IMAGE} are:"
        echo -n "$CNB_USER_ID"  | tee "$(step.results.UID.path)"
        echo ""
        echo -n "$CNB_GROUP_ID" | tee "$(step.results.GID.path)"
 
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
        #!/usr/bin/env bash
        set -eu
 
        echo "CNB UID: $CNB_USER_ID"
        echo "CNB GID: $CNB_GROUP_ID"
 
        if [[ "$(workspaces.cache.bound)" == "true" ]]; then
          echo "--> Setting permissions on '$(workspaces.cache.path)'..."
          chown -R "$CNB_USER_ID:$CNB_GROUP_ID" "$(workspaces.cache.path)"
        fi
 
        echo "--> Creating .docker folder"
        mkdir -p "/tekton/home/.docker"
 
        for path in "/tekton/home" "/tekton/home/.docker" "/tekton/creds" "/layers" "$(workspaces.source.path)"; do
          echo "--> Setting permissions on '$path'..."
          chown -R "$CNB_USER_ID:$CNB_GROUP_ID" "$path"
        done
 
        echo "--> Parsing additional configuration..."
        parsing_flag=""
        envs=()
        for arg in "$@"; do
            if [[ "$arg" == "--env-vars" ]]; then
                echo "-> Parsing env variables..."
                parsing_flag="env-vars"
            elif [[ "$parsing_flag" == "env-vars" ]]; then
                envs+=("$arg")
            fi
        done
 
        echo "--> Processing any environment variables..."
        ENV_DIR="/platform/env"
 
        echo "--> Creating 'env' directory: $ENV_DIR"
        mkdir -p "$ENV_DIR"
 
        for env in "${envs[@]}"; do
            IFS='=' read -r key value string <<< "$env"
            if [[ "$key" != "" && "$value" != "" ]]; then
                path="${ENV_DIR}/${key}"
                echo "--> Writing ${path}..."
                echo -n "$value" > "$path"
            fi
        done
        echo "--> Content of $(params.CNB_PLATFORM_DIR)/env"
        ls -la $(params.CNB_PLATFORM_DIR)/env
 
        echo "--> Show the project cloned within the workspace ..."
        ls -la $(workspaces.source.path)/$(params.SOURCE_SUBPATH)
 
      volumeMounts:
        - name: layers-dir
          mountPath: /layers
        - name: platform-dir
          mountPath: $(params.CNB_PLATFORM_DIR)
 
    - name: analyze
      image: $(params.CNB_BUILDER_IMAGE)
      imagePullPolicy: Always
      command: ["/cnb/lifecycle/analyzer"]
      env:
        - name: CNB_PLATFORM_API
          value: $(steps.get-labels-and-env.results.CNB_PLATFORM_API)
      args:
        - "-log-level=$(params.CNB_LOG_LEVEL)"
        - "-layers=$(params.CNB_LAYERS_DIR)"
        - "-run-image=$(params.CNB_RUN_IMAGE)"
        - "-cache-image=$(params.CNB_CACHE_IMAGE)"
        - "-uid=$(steps.get-labels-and-env.results.UID)"
        - "-gid=$(steps.get-labels-and-env.results.GID)"
        - "-insecure-registry=$(params.CNB_INSECURE_REGISTRIES)"
        - "-tag=$(params.TAGS)"
        - "-skip-layers=$(params.CNB_SKIP_LAYERS)"
        - "$(params.APP_IMAGE)"
      volumeMounts:
        - name: layers-dir
          mountPath: /layers
 
    - name: detect
      image: $(params.CNB_BUILDER_IMAGE)
      imagePullPolicy: Always
      command: ["/cnb/lifecycle/detector"]
      env:
        - name: CNB_PLATFORM_API
          value: $(steps.get-labels-and-env.results.CNB_PLATFORM_API)
      args:
        - "-log-level=$(params.CNB_LOG_LEVEL)"
        - "-app=$(workspaces.source.path)/$(params.SOURCE_SUBPATH)"
        - "-group=/layers/group.toml"
        - "-plan=/layers/plan.toml"
        - "-layers=$(params.CNB_LAYERS_DIR)"
        - "-platform=$(params.CNB_PLATFORM_DIR)"
      volumeMounts:
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
      script: |
        #!/usr/bin/env bash
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
        - name: layers-dir
          mountPath: /layers
        - name: kaniko-dir
          mountPath: /kaniko
 
    - name: extender
      when:
        - input: $(steps.get-labels-and-env.results.EXTENSION_LABELS)
          operator: notin
          values: ["empty"]
      image: $(params.CNB_BUILDER_IMAGE)
      imagePullPolicy: Always
      command: ["/cnb/lifecycle/extender"]
      env:
        - name: CNB_PLATFORM_API
          value: $(steps.get-labels-and-env.results.CNB_PLATFORM_API)
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
            - "SYS_ADMIN"
            - "SETFCAP"
      volumeMounts:
        - name: layers-dir
          mountPath: /layers
        - name: kaniko-dir
          mountPath: /kaniko
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
      command: ["/cnb/lifecycle/builder"]
      env:
        - name: CNB_PLATFORM_API
          value: $(steps.get-labels-and-env.results.CNB_PLATFORM_API)
      args:
        - "-log-level=$(params.CNB_LOG_LEVEL)"
        - "-app=$(workspaces.source.path)/$(params.SOURCE_SUBPATH)"
        - "-layers=$(params.CNB_LAYERS_DIR)"
        - "-group=/layers/group.toml"
        - "-plan=/layers/plan.toml"
        - "-platform=$(params.CNB_PLATFORM_DIR)"
      volumeMounts:
        - name: layers-dir
          mountPath: /layers
        - name: platform-dir
          mountPath: $(params.CNB_PLATFORM_DIR)
        - name: tekton-home-dir
          mountPath: /tekton/home
 
    - name: export
      image: $(params.CNB_BUILDER_IMAGE)
      imagePullPolicy: Always
      command: ["/cnb/lifecycle/exporter"]
      env:
        - name: CNB_PLATFORM_API
          value: $(steps.get-labels-and-env.results.CNB_PLATFORM_API)
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
        - name: layers-dir
          mountPath: /layers
 
    - name: results
      image: registry.access.redhat.com/ubi8/python-311@sha256:43605cb2491ef2297a7acf4b4bf0b7f54f0c91b96daf12ae41c49cc7f192b153
      script: |
        #!/usr/bin/env python3
 
        import tomllib
 
        def write_to_file(filename, content):
          with open(filename, "w") as f:
            f.write(content)
 
        with open("/layers/report.toml", "rb") as f:
            data = tomllib.load(f)
 
        img_data = data.get("image")
 
        tags = img_data.get("tags")
        digest = img_data.get("digest")
        image_id = img_data.get("image_id")
        manifest_size = img_data.get("manifest_size")
 
        print("#### Image data ####")
        print(f"tags: {tags}")
        print(f"Digest: {digest}")
 
        if None not in (image_id, manifest_size):
          print(f"image container id (when using daemon): {image_id}, manifest size: {manifest_size}")
 
        write_to_file('$(results.APP_IMAGE_DIGEST.path)',digest)
 
      volumeMounts:
        - name: layers-dir
          mountPath: /layers
 
  volumes:
    - name: tekton-home-dir
      emptyDir: {}
    - name: layers-dir
      emptyDir: {}
    - name: kaniko-dir
      emptyDir: {}
    - name: platform-dir
      emptyDir: {}

```
apply:
```
kubectl apply -f buildpacks-phases.yaml -n cicd
```

Verify:
```
kubectl get task buildpacks-phases -n cicd
```


### Install BuildKit Task

buildkit-build is a Tekton Task that runs when detect-build-type detects a Dockerfile, using BuildKit to build the application container image.
It takes the source code from the shared workspace, builds the image from the Dockerfile, and pushes it to Harbor with the configured credentials.

buildkit-build

create buildkit-build-task-update.yaml

```
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
          value: $(workspaces.dockerconfig.path)

      securityContext:
        privileged: true

      script: |
        #!/bin/sh
        set -eu

        echo "======================================"
        echo "Starting BuildKit build"
        echo "======================================"

        echo "IMAGE:"
        echo "$(params.IMAGE)"

        echo "DOCKERFILE:"
        echo "$(params.DOCKERFILE)"

        echo "CONTEXT:"
        echo "$(params.CONTEXT)"

        echo "Checking Dockerfile..."
        test -f "$(workspaces.source.path)/$(params.DOCKERFILE)"
        echo "Dockerfile found."

        # Fetch and trust Harbor CA certificate
        apk add --no-cache openssl ca-certificates 2>/dev/null || true

        openssl s_client \
          -connect lab25-harbor.lab25.sunfire.lab:443 \
          -showcerts </dev/null 2>/dev/null \
          | awk '/BEGIN CERTIFICATE/,/END CERTIFICATE/' \
          > /usr/local/share/ca-certificates/harbor-ca.crt

        update-ca-certificates 2>/dev/null || true

        # Create Docker config with Harbor credentials
        mkdir -p /tmp/dockerconfig

        HARBOR_AUTH=$(echo -n "admin:VMware1!" | base64 | tr -d '\n')

        cat > /tmp/dockerconfig/config.json <<EOF
        {
          "auths": {
            "lab25-harbor.lab25.sunfire.lab": {
              "auth": "${HARBOR_AUTH}"
            }
          }
        }
        EOF

        export DOCKER_CONFIG=/tmp/dockerconfig

        cd "$(workspaces.source.path)"

        echo "Creating BuildKit configuration..."

        cat > /tmp/buildkitd.toml <<EOF
        [registry."lab25-harbor.lab25.sunfire.lab"]
          insecure = true
        EOF

        echo "BuildKit configuration:"
        cat /tmp/buildkitd.toml

        echo "Starting BuildKit..."

        BUILDKITD_FLAGS="--config /tmp/buildkitd.toml" \
        buildctl-daemonless.sh build \
          --frontend dockerfile.v0 \
          --local context="$(workspaces.source.path)/$(params.CONTEXT)" \
          --local dockerfile="$(workspaces.source.path)" \
          --opt filename="$(params.DOCKERFILE)" \
          --output type=image,name="$(params.IMAGE)",push=true,name-canonical=true,registry.insecure=true \
          --metadata-file=/tmp/build-metadata.json

        echo "BuildKit build completed."

        echo ""
        echo "======================================"
        echo "BuildKit metadata"
        echo "======================================"

        cat /tmp/build-metadata.json

        echo ""
        echo "======================================"
        echo "Extracting image digest"
        echo "======================================"

        IMAGE_DIGEST=$(grep '"containerimage.digest"' /tmp/build-metadata.json \
          | sed 's/.*"containerimage.digest"[[:space:]]*:[[:space:]]*"\([^"]*\)".*/\1/')

        echo "IMAGE_DIGEST:"
        echo "${IMAGE_DIGEST}"

        if [ -z "${IMAGE_DIGEST}" ]; then
          echo "ERROR: BuildKit did not return image digest"
          exit 1
        fi

        case "${IMAGE_DIGEST}" in
          sha256:*)
            echo "Valid SHA256 digest."
            ;;
          *)
            echo "ERROR: Invalid digest:"
            echo "${IMAGE_DIGEST}"
            exit 1
            ;;
        esac

        echo ""
        echo "======================================"
        echo "Writing Tekton IMAGE_DIGEST result"
        echo "======================================"

        printf '%s' "${IMAGE_DIGEST}" > "$(results.IMAGE_DIGEST.path)"

        printf '%s' "${IMAGE_DIGEST}" \
          > "$(workspaces.source.path)/image-digest"

        echo "Tekton result:"
        cat "$(results.IMAGE_DIGEST.path)"

        echo "Common image digest file:"
        cat "$(workspaces.source.path)/image-digest"
```
apply 

```
kubectl apply -f buildkit-build-task-update.yaml 
```
Verify
```
kubectl get task buildkit-build -n cicd
```

### cosign task 
This sign-image Task is the security/signing stage after BuildKit or Buildpacks. Its job is to generate an SBOM, 
sign the image with Cosign, attach the SBOM as an attestation, and return the image digest.

sign-image and generate SBOM

create sign-image-task-update.yaml

```
apiVersion: tekton.dev/v1
kind: Task

metadata:
  name: sign-image
  namespace: cicd

spec:

  params:
    - name: IMAGE
      type: string

  results:
    - name: IMAGE_DIGEST
      description: Digest of the image that was signed
      type: string

  steps:

    # ============================================================
    # STEP 1 - Generate SBOM
    # ============================================================

    - name: sbom
      image: anchore/syft:latest
      computeResources: {}

      env:
        - name: DOCKER_CONFIG
          value: $(workspaces.dockerconfig.path)

        - name: SYFT_REGISTRY_INSECURE_SKIP_TLS_VERIFY
          value: "true"

      command:
        - /syft

      args:
        - scan
        - $(params.IMAGE)
        - "-o"
        - spdx-json=$(workspaces.source.path)/sbom.spdx.json


    # ============================================================
    # STEP 2 - Cosign image
    # ============================================================

    - name: sign
      image: ghcr.io/sigstore/cosign/cosign:v3.0.2
      computeResources: {}

      env:
        - name: DOCKER_CONFIG
          value: $(workspaces.dockerconfig.path)

        - name: COSIGN_INSECURE_IGNORE_SCT
          value: "true"

        - name: COSIGN_PASSWORD
          valueFrom:
            secretKeyRef:
              name: cosign-password
              key: password

        - name: REGISTRY_USERNAME
          valueFrom:
            secretKeyRef:
              name: harbor-credentials
              key: username

        - name: REGISTRY_PASSWORD
          valueFrom:
            secretKeyRef:
              name: harbor-credentials
              key: password

      command:
        - /ko-app/cosign

      args:
        - sign
        - --yes
        - --allow-insecure-registry
        - --registry-username=$(REGISTRY_USERNAME)
        - --registry-password=$(REGISTRY_PASSWORD)
        - --key
        - $(workspaces.cosign.path)/cosign.key
        - $(params.IMAGE)


    # ============================================================
    # STEP 3 - Attach SBOM attestation
    # ============================================================

    - name: attest-sbom
      image: ghcr.io/sigstore/cosign/cosign:v3.0.2
      computeResources: {}

      env:
        - name: DOCKER_CONFIG
          value: $(workspaces.dockerconfig.path)

        - name: COSIGN_INSECURE_IGNORE_SCT
          value: "true"

        - name: COSIGN_PASSWORD
          valueFrom:
            secretKeyRef:
              name: cosign-password
              key: password

        - name: REGISTRY_USERNAME
          valueFrom:
            secretKeyRef:
              name: harbor-credentials
              key: username

        - name: REGISTRY_PASSWORD
          valueFrom:
            secretKeyRef:
              name: harbor-credentials
              key: password

      command:
        - /ko-app/cosign

      args:
        - attest
        - --yes
        - --allow-insecure-registry
        - --registry-username=$(REGISTRY_USERNAME)
        - --registry-password=$(REGISTRY_PASSWORD)
        - --key
        - $(workspaces.cosign.path)/cosign.key
        - --type
        - spdxjson
        - --predicate
        - $(workspaces.source.path)/sbom.spdx.json
        - $(params.IMAGE)


    # ============================================================
    # STEP 4 - Read common image digest
    #
    # BuildKit OR Buildpacks creates:
    #
    #   $(workspaces.source.path)/image-digest
    #
    # This step reads the same file regardless of
    # which build method was used.
    # ============================================================

    - name: image-digest
      image: alpine:3.20
      computeResources: {}

      script: |
        #!/bin/sh

        set -eu

        echo "======================================"
        echo "Reading image digest"
        echo "======================================"

        DIGEST_FILE="$(workspaces.source.path)/image-digest"

        echo "Digest file:"
        echo "${DIGEST_FILE}"

        echo ""
        echo "Checking digest file..."

        if [ ! -f "${DIGEST_FILE}" ]; then
          echo "ERROR: Image digest file not found:"
          echo "${DIGEST_FILE}"
          exit 1
        fi

        IMAGE_DIGEST="$(cat "${DIGEST_FILE}")"

        echo ""
        echo "Image digest:"
        echo "${IMAGE_DIGEST}"

        # Remove accidental whitespace/newline
        IMAGE_DIGEST="$(echo "${IMAGE_DIGEST}" | tr -d '[:space:]')"

        echo ""
        echo "Cleaned image digest:"
        echo "${IMAGE_DIGEST}"

        # Validate digest
        case "${IMAGE_DIGEST}" in
          sha256:*)
            echo ""
            echo "Valid SHA256 digest."
            ;;
          *)
            echo ""
            echo "ERROR: Invalid image digest:"
            echo "${IMAGE_DIGEST}"
            exit 1
            ;;
        esac

        # Write Tekton result
        printf '%s' "${IMAGE_DIGEST}" > "$(results.IMAGE_DIGEST.path)"

        echo ""
        echo "======================================"
        echo "Tekton IMAGE_DIGEST result"
        echo "======================================"

        cat "$(results.IMAGE_DIGEST.path)"


  # ==============================================================
  # WORKSPACES
  # ==============================================================

  workspaces:

    - name: dockerconfig

    - name: cosign

#    - name: output

    - name: source

```
apply:
```
kubectl apply -f sign-image-task-update.yaml
```

Verify:

```
kubectl get task sign-image -n cicd
```

###  update-values task

update-values is the GitOps update stage of the Tekton pipeline. It updates the Helm chart's values.yaml 
with the newly built image and its digest, commits the change, and pushes it back to repository.

update-values

create update-values.yaml

```
apiVersion: tekton.dev/v1
kind: Task

metadata:
  name: update-values
  namespace: cicd

spec:

  description: >
    Clone the GitHub repository, update Helm values.yaml
    with the newly built Harbor image, commit the change,
    and push it back to GitHub.

  params:

    - name: REPO_URL
      type: string
      description: Git repository URL
      default: https://github.com/kondurupurandhar/TravelPortal-test-buildpacks.git

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
      default: Update TravelPortal image

  workspaces:

    - name: source
      description: Workspace used by this task

  steps:

    - name: update-and-push

      image: alpine/git:latest

      env:

        - name: GIT_USERNAME
          valueFrom:
            secretKeyRef:
              name: github-push-secret
              key: username

        - name: GIT_TOKEN
          valueFrom:
            secretKeyRef:
              name: github-push-secret
              key: token

      script: |
        #!/bin/sh

        set -eu

        WORKDIR="$(workspaces.source.path)"

        echo "========================================"
        echo "Update Helm values.yaml"
        echo "========================================"

        echo ""
        echo "Workspace:"
        echo "${WORKDIR}"

        echo ""
        echo "Repository:"
        echo "$(params.REPO_URL)"

        echo ""
        echo "Branch:"
        echo "$(params.GIT_BRANCH)"

        echo ""
        echo "Image:"
        echo "$(params.IMAGE)"

        echo ""
        echo "Image digest:"
        echo "$(params.IMAGE_DIGEST)"

        echo ""
        echo "Values file:"
        echo "$(params.VALUES_FILE)"

        # ----------------------------------------
        # Clean workspace
        # ----------------------------------------

        echo ""
        echo "========================================"
        echo "Cleaning workspace"
        echo "========================================"

        rm -rf "${WORKDIR:?}"/*
        rm -rf "${WORKDIR}"/.[!.]*
        rm -rf "${WORKDIR}"/..?*

        # ----------------------------------------
        # Clone repository
        # ----------------------------------------

        echo ""
        echo "========================================"
        echo "Cloning Git repository"
        echo "========================================"

        git clone \
          --branch "$(params.GIT_BRANCH)" \
          --depth 1 \
          "$(params.REPO_URL)" \
          "${WORKDIR}"

        echo ""
        echo "Repository cloned successfully."

        # ----------------------------------------
        # Configure Git safe directory
        # ----------------------------------------

        echo ""
        echo "========================================"
        echo "Configuring Git safe.directory"
        echo "========================================"

        git config --global --add safe.directory "${WORKDIR}"

        # ----------------------------------------
        # Enter repository
        # ----------------------------------------

        cd "${WORKDIR}"

        echo ""
        echo "========================================"
        echo "Git repository"
        echo "========================================"

        git status

        echo ""
        echo "Current branch:"
        git branch --show-current

        echo ""
        echo "Repository contents:"
        ls -la

        # ----------------------------------------
        # Check values.yaml
        # ----------------------------------------

        VALUES_FILE="${WORKDIR}/$(params.VALUES_FILE)"

        echo ""
        echo "========================================"
        echo "Checking values.yaml"
        echo "========================================"

        if [ ! -f "${VALUES_FILE}" ]; then

          echo "ERROR: values.yaml not found:"
          echo "${VALUES_FILE}"

          echo ""
          echo "Searching for values.yaml..."

          find "${WORKDIR}" \
            -type f \
            -name "values.yaml" \
            -print

          exit 1

        fi

        echo ""
        echo "Current values.yaml:"
        echo "----------------------------------------"

        cat "${VALUES_FILE}"

        echo "----------------------------------------"

        # ----------------------------------------
        # Extract image repository,  tag and digest
        # ----------------------------------------

        IMAGE="$(params.IMAGE)"
        IMAGE_REPOSITORY="${IMAGE%:*}"
        IMAGE_TAG="${IMAGE##*:}"
        IMAGE_DIGEST="$(params.IMAGE_DIGEST)"

        echo ""
        echo "========================================"
        echo "Image information"
        echo "========================================"

        echo "Full image:"
        echo "${IMAGE}"

        echo ""
        echo "Image repository:"
        echo "${IMAGE_REPOSITORY}"

        echo ""
        echo "Image tag:"
        echo "${IMAGE_TAG}"

        echo ""
        echo "Image digest:"
        echo "${IMAGE_DIGEST}"


        # ----------------------------------------
        # Validate digest
        # ----------------------------------------

        echo ""
        echo "========================================"
        echo "Validating image digest"
        echo "========================================"

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

        # ----------------------------------------
        # Update image repository, tag and digest
        # ----------------------------------------

        echo ""
        echo "========================================"
        echo "Updating values.yaml"
        echo "========================================"

        sed -i \
          "/^image:/,/^env:/ s|^  repository:.*|  repository: ${IMAGE_REPOSITORY}|" \
          "${VALUES_FILE}"

        sed -i \
          "/^image:/,/^env:/ s|^  tag:.*|  tag: \"${IMAGE_TAG}\"|" \
          "${VALUES_FILE}"

        sed -i \
          "/^image:/,/^env:/ s|^  digest:.*|  digest: \"${IMAGE_DIGEST}\"|" \
          "${VALUES_FILE}"


        echo ""
        echo "Updated values.yaml:"
        echo "----------------------------------------"

        cat "${VALUES_FILE}"

        echo "----------------------------------------"

        # ----------------------------------------
        # Configure Git
        # ----------------------------------------

        echo ""
        echo "========================================"
        echo "Configuring Git"
        echo "========================================"

        git config user.name "Tekton CI"
        git config user.email "tekton-ci@local"

        git config --global --add safe.directory "${WORKDIR}"

        # ----------------------------------------
        # Configure GitHub authentication
        # ----------------------------------------

        echo ""
        echo "========================================"
        echo "Configuring GitHub authentication"
        echo "========================================"

        git remote set-url origin \
          "https://${GIT_USERNAME}:${GIT_TOKEN}@github.com/kondurupurandhar/TravelPortal-test-buildpacks.git"

        echo "GitHub remote configured."

        # ----------------------------------------
        # Git diff
        # ----------------------------------------

        echo ""
        echo "========================================"
        echo "Git diff"
        echo "========================================"

        git diff -- "$(params.VALUES_FILE)"

        # ----------------------------------------
        # Check if anything changed
        # ----------------------------------------

        echo ""
        echo "========================================"
        echo "Checking for changes"
        echo "========================================"

        if git diff --quiet -- "$(params.VALUES_FILE)"; then

          echo "No changes detected in values.yaml."

          exit 0

        fi

        echo "Changes detected."

        # ----------------------------------------
        # Git add
        # ----------------------------------------

        echo ""
        echo "========================================"
        echo "Git add"
        echo "========================================"

        git add "$(params.VALUES_FILE)"

        git status

        # ----------------------------------------
        # Git commit
        # ----------------------------------------

        echo ""
        echo "========================================"
        echo "Git commit"
        echo "========================================"

        git commit \
          -m "$(params.COMMIT_MESSAGE): ${IMAGE}"

        # ----------------------------------------
        # Git push
        # ----------------------------------------

        echo ""
        echo "========================================"
        echo "Git push"
        echo "========================================"

        git push origin "$(params.GIT_BRANCH)"

        echo ""
        echo "========================================"
        echo "SUCCESS"
        echo "========================================"

        echo "Helm values.yaml updated."
        echo "Git commit created."
        echo "Changes pushed to GitHub."

```
apply:
```
kubectl apply -f update-values.yaml
```
Verify:

```
kubectl get task update-values -n cicd
```

# pipeline creation

Tasks are added to the Pipeline using taskRef, which references previously created Tekton Tasks. params pass required values to each Task, workspaces share files/credentials, and runAfter/when control the execution order and conditions.
The Pipeline first clones the repository, then detects whether a Dockerfile exists and runs BuildKit or Buildpacks accordingly. After the image is built, the sign Task generates the SBOM and signs/attests the image.
Finally, the update-values Task receives the image digest from the sign Task, updates values.yaml, commits the change, and pushes it back to GitHub—completing the GitOps update.

###  Pipeline

create travelportal-pipeline-values-update.yaml

```
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: travelportal-pipeline-values-update
  namespace: cicd
spec:
  params:
    - name: REPO_URL
      type: string
      default: https://github.com/kondurupurandhar/TravelPortal-test-buildpacks.git 
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
   # - name: sbom
   # - name: git-credentials

  tasks:

    # ----------------------------------------
    # 1. Clone GitHub repository
    # ----------------------------------------
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

    # ----------------------------------------
    # 2. Detect BuildKit vs Buildpacks
    # ----------------------------------------
    - name: detect
      runAfter:
        - clone
      taskRef:
        name: detect-build-type
      workspaces:
        - name: source
          workspace: source

    # ----------------------------------------
    # 3. Build using BuildKit
    # ----------------------------------------
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

    # ----------------------------------------
    # 4. Build using Buildpacks
    # ----------------------------------------
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



    # ----------------------------------------
    # 5. Generate SBOM + Sign
    # ----------------------------------------
    - name: sign
      runAfter:
        - buildkit
        - buildpack
      taskRef:
        name: sign-image
      params:
        - name: IMAGE
          value: "$(params.IMAGE)"
      workspaces:
        - name: dockerconfig
          workspace: dockerconfig
        - name: cosign
          workspace: cosign-key
#        - name: output
#          workspace: sbom
        - name: source
          workspace: source

    # ----------------------------------------
    # 6. Update values.yaml and push to Git
    # ----------------------------------------
    - name: update-values
      runAfter:
        - sign
      taskRef:
        name: update-values
      params:
        - name: REPO_URL
          value: "$(params.REPO_URL)"

        - name: IMAGE
          value: "$(params.IMAGE)"

        - name: IMAGE_DIGEST
          value: $(tasks.sign.results.IMAGE_DIGEST)

        - name: VALUES_FILE
          value: "helm-charts/values.yaml"         

        - name: GIT_BRANCH
          value: "$(params.REVISION)"

        - name: COMMIT_MESSAGE
          value: "Update TravelPortal image"

      workspaces:
        - name: source
          workspace: source

       # - name: git-credentials
       #   workspace: git-credentials

```
apply: 

```
kubectl apply -f travelportal-pipeline-values-update.yaml
```

Verify:

```
kubectl get pipeline -n cicd
```

### piplinerun 

The PipelineRun is used to start and execute the travelportal-pipeline-values-update Pipeline. It provides the pipeline parameters,connects the required workspaces and secrets, and applies additional Pod configuration needed during the pipeline execution.
The pipelineRef selects the travelportal-pipeline-values-update Pipeline, while params provide the GitHub repository, branch, Harbor image, and Buildpacks builder image that the Pipeline will use.
The workspaces provide the required storage and credentials: the source PVC stores application files, dockerconfig provides Harbor authentication, cosign-key provides the image-signing key, and the sbom PVC provides storage for SBOM data.
The taskRunSpecs customizes the BuildKit TaskRun by adding the harbor-ca ConfigMap as a volume. This allows the BuildKit Pod to access the Harbor CA certificate for secure TLS communication with the Harbor registry.

Pipelinerun

create travelportal-pipelinerun-pack-values-update.yaml

```
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
      value: https://github.com/kondurupurandhar/TravelPortal-test-buildpacks.git
    - name: REVISION
      value: main
    - name: IMAGE
      value: lab25-harbor.lab25.sunfire.lab/cicd/travelportal:latest
    - name: BUILDER_IMAGE
      value: paketobuildpacks/builder-jammy-base
  workspaces:
    - name: source
      persistentVolumeClaim:
        claimName: buildpacks-source-pvc
    - name: dockerconfig
      secret:
        secretName: harbor-registry-secret
    - name: cosign-key
      secret:
        secretName: cosign-key
    - name: sbom
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          storageClassName: lab-gold-storage-policy
          resources:
            requests:
              storage: 5Gi
   # - name: git-credentials
   #   secret:
   #     secretName: github-push-secret

  taskRunSpecs:
    - pipelineTaskName: buildkit
      podTemplate:
        volumes:
          - name: harbor-ca
            configMap:
              name: harbor-ca-cert

```
apply:

```
kubectl apply -f  travelportal-pipelinerun-pack-values-update.yaml
```

verify:
```
kubectl get pipelinerun -n cicd
```


Verify image in Harbor

From a machine that can reach Harbor:

```
docker login lab25-harbor.lab25.sunfire.lab
```
then
```
docker pull \
  lab25-harbor.lab25.sunfire.lab/cicd/travelportal:latest
```

You can also inspect it using:

```
docker images
```

Verify Cosign signature

Use:

```
cosign verify \
  --key cosign.pub \
  lab25-harbor.lab25.sunfire.lab/cicd/travelportal:latest
```

Verify SBOM attestation

Use:

```
cosign verify-attestation \
  --key cosign.pub \
  --type spdxjson \
  lab25-harbor.lab25.sunfire.lab/cicd/travelportal:latest
```

This validates that the SBOM attestation is associated with the image.

## Automatic pipeline trigger with Tekton Triggers

Until now the PipelineRun was started by hand with `kubectl apply`. In this section a push to the repository starts it automatically. The repository server sits on the internal lab network, so it calls the Tekton EventListener NodePort directly.

Lab values used in this section:

```
Namespace                : cicd
Pipeline                 : travelportal-pipeline-values-update
EventListener            : travelportal-github-listener
EventListener Service    : el-travelportal-github-listener (NodePort)
Webhook port             : 8080 -> 31877
Node used for webhook    : 10.12.92.3
Webhook URL              : http://10.12.92.3:31877
Repository server        : http://10.12.90.62
Repository               : http://10.12.90.62/admin/travelPortal-test-buildpack.git
Branch                   : main
```

The complete flow:

```
Developer
   |
   | git push origin main
   v
Repository (10.12.90.62)
   |
   | POST webhook
   v
NodePort 10.12.92.3:31877
   |
   v
Tekton EventListener
travelportal-github-listener
   |
   +--> CEL interceptor
   |      +--> only main branch
   |      +--> ignore commits made by the pipeline itself
   |
   +--> TriggerBinding
   |
   +--> TriggerTemplate
   |
   v
PipelineRun
   |
   v
travelportal-pipeline-values-update
   |
   +--> clone
   +--> detect
   +--> BuildKit OR Buildpacks
   +--> sign
   +--> update-values
   |
   v
Commit of helm-charts/values.yaml to the repository
```

The Tekton Triggers objects created in this section:

```
ServiceAccount        github-trigger-sa
Role                  github-trigger-role
RoleBinding           github-trigger-rolebinding
ClusterRole           github-trigger-cluster-role
ClusterRoleBinding    github-trigger-cluster-rolebinding
EventListener         travelportal-github-listener
CEL interceptor       ClusterInterceptor
TriggerBinding        travelportal-github-binding
TriggerTemplate       travelportal-github-template
```

The EventListener name contains `github` for historical reasons. The name does not affect functionality.

Check that Tekton Triggers is installed

```
kubectl get crd | grep triggers.tekton.dev
```

You should see:

```
clusterinterceptors.triggers.tekton.dev
clustertriggerbindings.triggers.tekton.dev
eventlisteners.triggers.tekton.dev
interceptors.triggers.tekton.dev
triggerbindings.triggers.tekton.dev
triggers.triggers.tekton.dev
triggertemplates.triggers.tekton.dev
```

Check the Triggers pods

```
kubectl get pods -n tekton-pipelines | grep triggers
kubectl get deployment -n tekton-pipelines | grep triggers
```

You should see:

```
tekton-triggers-controller-...
tekton-triggers-core-interceptors-...
tekton-triggers-webhook-...
```

Create the EventListener ServiceAccount and RBAC

The EventListener runs with:

```
serviceAccountName: github-trigger-sa
```

It needs permission to read the Triggers resources in cicd, read the cluster-scoped Triggers resources (`clusterinterceptors`, `clustertriggerbindings`), and create PipelineRuns.

Create the ServiceAccount

webhook-sa.yaml:

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: github-trigger-sa
  namespace: cicd
```

Apply:

```
kubectl apply -f webhook-sa.yaml
```

Create Role (RBAC)

github-trigger-rbac.yaml:

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: github-trigger-role
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

```
kubectl apply -f github-trigger-rbac.yaml
```

Create RoleBinding (RBAC)

github-trigger-rolebinding.yaml:

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: github-trigger-rolebinding
  namespace: cicd
subjects:
  - kind: ServiceAccount
    name: github-trigger-sa
    namespace: cicd
roleRef:
  kind: Role
  name: github-trigger-role
  apiGroup: rbac.authorization.k8s.io
```

Apply:

```
kubectl apply -f github-trigger-rolebinding.yaml
```

Create cluster Role (RBAC)

clusterrole.yaml

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: github-trigger-cluster-role
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

```
kubectl apply -f clusterrole.yaml
```

Create ClusterRoleBinding

github-trigger-cluster-rbac.yaml:

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: github-trigger-cluster-rolebinding
subjects:
  - kind: ServiceAccount
    name: github-trigger-sa
    namespace: cicd
roleRef:
  kind: ClusterRole
  name: github-trigger-cluster-role
  apiGroup: rbac.authorization.k8s.io
```

Apply:

```
kubectl apply -f github-trigger-cluster-rbac.yaml
```

Verify the objects:

```
kubectl get sa github-trigger-sa -n cicd
kubectl get role github-trigger-role -n cicd
kubectl get rolebinding github-trigger-rolebinding -n cicd
kubectl get clusterrole github-trigger-cluster-role
kubectl get clusterrolebinding github-trigger-cluster-rolebinding
```

Verify the permissions

```
kubectl auth can-i list triggerbindings.triggers.tekton.dev \
  --as=system:serviceaccount:cicd:github-trigger-sa -n cicd

kubectl auth can-i list triggertemplates.triggers.tekton.dev \
  --as=system:serviceaccount:cicd:github-trigger-sa -n cicd

kubectl auth can-i list eventlisteners.triggers.tekton.dev \
  --as=system:serviceaccount:cicd:github-trigger-sa -n cicd

kubectl auth can-i list clusterinterceptors.triggers.tekton.dev \
  --as=system:serviceaccount:cicd:github-trigger-sa

kubectl auth can-i list clustertriggerbindings.triggers.tekton.dev \
  --as=system:serviceaccount:cicd:github-trigger-sa

kubectl auth can-i create pipelineruns.tekton.dev \
  --as=system:serviceaccount:cicd:github-trigger-sa -n cicd
```

Expected for all of them:

```
yes
```

By default the repository server blocks outbound webhook calls to hosts that are not on its allow list. Without the change below, the webhook fails with:

```
webhook can only call allowed HTTP servers
(check your security.ALLOWED_HOST_LIST setting)
```

Create the TriggerBinding

github-trigger-binding.yaml

```
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
  name: travelportal-github-binding
  namespace: cicd
spec:
  params:
    - name: REPO_URL
      value: http://10.12.90.62/admin/travelPortal-test-buildpack.git

    - name: REVISION
      value: main

    - name: COMMIT_MESSAGE
      value: $(body.head_commit.message)

    - name: COMMIT_SHA
      value: $(body.after)
```

Apply:

```
kubectl apply -f github-trigger-binding.yaml
```

Verify:

```
kubectl get triggerbinding travelportal-github-binding -n cicd -o yaml
```

The important values are:

```
REPO_URL = http://10.12.90.62/admin/travelPortal-test-buildpack.git
REVISION = main
```

Create the TriggerTemplate

github-trigger-template.yaml

```
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: travelportal-github-template
  namespace: cicd
spec:
  params:
    - name: REPO_URL
    - name: REVISION
      default: main
    - name: COMMIT_MESSAGE
      default: Repository push - TravelPortal build
    - name: COMMIT_SHA

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
          - name: REVISION
            value: $(tt.params.REVISION)
          - name: IMAGE
            value: lab25-harbor.lab25.sunfire.lab/cicd/travelportal:latest
          - name: BUILDER_IMAGE
            value: paketobuildpacks/builder-jammy-base
        taskRunTemplate:
          serviceAccountName: default
        taskRunSpecs:
          - pipelineTaskName: buildkit
            podTemplate:
              volumes:
                - name: harbor-ca
                  configMap:
                    name: harbor-ca-cert
        timeouts:
          pipeline: 1h0m0s
        workspaces:
          - name: source
            persistentVolumeClaim:
              claimName: buildpacks-source-pvc
          - name: dockerconfig
            secret:
              secretName: harbor-registry-secret
          - name: cosign-key
            secret:
              secretName: cosign-key
```

Apply:

```
kubectl apply -f github-trigger-template.yaml
```

Verify:

```
kubectl get triggertemplate travelportal-github-template -n cicd -o yaml
```

The PipelineRun must reference `travelportal-pipeline-values-update` and pass the repository URL `http://10.12.90.62/admin/travelPortal-test-buildpack.git`.

Create the EventListener

github-eventlistener.yaml

```
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: travelportal-github-listener
  namespace: cicd
spec:
  serviceAccountName: github-trigger-sa

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
        - ref: travelportal-github-binding

      template:
        ref: travelportal-github-template
```

Apply:

```
kubectl apply -f github-eventlistener.yaml
```

Verify:

```
kubectl get eventlistener travelportal-github-listener -n cicd
```

Expected:

```
AVAILABLE=True
READY=True
```

Verify the CEL ClusterInterceptor

The EventListener uses the cluster-scoped `cel` interceptor. Verify that it is available:

```
kubectl get clusterinterceptors
kubectl get clusterinterceptor cel -o yaml
```

Check the generated Service:

```
kubectl get svc el-travelportal-github-listener -n cicd
```

The webhook must use the application port 8080, which is mapped to 31877, so the webhook URL is:

```
http://10.12.92.3:31877
```

Check the Service endpoint and the EventListener pod:

```
kubectl get endpoints el-travelportal-github-listener -n cicd -o wide
```

```
kubectl get pods -n cicd \
  -l eventlistener=travelportal-github-listener \
  -o wide
```

Test the EventListener before adding the webhook

Test the NodePort:

```
curl -v http://10.12.92.3:31877
```

Send a webhook-style POST:

```
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

A working EventListener returns:

```
HTTP/1.1 202 Accepted
```

with a Tekton event ID. This proves the path jumpbox → NodePort → EventListener works.

Add the webhook in the repository

Repository:

```
admin/travelPortal-test-buildpack
```

Open:

```
Repository
  -> Settings
  -> Webhooks
  -> Add Webhook
```

Use:

```
Target URL   : http://10.12.92.3:31877
Method       : POST
Content type : application/json
Event        : Push events
Branch       : main
```

Save the webhook, then use **Test Delivery** or push a real commit.

The values read from the push payload are:

```
body.ref
body.after
body.head_commit.message
```

Pipeline changes for automatic runs

Image digest

BuildKit and Buildpacks return their digest under different result names:

```
BuildKit    -> IMAGE_DIGEST
Buildpacks  -> APP_IMAGE_DIGEST
```

Both Tasks also write the same file to the shared workspace:

```
$(workspaces.source.path)/image-digest
```

Repository credentials for update-values

Create a Personal Access Token in the repository server and store it as a Secret.

```
kubectl create secret generic repo-git-credentials \
  -n cicd \
  --from-literal=username=admin \
  --from-literal=token='<REPO_TOKEN>'
```

The update-values Task reads it with:

```
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
```

The developer does not create a PipelineRun manually:

```
git add .
git commit -m "developer change"
git push origin main
```

Watch the run:

```
kubectl logs -f deployment/el-travelportal-github-listener -n cicd
kubectl get pipelineruns -n cicd -w
```

You should see a new run named:

```
travelportal-build-xxxxx
```

The Pipeline then picks BuildKit or Buildpacks depending on whether the repository contains a Dockerfile. When update-values pushes `values.yaml`, the webhook fires again and the CEL filter rejects that commit, so no second run starts.

Final validation checklist

```
kubectl get crd | grep triggers.tekton.dev
kubectl get pods -n tekton-pipelines | grep triggers
kubectl get sa github-trigger-sa -n cicd
kubectl get eventlistener travelportal-github-listener -n cicd
kubectl get svc el-travelportal-github-listener -n cicd
kubectl get endpoints el-travelportal-github-listener -n cicd
kubectl get triggerbinding travelportal-github-binding -n cicd
kubectl get triggertemplate travelportal-github-template -n cicd
kubectl get pipeline travelportal-pipeline-values-update -n cicd
```

Expected:

```
EventListener   → AVAILABLE=True, READY=True
Service         → 8080:31877/TCP
Endpoint        → :8080
Direct POST     → HTTP/1.1 202 Accepted
Webhook target  → http://10.12.92.3:31877
```

| Pipeline triggers itself repeatedly | CI commit message does not match the CEL filter | Commit with `ci: update TravelPortal image digest` and re-apply the EventListener |
