# Webhook implementation on tekton pipeline with buildkit, buildpack and cosign on vks

## Introduction

- **Webhook:** An HTTP call the repository sends to Tekton every time a developer pushes code.

- **Tekton Triggers:** The Tekton component that receives events and creates PipelineRuns automatically.

- **EventListener:** The endpoint that listens for the push event and runs the trigger chain.

- **CEL Interceptor:** A filter on the EventListener. It lets only the main branch through and ignores commits made by the pipeline itself.

- **TriggerBinding / TriggerTemplate:** The Binding reads values from the push payload (repository, branch, commit). The Template uses them to create the PipelineRun.

- **Harbor:** The private registry that stores, scans and serves the signed images.

## How it works

When a developer changes the code:

- The repository sends a **push event** to the EventListener.
- The pipeline **clones the latest repository** and starts the build again.
- **BuildKit** builds the image if a Dockerfile exists, otherwise **Buildpacks** builds it.
- An **SBOM** is generated for the image.
- The image and SBOM are **signed with Cosign**.
- The image is **pushed to Harbor**, which **scans** it and stores the image, signature and SBOM.

## Benefits

| Benefit | In simple words |
|---|---|
| No manual PipelineRun | A push is the only human action. |
| Always the latest code | Every run clones the newest commit of the branch. |
| Every image is signed and scanned | Cosign signs it, Harbor scans it. |
| No build loops | Commits made by the pipeline itself never trigger a new build. |

## Architecture

```
1. Developer pushes code
             │
             ▼
 2. Repository sends webhook (POST)
             │
             ▼
 3. NodePort 10.12.92.3:31877
             │
             ▼
 4. EventListener
    ├── CEL filter: main branch only, ignore CI commits
    ├── TriggerBinding: repo URL, branch, commit
    └── TriggerTemplate: creates the PipelineRun
             │
             ▼
 5. Pipeline clones the latest repository
             │
             ▼
 6. Dockerfile present?
    ┌────────┴────────┐
   Yes                 No
    │                   │
    ▼                   ▼
 BuildKit            Buildpacks
    └────────┬──────────┘
             ▼
 7. SBOM generated (Syft)
             │
             ▼
 8. Image and SBOM signed (Cosign)
             │
             ▼
 9. Image pushed to Harbor
    (Harbor scans, stores image + signature + SBOM)
             │
             ▼
10. values.yaml updated in the repository
    (CI commit is ignored by the CEL filter)
```

| Step | What it means in plain words |
|---|---|
| Webhook | Repository → Tekton call on every push |
| EventListener | Receives the webhook and filters it |
| TriggerBinding | Picks repo URL, branch and commit from the payload |
| TriggerTemplate | Creates the PipelineRun |
| clone | Pulls the latest code |
| detect | Chooses BuildKit or Buildpacks |
| sign | Syft SBOM + Cosign signature and attestation |
| Harbor | Stores and scans the image |
| update-values | Writes the new digest into helm-charts/values.yaml |

## Prerequisites

| Component | What it's for |
|---|---|
| Tekton Pipelines + Dashboard | Already installed (see the main guide) |
| Pipeline `travelportal-pipeline-values-update` | Existing pipeline in namespace `cicd` |
| Secrets `harbor-registry-secret`, `cosign-key` | Harbor login and Cosign keys |
| PVC `buildpacks-source-pvc` | Workspace for the source code |
| Harbor project `cicd` | Image storage, scan on push |
| Network | The repository server must reach the node IP and NodePort |

Lab values:

```
Namespace          : cicd
Pipeline           : travelportal-pipeline-values-update
EventListener      : travelportal-github-listener
EventListener svc  : el-travelportal-github-listener (NodePort)
Webhook port       : 8080 -> 31877
Node IP            : 10.12.92.3
Webhook URL        : http://10.12.92.3:31877
Repository server  : http://10.12.90.62
Repository         : http://10.12.90.62/admin/travelPortal-test-buildpack.git
Branch             : main
```

## Install Tekton Triggers

```
kubectl apply -f https://infra.tekton.dev/tekton-releases/triggers/latest/release.yaml
kubectl apply -f https://infra.tekton.dev/tekton-releases/triggers/latest/interceptors.yaml
```

Verify:

```
kubectl get pods -n tekton-pipelines | grep triggers
kubectl get crd | grep triggers.tekton.dev
```

You should see:

```
tekton-triggers-controller-...          1/1   Running
tekton-triggers-core-interceptors-...   1/1   Running
tekton-triggers-webhook-...             1/1   Running
```

## Create ServiceAccount and RBAC

The EventListener runs with `github-trigger-sa`. It needs to read Triggers resources and create PipelineRuns.

Create github-trigger-rbac.yaml

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: github-trigger-sa
  namespace: cicd
---
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
    verbs: ["get", "list", "watch"]
  - apiGroups: ["tekton.dev"]
    resources: ["pipelineruns"]
    verbs: ["create", "get", "list", "watch"]
---
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
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: github-trigger-cluster-role
rules:
  - apiGroups: ["triggers.tekton.dev"]
    resources:
      - clusterinterceptors
      - clustertriggerbindings
    verbs: ["get", "list", "watch"]
---
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
kubectl apply -f github-trigger-rbac.yaml
```

Verify:

```
kubectl auth can-i list triggertemplates.triggers.tekton.dev \
  --as=system:serviceaccount:cicd:github-trigger-sa -n cicd

kubectl auth can-i list clusterinterceptors.triggers.tekton.dev \
  --as=system:serviceaccount:cicd:github-trigger-sa

