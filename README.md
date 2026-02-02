# tekton (k3s homelab CI/CD)

This repo contains the manifests + runbook for a personal CI/CD setup on k3s:

- CI: Tekton (Pipelines + Triggers)
- CD: FluxCD (GitRepository + Kustomization) + Flux image automation
- Registry: Harbor (`registry.germainleignel.com/personal`)

See: `docs/tekton-k3s-cicd-runbook.md`

## TL;DR

k3s cluster with Tekton building images on Git push, FluxCD scanning the registry and auto-deploying.

**Flow:** GitHub push -> Tekton builds/pushes image -> Flux detects new tag -> updates Deployment -> applies to cluster.

**Key URLs:**
- Tekton webhook: `https://tekton.germainleignel.com/github`
- GitOps UI: `https://gitops.germainleignel.com/`
- Demo app: `https://demo-app.germainleignel.com/`

**Quick verify:**
```bash
kubectl get pods -n tekton-pipelines
kubectl get eventlistener -n tekton-pipelines
tkn pipelinerun list -n tekton-pipelines
flux check
```

## For Humans

### Prerequisites

This repo assumes a working k3s cluster with the following already in place:

- k3s running (single-node or multi-node)
- Tekton Pipelines + Triggers installed
- FluxCD (source, kustomize, image-automation controllers) installed
- Harbor registry reachable at `registry.germainleignel.com`
- DNS records and Ingress + TLS configured for `*.germainleignel.com`

For full setup steps, see: `docs/tekton-k3s-cicd-runbook.md`

### Quick Start

Verify the stack is healthy:

```bash
# Tekton
kubectl get pods -n tekton-pipelines
tkn pipelinerun list -n tekton-pipelines
kubectl get eventlistener -n tekton-pipelines

# Flux
flux check
flux get kustomizations -n flux-system

# Demo app (expect 401 without creds)
curl -I https://demo-app.germainleignel.com/
```

Key directories:
- `tekton/` - Pipeline and manual run manifests
- `tekton/triggers/` - EventListener, TriggerTemplate, TriggerBinding, Ingress
- `flux/` - GitRepository, Kustomization, ImageRepository, ImagePolicy, ImageUpdateAutomation
- `tekton-demo-app/` - Example app repo structure (see https://github.com/Germain-L/tekton-demo-app)

### Architecture

Flow:

1) GitHub push to `main`
2) GitHub webhook -> Tekton Triggers EventListener
3) Tekton PipelineRun:
   - clone repo
   - build/push with Kaniko
4) Flux:
   - scans registry tags
   - updates {{APP_NAME}}/k8s/deployment.yaml image tag in the demo repo
   - applies Kustomize manifests to the cluster

### Repos

- Demo app repo: https://github.com/Germain-L/tekton-demo-app
- This repo: cluster-side YAML and operational notes

### URLs

- Tekton webhook endpoint: `https://tekton.germainleignel.com/github`
- Weave GitOps UI: `https://gitops.germainleignel.com/` (basic auth)
- Demo app: `https://demo-app.germainleignel.com/` (basic auth)

### Files

- Tekton pipeline + manual run: `tekton/`
- Tekton triggers resources: `tekton/triggers/`
- Flux CD resources: `flux/`
- Runbook: `docs/tekton-k3s-cicd-runbook.md`

### Cluster Objects (namespaces)

- `tekton-pipelines`: Tekton controllers, pipeline, triggers, EventListener
- `flux-system`: Flux controllers, Weave GitOps, image automation resources
- `demo-app`: demo workload

### Secrets (names only)

No secrets are stored in this repo.

- `tekton-pipelines/harbor-creds`: Kaniko docker config mounted at `/kaniko/.docker/config.json`
- `tekton-pipelines/github-webhook-secret`: GitHub webhook signature secret (key `secretToken`)
- `flux-system/demo-app-git`: Git credentials for Flux to push automation commits
- `flux-system/harbor-registry`: Registry credentials for Flux ImageRepository scans
- `demo-app/regcred`: imagePullSecret for Harbor

### Verify

```bash
# Tekton
kubectl get pods -n tekton-pipelines
kubectl get pipeline -n tekton-pipelines
kubectl get eventlistener -n tekton-pipelines
tkn pipelinerun list -n tekton-pipelines

# Flux
flux check
flux get sources git -n flux-system
flux get kustomizations -n flux-system
kubectl get imagerepository,imagepolicy,imageupdateautomation -n flux-system

# Demo app (expect 401 without creds)
curl -I https://demo-app.germainleignel.com/
```

