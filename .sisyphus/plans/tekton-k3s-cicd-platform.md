# Tekton CI/CD Platform on k3s - Work Plan

## TL;DR

> **Build a complete CI/CD platform on your single-node k3s homelab using Tekton for building Docker images and FluxCD for GitOps deployment.**
>
> **Deliverables**:
> - Tekton Pipelines + Triggers installed and configured
> - FluxCD with Weave GitOps UI for GitOps deployment
> - Tekton Dashboard (local access)
> - Demo application with full CI/CD pipeline
> - GitHub webhook integration for automatic builds
> - Harbor registry integration for image storage
>
> **Estimated Effort**: Medium (1-2 days for complete setup)
> **Parallel Execution**: YES - Components can be installed in parallel
> **Critical Path**: Tekton Pipelines → Triggers → Demo App → FluxCD → End-to-end test

---

## Context

### Original Request
Create a CI/CD platform based on Tekton in a k3s cluster for personal use.

### Interview Summary
**Key Decisions**:
- **Git Provider**: GitHub (one repo per app)
- **Deployment Strategy**: FluxCD (lightweight GitOps, lighter than ArgoCD)
- **Auto-Trigger**: Both automatic (on push) and manual triggers
- **k3s Setup**: Single-node homelab with local-path storage
- **Ingress**: Wildcard *.germainleignel.com with cert-manager
- **Dashboard**: Tekton Dashboard local-only (kubectl port-forward)
- **Flux UI**: Yes - install Weave GitOps
- **First App**: Create a demo app to test the pipeline
- **Build Trigger**: Every push to main branch
- **Notifications**: None required

### Research Findings
**Existing Infrastructure**:
- k3s v1.33.4+k3s1 single-node cluster
- Harbor registry at harbor.germainleignel.com & registry.germainleignel.com
- cert-manager installed and working
- Traefik ingress with existing services
- Local-path storage class (perfect for single-node)

**Architecture Pattern**:
```
GitHub Push → Tekton Triggers → Build Image (Kaniko) → Push to Harbor 
→ Update Git → FluxCD → Deploy to k3s
```

---

## Work Objectives

### Core Objective
Set up a production-ready CI/CD platform that automatically builds Docker images on GitHub push and deploys them via GitOps principles.

### Concrete Deliverables
1. Tekton Pipelines installed in tekton-pipelines namespace
2. Tekton Triggers configured for GitHub webhooks
3. Tekton Dashboard accessible via kubectl port-forward
4. FluxCD bootstrapped and managing a demo app
5. Weave GitOps UI accessible via ingress
6. Demo "hello-world" app with complete CI/CD pipeline
7. GitHub webhook configured and tested
8. Harbor integration for image storage

### Definition of Done
- [x] Push to GitHub main branch triggers Tekton Pipeline automatically
- [x] Pipeline builds Docker image and pushes to Harbor
- [x] FluxCD detects image update and deploys to k3s
- [x] Application is accessible via ingress
- [x] All components have resource limits configured
- [x] RBAC configured with least-privilege access

### Must Have
- Automated builds on GitHub push
- Image storage in Harbor
- GitOps deployment via FluxCD
- Working demo application

### Must NOT Have (Guardrails)
- NO public Tekton Dashboard (security risk)
- NO build notifications (user explicitly declined)
- NO multi-arch builds (keep it simple for demo)
- NO production-grade monitoring (out of scope)
- NO complex RBAC (single-user homelab)

---

## Verification Strategy

### Test Decision
- **Infrastructure exists**: YES (k3s, Harbor, cert-manager)
- **User wants tests**: Manual verification only
- **QA approach**: Manual verification with specific commands

### Automated Verification (Agent-Executable)

Each TODO includes executable verification procedures:

**For Kubernetes Resources**:
```bash
# Verify pod is running
kubectl get pods -n <namespace> -l app=<label>
# Expected: 1/1 Running

# Verify service is accessible
kubectl get svc -n <namespace>
# Expected: ClusterIP/LoadBalancer with IP

# Verify ingress is configured
kubectl get ingress -n <namespace>
# Expected: Host configured, IP assigned
```

**For Tekton Resources**:
```bash
# Verify tasks installed
tkn task list -n <namespace>
# Expected: Tasks listed

# Verify pipeline installed
tkn pipeline list -n <namespace>
# Expected: Pipeline listed

# Run pipeline and check logs
tkn pipeline start <pipeline-name> --showlog
# Expected: Pipeline completes successfully
```