kubectl auth can-i create pipelineruns.tekton.dev \
  --as=system:serviceaccount:cicd:github-trigger-sa -n cicd
```

Expected:

```
yes
```

## Create TriggerBinding

Create triggerbinding.yaml

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

```
kubectl apply -f triggerbinding.yaml
```

## Create TriggerTemplate

Create triggertemplate.yaml

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

```
kubectl apply -f triggertemplate.yaml
```

## Create EventListener

Create eventlistener.yaml

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

The second line of the filter is the build-loop protection (see "Pipeline requirements" below).

```
kubectl apply -f eventlistener.yaml
```

Verify:

```
kubectl get eventlistener travelportal-github-listener -n cicd
kubectl get svc el-travelportal-github-listener -n cicd
kubectl get endpoints el-travelportal-github-listener -n cicd
```

Expected:

```
AVAILABLE=True   READY=True
8080:31877/TCP
endpoint on :8080
```

The webhook must use port 8080 (mapped to 31877). Always read the real NodePort from the Service.

## Test the EventListener

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

Expected:

```
HTTP/1.1 202 Accepted
```

Then:

```
kubectl get pipelineruns -n cicd
```

A new `travelportal-build-xxxxx` run should appear.

## Configure the webhook in the repository

Open:

```
Repository -> Settings -> Webhooks -> Add Webhook
```

Use:

```
Target URL   : http://10.12.92.3:31877
Method       : POST
Content type : application/json
Event        : Push events
Branch       : main
```

Save and use **Test Delivery**.

If the repository server rejects the call with:

```
webhook can only call allowed HTTP servers
(check your security.ALLOWED_HOST_LIST setting)
```

add the node IP to the allow list in the `[security]` section of the repository server configuration and restart it:

```
ALLOWED_HOST_LIST = 10.12.90.62,10.12.92.3
```

## Pipeline requirements

The pipeline must clone the latest repository and pass one digest to the sign stage, whichever build ran.

Digest handling:

```
BuildKit    -> result IMAGE_DIGEST
Buildpacks  -> result APP_IMAGE_DIGEST
Both        -> write $(workspaces.source.path)/image-digest
sign task   -> reads image-digest and returns IMAGE_DIGEST
```

The sign task must have the `source` workspace bound:

```
- name: source
  workspace: source
```

The update-values task must:

1. Build the remote from the `REPO_URL` parameter, not a fixed URL:

```
REPO_URL="$(params.REPO_URL)"
REPO_WITHOUT_SCHEME="${REPO_URL#http://}"
REPO_WITHOUT_SCHEME="${REPO_WITHOUT_SCHEME#https://}"

git remote set-url origin \
  "http://${GIT_USERNAME}:${GIT_TOKEN}@${REPO_WITHOUT_SCHEME}"
```

2. Commit with the exact message the EventListener filter ignores:

```
git commit -m "ci: update TravelPortal image digest"
```

3. Never force push. If the push is rejected (fetch first):

```
git fetch origin main
git reset --hard origin/main
```

then re-apply the values.yaml change, commit and push.

Repository token secret:

```
kubectl create secret generic repo-git-credentials \
  -n cicd \
  --from-literal=username=admin \
  --from-literal=token='<REPO_TOKEN>'
```

## Harbor scan on push

Enable automatic scanning for the project:

```
Harbor -> Projects -> cicd -> Configuration
  -> Automatically scan images on push  (enable)
  -> Save
```

Optional: under the same page, enable Cosign signature verification so Harbor only serves signed images.

## Test end to end

Developer:

```
git add .
git commit -m "developer change"
git push origin main
```

Watch:

```
kubectl logs -f deployment/el-travelportal-github-listener -n cicd
kubectl get pipelineruns -n cicd -w
kubectl get taskrun -n cicd
```

Expected order of tasks:

```
clone -> detect -> buildkit OR buildpack -> sign -> update-values
```

When update-values pushes values.yaml, the webhook fires again and the CEL filter rejects that commit, so no second run starts.

## Verify in Harbor

Image, scan result, signature and SBOM:

```
Harbor -> Projects -> cicd -> travelportal
  -> latest
       Vulnerabilities : scan result shown
       Signature       : signed (Cosign)
       Accessories     : SBOM attestation
```

Verify the signature:

```
cosign verify \
  --key cosign.pub \
  lab25-harbor.lab25.sunfire.lab/cicd/travelportal:latest
```

Verify the SBOM attestation:

```
cosign verify-attestation \
  --key cosign.pub \
  --type spdxjson \
  lab25-harbor.lab25.sunfire.lab/cicd/travelportal:latest
```

## Troubleshooting

| Problem | Fix |
|---|---|
| `Connection refused` on webhook | Wrong port. Use the 8080 mapping from `kubectl get svc el-travelportal-github-listener -n cicd` (31877), not 32312 or an old port |
| `webhook can only call allowed HTTP servers` | Add `10.12.92.3` to `ALLOWED_HOST_LIST` on the repository server |
| `github-trigger-sa cannot list ...` | Re-apply github-trigger-rbac.yaml and re-run the `can-i` checks |
| 202 returned but no PipelineRun | `kubectl logs deployment/el-travelportal-github-listener -n cicd`, then check the CEL filter against the real payload |
| `declared workspace "source" is required but has not been bound` | Bind `source` for the sign task in the Pipeline |
| Push rejected (fetch first) | `git fetch origin main` and `git reset --hard origin/main`; never force push |
| Pipeline keeps re-triggering | The CI commit message must start with `ci: update TravelPortal image digest` |