### Notes

- Tekton Triggers needs both release.yaml and interceptors.yaml (otherwise github/cel interceptors will fail).
- If TaskRuns are blocked by PodSecurity, check the `tekton-pipelines` namespace PSA labels.

## For LLM Agents

### Pre-flight Checks

Run these commands to verify the cluster is ready for onboarding:

```bash
# Verify namespaces exist
kubectl get namespace tekton-pipelines flux-system

# Verify Tekton EventListener exists
kubectl get eventlistener -n tekton-pipelines

# Verify Flux controllers are healthy
flux check

# Optional: verify required secrets exist (names only)
kubectl get secret harbor-creds -n tekton-pipelines
kubectl get secret github-webhook-secret -n tekton-pipelines
kubectl get secret harbor-registry -n flux-system

# Optional: confirm demo patterns exist in this repo
ls -la tekton/ tekton/triggers/ flux/
```

### Guardrails (MUST / MUST NOT)

**Security:**
- DO NOT commit secrets to this repo; do not print secret values in logs or outputs
- DO NOT expose the Tekton Dashboard publicly without authentication
- Validate webhook signatures using the GitHub interceptor `secretRef` (see Step 3e)

**Scope:**
- DO NOT change cluster-wide components (Tekton/Flux installs, CRDs, controllers)
- DO NOT modify other apps' resources (namespaces, deployments, services not owned by the current app)
- Work only within the designated app namespace (`{{NAMESPACE}}`)

**Best Practices:**
- Prefer Tekton Catalog raw URLs over Tekton Hub (Hub is deprecated, shutting down in 2026)
- Use secret names only in documentation; never include actual secret values or base64-encoded data
- Verify changes with `kubectl diff` before applying when possible

### Step 0: Gather Inputs

Before onboarding a new app, ask for these inputs:

| Variable | Question | Default |
|----------|----------|---------|
| `APP_NAME` | What is the application name? (DNS label format) | (required) |
| `GITHUB_REPO_EXISTS` | Does the GitHub repo already exist? | `no` |
| `GITHUB_USER` | GitHub username or organization? | (required) |
| `REGISTRY_PROJECT` | Harbor registry project path? | `personal` |
| `DOMAIN` | Base domain for endpoints? | `germainleignel.com` |
| `NAMESPACE` | Kubernetes namespace for the app? | `{{APP_NAME}}` |

**Validation snippet:**

```bash
# Validate APP_NAME is a valid DNS label
if [[ ! "$APP_NAME" =~ ^[a-z0-9-]+$ ]]; then
  echo "ERROR: APP_NAME must match ^[a-z0-9-]+$"
  exit 1
fi

# Validate required vars are non-empty
for var in APP_NAME GITHUB_USER; do
  if [ -z "${!var}" ]; then
    echo "ERROR: $var is required"
    exit 1
  fi
done
```

### Template Variables

When onboarding a new app, replace these:
- `{{APP_NAME}}`: Name of the application (e.g., `demo-app`)
- `{{GITHUB_USER}}`: GitHub username or org (e.g., `Germain-L`)
- `{{REGISTRY_PROJECT}}`: Harbor project path (e.g., `personal`)
- `{{DOMAIN}}`: Base domain for endpoints (e.g., `germainleignel.com`)
- `{{NAMESPACE}}`: Kubernetes namespace for the app (e.g., `demo-app`)

### Decision/Inputs Block

**Goal:** Onboard a new app to this CI/CD pipeline.

**Inputs needed:**
- App name (used for repo, image, namespace)
- GitHub repo URL
- Registry path
- Domain/subdomain for the app

**Key data points:**
- Webhook URL: `https://tekton.{{DOMAIN}}/github`
- Webhook secret: `tekton-pipelines/github-webhook-secret`
- Registry: `registry.{{DOMAIN}}/{{REGISTRY_PROJECT}}/{{APP_NAME}}`
- Flux image policy marker: `# {"$imagepolicy":"flux-system:{{APP_NAME}}"}`
- Required secrets (names only): `tekton-pipelines/harbor-creds`, `tekton-pipelines/github-webhook-secret`, `flux-system/{{APP_NAME}}-git`, `flux-system/harbor-registry`, `{{NAMESPACE}}/regcred`

**Reference files:** `tekton/`, `tekton/triggers/`, `flux/`, `docs/tekton-k3s-cicd-runbook.md`