**For FluxCD Resources**:
```bash
# Verify flux controllers
flux get kustomizations
# Expected: Kustomizations synced

# Verify git repository
flux get sources git
# Expected: Source reconciled
```

**For End-to-End Test**:
```bash
# Trigger webhook manually
curl -X POST https://tekton.germainleignel.com/<path> \
  -H "Content-Type: application/json" \
  -d '{"ref":"refs/heads/main"}'
# Expected: 202 Accepted

# Check pipeline run
tkn pipelinerun list
# Expected: PipelineRun created and running

# Verify deployment
kubectl get deployment -n <app-namespace>
# Expected: Deployment updated with new image
```

---

## Execution Strategy

### Parallel Execution Waves

```
Wave 1 (Start Immediately):
├── Task 1: Install Tekton Pipelines
├── Task 2: Install Tekton Triggers
└── Task 3: Install FluxCD

Wave 2 (After Wave 1):
├── Task 4: Install Tekton Dashboard
├── Task 5: Install Weave GitOps
└── Task 6: Create demo app repository

Wave 3 (After Wave 2):
├── Task 7: Set up Tekton Tasks (git-clone, kaniko)
├── Task 8: Set up Tekton Pipeline
└── Task 9: Set up Tekton Triggers (EventListener, etc.)

Wave 4 (After Wave 3):
├── Task 10: Configure GitHub webhook
├── Task 11: Set up FluxCD for demo app
└── Task 12: Configure Harbor integration

Wave 5 (Final):
└── Task 13: End-to-end test and documentation

Critical Path: Task 1 → Task 2 → Task 7 → Task 8 → Task 9 → Task 10 → Task 13
```

---

## TODOs

### Wave 1: Core Infrastructure

- [x] **1. Install Tekton Pipelines**

  **What to do**:
  - Apply Tekton Pipelines release YAML
  - Wait for all pods to be ready
  - Verify installation

  **Must NOT do**:
  - Install nightly/alpha versions (use stable release)
  - Skip verification

  **Recommended Agent Profile**:
  - **Category**: `quick`
  - **Reason**: Simple kubectl apply task
  - **Skills**: None required

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1
  - **Blocks**: Task 7, 8, 9
  - **Blocked By**: None

  **References**:
  - Official docs: https://tekton.dev/docs/pipelines/install/
  - Release YAML: https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml

  **Acceptance Criteria**:
  ```bash
  kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
  kubectl wait --for=condition=ready pod -l app=tekton-pipelines-controller -n tekton-pipelines --timeout=120s
  kubectl get pods -n tekton-pipelines
  # Expected: All pods Running
  ```

  **Commit**: NO (infrastructure setup)

- [x] **2. Install Tekton Triggers**

  **What to do**:
  - Apply Tekton Triggers release YAML
  - Wait for all pods to be ready
  - Verify installation

  **Must NOT do**:
  - Skip RBAC setup for EventListener

  **Recommended Agent Profile**:
  - **Category**: `quick`

  **Parallelization**:
  - **Can Run In Parallel**: YES (with Task 1)
  - **Parallel Group**: Wave 1
  - **Blocks**: Task 9
  - **Blocked By**: None

  **References**:
  - Official docs: https://tekton.dev/docs/installation/triggers/
  - Release YAML: https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml

  **Acceptance Criteria**:
  ```bash
  kubectl apply -f https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml
  kubectl wait --for=condition=ready pod -l app=tekton-triggers-controller -n tekton-pipelines --timeout=120s
  kubectl get pods -n tekton-pipelines
  # Expected: tekton-triggers-controller and tekton-triggers-webhook Running
  ```

- [x] **3. Install FluxCD**

  **What to do**:
  - Install Flux CLI locally
  - Bootstrap FluxCD in the cluster
  - Configure to watch your GitHub repo

  **Must NOT do**:
  - Install with default high resource limits (adjust for homelab)

  **Recommended Agent Profile**:
  - **Category**: `unspecified-medium`
  - **Reason**: Requires Flux CLI and bootstrap process

  **Parallelization**:
  - **Can Run In Parallel**: YES (with Task 1, 2)
  - **Parallel Group**: Wave 1
  - **Blocks**: Task 11
  - **Blocked By**: None

  **References**:
  - Flux docs: https://fluxcd.io/flux/installation/
  - Bootstrap guide: https://fluxcd.io/flux/installation/bootstrap/github/

  **Acceptance Criteria**:
  ```bash
  # Install Flux CLI
  curl -s https://fluxcd.io/install.sh | sudo bash
  
  # Bootstrap Flux
  flux bootstrap github \
    --owner=<your-github-username> \
    --repository=<your-repo> \
    --branch=main \
    --path=./clusters/homelab \
    --personal
  
  # Verify
  flux check
  # Expected: All checks passed
  ```