**Verification commands:** `kubectl`, `tkn`, `flux`, `curl` (see Verify section above)

```text
To onboard a new app: create a new GitHub repo with a Dockerfile + k8s/ (Deployment/Service/Ingress) that references registry.germainleignel.com/personal/<app> and includes the Flux marker comment # {"$imagepolicy":"flux-system:<app>"} in k8s/deployment.yaml; copy/adapt this repo's tekton/ + tekton/triggers/ (pipeline + EventListener/TriggerTemplate/TriggerBinding/Ingress) and set the GitHub webhook URL to https://tekton.germainleignel.com/github with secret tekton-pipelines/github-webhook-secret, wire GitOps + image automation from flux/ (GitRepository/Kustomization/ImageRepository/ImagePolicy/ImageUpdateAutomation) with secrets flux-system/demo-app-git + flux-system/harbor-registry and demo-app/regcred + tekton-pipelines/harbor-creds (names only), then verify with kubectl + tkn + flux + curl; see docs/tekton-k3s-cicd-runbook.md.
```

### Step 1: GitHub Repo

Create or use an existing GitHub repository for the application.

**Option A: New repo from scratch**

```bash
# Create local directory structure
mkdir {{APP_NAME}}
cd {{APP_NAME}}
git init

# Create initial files (see Step 2 for layout)
# Then create remote repo and push
git remote add origin https://github.com/{{GITHUB_USER}}/{{APP_NAME}}.git
git add .
git commit -m "Initial commit"
git push -u origin main
```

Optional (requires `gh` CLI):
```bash
gh repo create {{GITHUB_USER}}/{{APP_NAME}} --public --source=. --push
```

**Option B: Existing repo integration**

If the repo already exists, ensure it has the required structure (Dockerfile + k8s/):

```bash
# Clone existing repo
git clone https://github.com/{{GITHUB_USER}}/{{APP_NAME}}.git
cd {{APP_NAME}}

# Verify main branch exists
 git branch -r | grep origin/main
```

### Step 2: App Repo Layout (Dockerfile + k8s/)

The application repository must contain:

**Required files:**
```
{{APP_NAME}}/
├── Dockerfile
├── k8s/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
└── (application source code)
```

**Dockerfile:** Standard container build. Kaniko will build and push to the registry.

**k8s/deployment.yaml:** Must include the Flux image policy marker comment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{APP_NAME}}
  namespace: {{NAMESPACE}}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: {{APP_NAME}}
  template:
    metadata:
      labels:
        app: {{APP_NAME}}
    spec:
      containers:
      - name: {{APP_NAME}}
        image: registry.{{DOMAIN}}/{{REGISTRY_PROJECT}}/{{APP_NAME}}:PLACEHOLDER # {"$imagepolicy": "flux-system:{{APP_NAME}}"}
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
```

**k8s/service.yaml:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{APP_NAME}}
  namespace: {{NAMESPACE}}
spec:
  selector:
    app: {{APP_NAME}}
  ports:
  - port: 80
    targetPort: 8080
```

**k8s/ingress.yaml:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{APP_NAME}}
  namespace: {{NAMESPACE}}
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - {{APP_NAME}}.{{DOMAIN}}
      secretName: {{APP_NAME}}-tls
  rules:
    - host: {{APP_NAME}}.{{DOMAIN}}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{APP_NAME}}
                port:
                  number: 80
```

### Step 3: Tekton (Pipeline + Triggers)

**Note:** Tekton Hub is deprecated (shutting down in 2026). Use Tekton Catalog raw URLs instead.

**Step 3a: Install required Tasks from Tekton Catalog**

```bash
# Install git-clone task (v0.9)
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-clone/0.9/git-clone.yaml

# Install kaniko task (v0.7)
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/kaniko/0.7/kaniko.yaml
```

**Step 3b: Create Pipeline**

Copy `tekton/pipeline.yaml` to a new file and replace:
- `demo-app-pipeline` -> `{{APP_NAME}}-pipeline`
- `registry.germainleignel.com/personal/demo-app` -> `registry.{{DOMAIN}}/{{REGISTRY_PROJECT}}/{{APP_NAME}}`

Key parameters from the pipeline:
- `repo-url`: Git repository URL (required)
- `revision`: Git revision, default `main`
- `image-repo`: Image repository, default `registry.{{DOMAIN}}/{{REGISTRY_PROJECT}}/{{APP_NAME}}`

The pipeline pushes two tags:
- `$(params.image-repo):$(tasks.git-clone.results.commit)` (git SHA)
- `$(params.image-repo):latest`

**Step 3c: Create TriggerTemplate**

Copy `tekton/triggers/trigger-template.yaml` and replace:
- `demo-app-template` -> `{{APP_NAME}}-template`
- `demo-app-pipeline` -> `{{APP_NAME}}-pipeline`
- `demo-app-` -> `{{APP_NAME}}-` (generateName prefix)
- `registry.germainleignel.com/personal/demo-app` -> `registry.{{DOMAIN}}/{{REGISTRY_PROJECT}}/{{APP_NAME}}`
- `harbor-creds` -> your dockerconfig secret name (if different)

Note: The TriggerTemplate uses a `volumeClaimTemplate` for the `source` workspace and mounts `harbor-creds` secret for the `dockerconfig` workspace.

**Step 3d: Create TriggerBinding**

Create `{{APP_NAME}}-binding` that maps webhook payload to pipeline params:

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
  name: {{APP_NAME}}-binding
  namespace: tekton-pipelines
spec:
  params:
    - name: repo-url
      value: $(body.repository.clone_url)
    - name: revision
      value: $(body.head_commit.id)
    - name: image-repo
      value: registry.{{DOMAIN}}/{{REGISTRY_PROJECT}}/{{APP_NAME}}
```

**Step 3e: EventListener Strategy**

Choose one approach:

**Option A: Add trigger to existing EventListener**

If using the existing `demo-app-listener`, add a new trigger entry with a CEL filter on the repository:

```yaml
# Add to existing EventListener spec.triggers list
triggers:
  - name: {{APP_NAME}}-push-main
    interceptors:
      - ref:
          name: github
          kind: ClusterInterceptor
        params:
          - name: secretRef
            value:
              secretName: github-webhook-secret
              secretKey: secretToken
          - name: eventTypes
            value: [push]
      - ref:
          name: cel
          kind: ClusterInterceptor
        params:
          - name: filter
            value: "body.ref == 'refs/heads/main' && body.repository.full_name == '{{GITHUB_USER}}/{{APP_NAME}}'"
    bindings:
      - ref: {{APP_NAME}}-binding
    template:
      ref: {{APP_NAME}}-template
```

**Option B: Create dedicated EventListener per app**

Copy `tekton/triggers/event-listener.yaml` and `tekton/triggers/ingress.yaml`, then replace:
- `demo-app-listener` -> `{{APP_NAME}}-listener`
- Update Ingress host (e.g., `tekton-{{APP_NAME}}.{{DOMAIN}}`)
- Update service name to `el-{{APP_NAME}}-listener`

**Step 3f: Apply Tekton resources**

```bash
kubectl apply -f {{APP_NAME}}-pipeline.yaml
kubectl apply -f {{APP_NAME}}-binding.yaml
kubectl apply -f {{APP_NAME}}-template.yaml

# For Option A (add to existing listener)
kubectl apply -f updated-event-listener.yaml

# For Option B (dedicated listener)
kubectl apply -f {{APP_NAME}}-listener.yaml
kubectl apply -f {{APP_NAME}}-ingress.yaml
```

### Step 4: Flux (GitOps + Image Automation)

**Step 4a: Create GitRepository source**

Copy `flux/demo-app-source.yaml` and replace:
- `demo-app` -> `{{APP_NAME}}`
- `https://github.com/Germain-L/tekton-demo-app.git` -> `https://github.com/{{GITHUB_USER}}/{{APP_NAME}}.git`
- `demo-app-git` -> `{{APP_NAME}}-git` (secretRef name)

**Step 4b: Create Kustomization**

Create `{{APP_NAME}}-kustomization.yaml`:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: {{APP_NAME}}
  namespace: flux-system
spec:
  interval: 10m
  path: ./k8s
  prune: true
  sourceRef:
    kind: GitRepository
    name: {{APP_NAME}}
  targetNamespace: {{NAMESPACE}}
```

**Step 4c: Create Image Automation resources**

Copy `flux/demo-app-image-automation.yaml` and replace:
- `demo-app` -> `{{APP_NAME}}` (all occurrences)
- `registry.germainleignel.com/personal/demo-app` -> `registry.{{DOMAIN}}/{{REGISTRY_PROJECT}}/{{APP_NAME}}`
- `harbor-registry` -> your registry secret name (if different)

Key values from the demo:
- `filterTags.pattern: '^[0-9a-f]{7,40}$'` - matches git SHA tags
- `policy.alphabetical.order: asc` - alphabetical ascending (repo pushes git SHA tags)
- `update.strategy: Setters` - uses Flux setter markers
- `update.path: ./k8s` - path within repo to update

**Step 4d: Create namespace and imagePullSecret**

```bash
# Create namespace
kubectl create namespace {{NAMESPACE}}