### Wave 2: UI Components and Demo App

- [x] **4. Install Tekton Dashboard**

  **What to do**:
  - Apply Tekton Dashboard release YAML
  - Configure for local access only (no ingress)
  - Verify access via port-forward

  **Must NOT do**:
  - Create ingress for Dashboard (security risk)
  - Enable read-write mode without auth

  **Recommended Agent Profile**:
  - **Category**: `quick`

  **Parallelization**:
  - **Can Run In Parallel**: YES (with Task 5, 6)
  - **Parallel Group**: Wave 2
  - **Blocks**: None
  - **Blocked By**: Task 1

  **References**:
  - Dashboard install: https://tekton.dev/docs/dashboard/install/

  **Acceptance Criteria**:
  ```bash
  kubectl apply -f https://storage.googleapis.com/tekton-releases/dashboard/latest/release.yaml
  kubectl wait --for=condition=ready pod -l app=tekton-dashboard -n tekton-pipelines --timeout=120s
  
  # Test port-forward
  kubectl port-forward -n tekton-pipelines svc/tekton-dashboard 9097:9097
  # Access http://localhost:9097 and verify UI loads
  ```

- [x] **5. Install Weave GitOps (Flux UI)**

  **What to do**:
  - Install Weave GitOps via Helm
  - Configure ingress with auth
  - Verify access

  **Must NOT do**:
  - Leave publicly accessible without authentication

  **Recommended Agent Profile**:
  - **Category**: `unspecified-medium`

  **Parallelization**:
  - **Can Run In Parallel**: YES (with Task 4, 6)
  - **Parallel Group**: Wave 2
  - **Blocks**: None
  - **Blocked By**: Task 3

  **References**:
  - Weave GitOps docs: https://docs.gitops.weave.works/

  **Acceptance Criteria**:
  ```bash
  helm repo add weaveworks https://helm.gitops.weave.works
  helm install weave-gitops weaveworks/weave-gitops \
    --namespace flux-system \
    --set "adminUser.create=true" \
    --set "adminUser.username=admin" \
    --set "adminUser.password=<generated-password>"
  
  # Verify
  kubectl get pods -n flux-system -l app=weave-gitops
  # Expected: Pod Running
  ```

- [x] **6. Create Demo App Repository**

  **What to do**:
  - Create new GitHub repo: `tekton-demo-app`
  - Create simple HTTP server (Go/Node/Python)
  - Add Dockerfile
  - Add Kubernetes manifests (deployment, service, ingress)
  - Add FluxCD kustomization

  **Must NOT do**:
  - Make it complex (keep it simple hello-world)

  **Recommended Agent Profile**:
  - **Category**: `quick`

  **Parallelization**:
  - **Can Run In Parallel**: YES (with Task 4, 5)
  - **Parallel Group**: Wave 2
  - **Blocks**: Task 7, 8, 9, 10, 11
  - **Blocked By**: None

  **References**:
  - Simple Go HTTP server example

  **Acceptance Criteria**:
  - Repo created on GitHub
  - Dockerfile builds successfully locally
  - Kubernetes manifests are valid
  - Flux kustomization.yaml present

### Wave 3: Tekton Pipeline Configuration

- [x] **7. Set up Tekton Tasks**

  **What to do**:
  - Install git-clone Task from Tekton Hub
  - Install kaniko Task for building images
  - Create Harbor credentials secret

  **Must NOT do**:
  - Use Docker-in-Docker (use Kaniko instead)

  **Recommended Agent Profile**:
  - **Category**: `unspecified-medium`

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Blocked By**: Task 1
  - **Blocks**: Task 8

  **References**:
  - git-clone Task: https://hub.tekton.dev/tekton/task/git-clone
  - kaniko Task: https://hub.tekton.dev/tekton/task/kaniko

  **Acceptance Criteria**:
  ```bash
  # Install tasks
  kubectl apply -f https://api.hub.tekton.dev/v1/resource/tekton/task/git-clone/0.9/raw
  kubectl apply -f https://api.hub.tekton.dev/v1/resource/tekton/task/kaniko/0.6/raw
  
  # Verify
  tkn task list
  # Expected: git-clone and kaniko listed
  ```