# Create imagePullSecret for Harbor (if not using cluster-wide auth)
kubectl create secret docker-registry regcred \
  --docker-server=registry.{{DOMAIN}} \
  --docker-username=<username> \
  --docker-password=<password> \
  -n {{NAMESPACE}}
```

**Step 4e: Apply Flux resources**

```bash
kubectl apply -f {{APP_NAME}}-source.yaml
kubectl apply -f {{APP_NAME}}-kustomization.yaml
kubectl apply -f {{APP_NAME}}-image-automation.yaml
```

### Step 5: GitHub Webhook

Configure GitHub to send webhook events to Tekton.

**Get the webhook URL:**

For Option A (shared EventListener):
```
https://tekton.{{DOMAIN}}/github
```

For Option B (dedicated EventListener):
```
https://tekton-{{APP_NAME}}.{{DOMAIN}}/github
```

**Configure in GitHub:**

1. Go to GitHub repo -> Settings -> Webhooks -> Add webhook
2. **Payload URL:** `https://tekton.{{DOMAIN}}/github` (or your dedicated URL)
3. **Content type:** `application/json`
4. **Secret:** Create a secret and store it in Kubernetes:
   ```bash
   kubectl create secret generic github-webhook-secret \
     --from-literal=secretToken=<your-secret> \
     -n tekton-pipelines
   ```
5. **Events:** Select "Just the push event"
6. **Active:** Check the box

**Verify webhook delivery:**

After pushing to main, check the webhook deliveries in GitHub (repo -> Settings -> Webhooks -> Recent Deliveries).

### Step 6: End-to-End Verify

**Step 6a: Trigger a build**

```bash
# Push a change to main
cd {{APP_NAME}}
echo "# Test" >> README.md
git add .
git commit -m "Test build"
git push origin main
```

**Step 6b: Verify Tekton PipelineRun**

```bash
# List PipelineRuns
tkn pipelinerun list -n tekton-pipelines

# Watch logs
tkn pipelinerun logs -f {{APP_NAME}}-<generated-id> -n tekton-pipelines

# Check PipelineRun status
kubectl get pipelinerun -n tekton-pipelines -l triggers.tekton.dev/eventlistener={{APP_NAME}}-listener
```

**Step 6c: Verify image pushed to registry**

```bash
# The pipeline pushes tags:
# - registry.{{DOMAIN}}/{{REGISTRY_PROJECT}}/{{APP_NAME}}:<git-sha>
# - registry.{{DOMAIN}}/{{REGISTRY_PROJECT}}/{{APP_NAME}}:latest

# Check in Harbor UI or pull locally
docker pull registry.{{DOMAIN}}/{{REGISTRY_PROJECT}}/{{APP_NAME}}:latest
```

**Step 6d: Verify Flux image automation**

```bash
# Check ImageRepository status
kubectl get imagerepository {{APP_NAME}} -n flux-system

# Check ImagePolicy status
kubectl get imagepolicy {{APP_NAME}} -n flux-system

# Check ImageUpdateAutomation status
kubectl get imageupdateautomation {{APP_NAME}} -n flux-system

# Force a reconciliation
flux reconcile image repository {{APP_NAME}} -n flux-system
flux reconcile image policy {{APP_NAME}} -n flux-system
```

**Step 6e: Verify Git commit from Flux**

```bash
# Pull latest to see if Flux updated the image tag
git pull origin main
grep "image:" {{APP_NAME}}/k8s/deployment.yaml
# Should show the new git SHA tag with the Flux marker preserved
```

**Step 6f: Verify deployment in cluster**

```bash
# Check pods
kubectl get pods -n {{NAMESPACE}}

# Check deployment image
kubectl get deployment {{APP_NAME}} -n {{NAMESPACE}} -o jsonpath='{.spec.template.spec.containers[0].image}'

# Check service
kubectl get svc -n {{NAMESPACE}}

# Check ingress
kubectl get ingress -n {{NAMESPACE}}

# Test app endpoint (expect 401 if basic auth enabled)
curl -I https://{{APP_NAME}}.{{DOMAIN}}/
```