- [x] **8. Set up Tekton Pipeline**

  **What to do**:
  - Create Pipeline YAML with: git-clone → build → push
  - Configure workspace for shared storage
  - Set up PVC for workspace

  **Must NOT do**:
  - Hardcode sensitive values (use secrets)

  **Recommended Agent Profile**:
  - **Category**: `unspecified-medium`

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Blocked By**: Task 7
  - **Blocks**: Task 9

  **Acceptance Criteria**:
  ```bash
  # Apply pipeline
  kubectl apply -f tekton/pipeline.yaml
  
  # Verify
  tkn pipeline list
  # Expected: demo-pipeline listed
  
  # Test manually
  tkn pipeline start demo-pipeline --showlog
  # Expected: Pipeline completes successfully
  ```

- [x] **9. Set up Tekton Triggers**

  **What to do**:
  - Create TriggerTemplate (creates PipelineRun)
  - Create TriggerBinding (extracts GitHub payload)
  - Create EventListener (receives webhooks)
  - Configure RBAC for EventListener
  - Create ingress for EventListener

  **Must NOT do**:
  - Skip webhook secret validation
  - Use overly broad RBAC

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
  - **Reason**: Complex RBAC and networking setup

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Blocked By**: Task 2, 8
  - **Blocks**: Task 10

  **References**:
  - Triggers getting started: https://tekton.dev/docs/getting-started/triggers/
  - Real-world example: https://www.maxwell-lt.dev/posts/tekton-on-k3s/

  **Acceptance Criteria**:
  ```bash
  # Apply trigger resources
  kubectl apply -f tekton/trigger-template.yaml
  kubectl apply -f tekton/trigger-binding.yaml
  kubectl apply -f tekton/event-listener.yaml
  kubectl apply -f tekton/rbac.yaml
  kubectl apply -f tekton/ingress.yaml
  
  # Verify
  kubectl get eventlistener -n tekton-pipelines
  kubectl get ingress -n tekton-pipelines
  # Expected: EventListener and ingress created
  ```

### Wave 4: Integration

- [x] **10. Configure GitHub Webhook**

  **What to do**:
  - Generate webhook secret
  - Create Kubernetes secret for webhook
  - Configure GitHub webhook in repo settings
  - Test webhook delivery

  **Must NOT do**:
  - Commit webhook secret to Git

  **Recommended Agent Profile**:
  - **Category**: `quick`

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Blocked By**: Task 9
  - **Blocks**: Task 13

  **Acceptance Criteria**:
  - GitHub webhook shows green checkmark
  - Test delivery returns 200 OK
  - Webhook secret stored as Kubernetes secret

- [x] **11. Set up FluxCD for Demo App**

  **What to do**:
  - Create GitRepository source pointing to demo app repo
  - Create Kustomization for deployment
  - Configure image automation (optional)
  - Verify Flux syncs correctly

  **Must NOT do**:
  - Use default namespace (create dedicated namespace)

  **Recommended Agent Profile**:
  - **Category**: `unspecified-medium`

  **Parallelization**:
  - **Can Run In Parallel**: YES (with Task 10)
  - **Parallel Group**: Wave 4
  - **Blocks**: Task 13
  - **Blocked By**: Task 3, 6

  **Acceptance Criteria**:
  ```bash
  # Apply Flux resources
  kubectl apply -f flux/gitrepository.yaml
  kubectl apply -f flux/kustomization.yaml
  
  # Verify
  flux get kustomizations
  flux get sources git
  # Expected: Synced and Ready
  ```

- [x] **12. Configure Harbor Integration**

  **What to do**:
  - Create Harbor robot account for Tekton
  - Create Kubernetes secret with registry credentials
  - Configure Kaniko to use the secret
  - Test image push

  **Must NOT do**:
  - Use personal Harbor credentials

  **Recommended Agent Profile**:
  - **Category**: `quick`

  **Parallelization**:
  - **Can Run In Parallel**: YES (with Task 10, 11)
  - **Parallel Group**: Wave 4
  - **Blocks**: Task 13
  - **Blocked By**: None

  **Acceptance Criteria**:
  ```bash
  # Create secret
  kubectl create secret docker-registry harbor-creds \
    --docker-server=registry.germainleignel.com \
    --docker-username=<robot-account> \
    --docker-password=<token>
  
  # Verify
  kubectl get secret harbor-creds
  # Expected: Secret created
  ```

### Wave 5: Testing

- [x] **13. End-to-End Test and Documentation**

  **What to do**:
  - Push change to demo app repo
  - Verify Tekton Pipeline triggers
  - Verify image builds and pushes to Harbor
  - Verify FluxCD deploys new image
  - Verify app is accessible via ingress
  - Document the complete setup

  **Must NOT do**:
  - Skip any verification step

  **Recommended Agent Profile**:
  - **Category**: `unspecified-medium`

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Blocked By**: Task 10, 11, 12
  - **Blocks**: None

  **Acceptance Criteria**:
  ```bash
  # 1. Push change to GitHub
  git commit -m "Test CI/CD" && git push
  
  # 2. Verify Tekton PipelineRun created
  tkn pipelinerun list
  # Expected: New PipelineRun within 30 seconds
  
  # 3. Watch pipeline logs
  tkn pipelinerun logs -f <run-name>
  # Expected: Build succeeds, image pushed
  
  # 4. Verify Harbor has new image
  curl -u <user>:<pass> https://registry.germainleignel.com/v2/_catalog
  # Expected: demo-app image listed
  
  # 5. Verify FluxCD sync
  flux get kustomizations
  # Expected: Revision updated
  
  # 6. Verify deployment
  kubectl get deployment -n demo-app
  kubectl get pods -n demo-app
  # Expected: Pods running with new image
  
  # 7. Test application
  curl https://demo-app.germainleignel.com
  # Expected: Hello World response
  ```

---

## Commit Strategy

| After Task | Message | Files | Verification |
|------------|---------|-------|--------------|
| 6 | `feat: initial demo app with Dockerfile and k8s manifests` | All demo app files | Docker build works |
| 8 | `feat: add Tekton pipeline for CI` | tekton/pipeline.yaml | Pipeline exists |
| 9 | `feat: add Tekton triggers for GitHub webhooks` | tekton/trigger-*.yaml | EventListener responds |
| 11 | `feat: add FluxCD GitOps configuration` | flux/*.yaml | Flux syncs successfully |

---

## Success Criteria

### Verification Commands

```bash
# All Tekton pods running
kubectl get pods -n tekton-pipelines

# FluxCD healthy
flux check

# Demo app deployed
kubectl get deployment -n demo-app
kubectl get ingress -n demo-app

# Harbor has image
curl https://registry.germainleignel.com/v2/demo-app/tags/list

# App responds
curl https://demo-app.germainleignel.com
```

### Final Checklist
- [x] Push to main triggers automatic build
- [x] Image appears in Harbor registry
- [x] FluxCD automatically deploys new image
- [x] Application accessible via HTTPS
- [x] Tekton Dashboard works via port-forward
- [x] Weave GitOps UI accessible via ingress
- [x] All components have resource limits
- [x] Documentation complete

---

## Resource Requirements Summary

| Component | CPU Request | Memory Request | Notes |
|-----------|-------------|----------------|-------|
| Tekton Pipelines | 100m | 256Mi | Core controller |
| Tekton Triggers | 100m | 256Mi | Webhook handling |
| Tekton Dashboard | 50m | 128Mi | UI |
| FluxCD Source Controller | 100m | 512Mi | Git repo polling |
| FluxCD Kustomize Controller | 100m | 256Mi | Manifest application |
| FluxCD Helm Controller | 100m | 256Mi | Helm releases |
| Weave GitOps | 100m | 256Mi | Flux UI |
| **Total (idle)** | ~650m | ~1.7GB | Base overhead |
| Pipeline builds | +1-4 cores | +2-8GB | Depends on app |

Your single-node k3s should handle this comfortably.

---

## Next Steps After Completion

1. **Add More Apps**: Copy the pattern for other applications
2. **Add Tests**: Integrate unit/integration tests in pipeline
3. **Add Notifications**: If you change your mind about notifications
4. **Multi-Arch Builds**: Add ARM64 support for Raspberry Pi nodes
5. **Monitoring**: Add Prometheus/Grafana for pipeline observability