**Step 6g: Full flow verification**

```bash
# All-in-one status check
echo "=== Tekton ==="
kubectl get pipelinerun -n tekton-pipelines | grep {{APP_NAME}}

echo "=== Flux Sources ==="
flux get sources git -n flux-system | grep {{APP_NAME}}

echo "=== Flux Kustomizations ==="
flux get kustomizations -n flux-system | grep {{APP_NAME}}

echo "=== Flux Images ==="
kubectl get imagerepository,imagepolicy,imageupdateautomation -n flux-system | grep {{APP_NAME}}

echo "=== App Pods ==="
kubectl get pods -n {{NAMESPACE}}

echo "=== App Endpoint ==="
curl -s -o /dev/null -w "%{http_code}" https://{{APP_NAME}}.{{DOMAIN}}/
```

**Expected flow:**
1. Git push triggers GitHub webhook
2. Tekton EventListener receives payload, validates signature
3. TriggerTemplate creates PipelineRun
4. Pipeline clones repo, builds image with Kaniko, pushes to Harbor
5. Flux ImageRepository detects new tag in registry
6. Flux ImagePolicy selects latest tag (alphabetical asc on git SHA)
7. Flux ImageUpdateAutomation commits tag update to Git
8. Flux Kustomization applies updated deployment to cluster
9. Kubernetes pulls new image and rolls out deployment

### Common Failures + Fixes

**EventListener rejects webhook (bad secretToken):**
```bash
# Check secret exists with correct key (shows key names only, not values)
kubectl get secret github-webhook-secret -n tekton-pipelines -o go-template='{{range $k,$v := .data}}{{printf "%s\n" $k}}{{end}}'
# Verify GitHub webhook secret matches the value in the secret
# Describe EventListener for errors
kubectl describe eventlistener demo-app-listener -n tekton-pipelines
```

**PipelineRun fails in kaniko auth:**
```bash
# Ensure harbor-creds secret exists in tekton-pipelines namespace
kubectl get secret harbor-creds -n tekton-pipelines
# Check the secret is mounted in the PipelineRun workspace as dockerconfig
# Verify the secret contains valid Harbor credentials
```

**Flux image automation not updating:**
```bash
# Check ImageRepository status
kubectl get imagerepository {{APP_NAME}} -n flux-system
# Check ImagePolicy status
kubectl get imagepolicy {{APP_NAME}} -n flux-system
# Check ImageUpdateAutomation status
kubectl get imageupdateautomation {{APP_NAME}} -n flux-system
# Verify the Flux marker comment exists in {{APP_NAME}}/k8s/deployment.yaml
# Force reconciliation: flux reconcile image repository {{APP_NAME}} -n flux-system
```

**Flux can't push to Git:**
```bash
# Check GitRepository secretRef exists and has valid credentials
kubectl get secret {{APP_NAME}}-git -n flux-system
# Verify the secret contains a valid GitHub token with repo write access
# Check Flux source status: flux get sources git -n flux-system
```

**Interceptors crashloop:**
```bash
# Ensure interceptors are installed (Tekton Triggers needs both release.yaml and interceptors.yaml)
kubectl get pods -n tekton-pipelines | grep interceptor
# See docs/tekton-k3s-cicd-runbook.md for interceptor troubleshooting
```

### Rollback

To safely remove an app and its resources:

```bash
# Delete Flux resources for the app
kubectl delete imageupdateautomation {{APP_NAME}} -n flux-system
kubectl delete imagepolicy {{APP_NAME}} -n flux-system
kubectl delete imagerepository {{APP_NAME}} -n flux-system
kubectl delete kustomization {{APP_NAME}} -n flux-system
kubectl delete gitrepository {{APP_NAME}} -n flux-system

# Delete Tekton resources for the app
kubectl delete triggerbinding {{APP_NAME}}-binding -n tekton-pipelines
kubectl delete triggertemplate {{APP_NAME}}-template -n tekton-pipelines
kubectl delete pipeline {{APP_NAME}}-pipeline -n tekton-pipelines
# If using a dedicated EventListener:
kubectl delete eventlistener {{APP_NAME}}-listener -n tekton-pipelines

# Delete the app namespace (removes deployment, service, ingress, secrets)
kubectl delete namespace {{NAMESPACE}}
```

**Note:** This deletes only the app-specific resources you created. It does NOT affect cluster-wide components (Tekton Pipelines, Flux controllers, or other apps).
