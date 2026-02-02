# Learnings

This file records patterns, conventions, and successful approaches discovered during work on the llm-optimized-readme task.

## 2026-02-02T10:13:28Z Task: init-notepad

Initialized notepad for llm-optimized-readme. This file will capture successful patterns, useful techniques, and lessons learned that can be applied to future tasks.
## 2026-02-02T10:13:28Z Task: explore-existing-docs
Found onboarding references in:
- /home/gmn/projects/tekton/README.md
- /home/gmn/projects/tekton/docs/tekton-k3s-cicd-runbook.md

- Key snippets captured: onboarding instruction in README; verification commands in README; additional verify notes in runbook.
- /home/gmn/projects/tekton/README.md (onboarding + verify)

## 2026-02-02T10:13:28Z Task: librarian-docs

### Authoritative Documentation References

#### Tekton Triggers (GitHub Interceptor)

1. **Tekton Triggers Interceptors Documentation**
   - URL: https://tekton.dev/docs/triggers/interceptors/
   - Purpose: Official documentation for all interceptor types including GitHub, GitLab, Bitbucket, CEL
   - Key features: webhook validation, event filtering, payload transformation

2. **Tekton Triggers Installation Guide**
   - URL: https://tekton.dev/docs/triggers/install/
   - Purpose: How to install Tekton Triggers with interceptors
   - Includes latest release commands and monitoring

3. **Getting Started with Triggers**
   - URL: https://tekton.dev/docs/getting-started/triggers/
   - Purpose: Tutorial for creating first TriggerTemplate, TriggerBinding, EventListener
   - Minikube-based setup examples

#### Tekton Catalog Tasks (git-clone, kaniko)

4. **Tekton Catalog Repository**
   - URL: https://github.com/tektoncd/catalog
   - Purpose: Official catalog of shared Tasks and Pipelines
   - Note: Tekton Hub is deprecated and shutting down in 2026, use Catalog instead

5. **git-clone Task (v0.9)**
   - Raw URL: https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-clone/0.9/git-clone.yaml
   - Purpose: Clone git repositories with support for SSH, basic auth, sparse checkout, submodules
   - Workspaces: output (required), ssh-directory (optional), basic-auth (optional), ssl-ca-directory (optional)
   - Parameters: url (required), revision, refspec, depth, submodules, subdirectory, etc.

6. **kaniko Task (v0.7)**
   - Raw URL: https://raw.githubusercontent.com/tektoncd/catalog/main/task/kaniko/0.7/kaniko.yaml
   - Purpose: Build and push container images without Docker daemon
   - Workspaces: source (required), dockerconfig (optional for registry auth)
   - Results: IMAGE_DIGEST, IMAGE_URL (needed for Tekton Chains signing)

7. **Clone Repository How-to Guide**
   - URL: https://tekton.dev/docs/how-to-guides/clone-repository/
   - Purpose: Step-by-step guide for using git-clone Task in pipelines
   - Shows workspace and parameter configuration

8. **Artifact Hub - git-clone Task**
   - URL: https://artifacthub.io/packages/tekton-task/tekton-catalog-tasks/git-clone/0.9.0
   - Purpose: Package listing with parameters, workspaces, results documentation

9. **Artifact Hub - kaniko Task**
   - URL: https://artifacthub.io/packages/tekton-task/tekton-catalog-tasks/kaniko
   - Purpose: Package listing with authentication requirements and build context setup

#### Flux Image Automation Markers

10. **Flux Image Automation Guide**
    - URL: https://fluxcd.io/flux/guides/image-update/
    - Purpose: Complete guide for automated container image updates to Git
    - Shows ImageRepository, ImagePolicy, ImageUpdateAutomation setup

11. **Flux Image Update Automations API**
    - URL: https://fluxcd.io/flux/components/image/imageupdateautomations/
    - Purpose: API reference for ImageUpdateAutomation CRD
    - Documents commit, push, update strategies

12. **Flux Security Best Practices**
    - URL: https://fluxcd.io/flux/security/best-practices/
    - Purpose: Security recommendations for Flux deployments
    - Covers multi-tenancy, RBAC, secret management

### Key Best Practices for README Guardrails

#### 1. Webhook Security (Tekton Triggers)

**MUST: Validate webhook origin with GitHub secret tokens**
- Always use `secretRef` to reference a Kubernetes secret containing the webhook secret
- Never expose EventListener without webhook validation
- Create secret in GitHub webhook settings and match in Kubernetes Secret resource
- Example snippet:
  ```yaml
  interceptors:
    - ref:
        name: "github"
      params:
        - name: "secretRef"
          value:
            secretName: github-webhook-secret
            secretKey: secretToken
        - name: "eventTypes"
          value: ["push", "pull_request"]
  ```

**MUST: Filter by event types**
- Specify only needed event types to reduce unnecessary pipeline executions
- Use CEL interceptors for additional filtering logic
- Example: filter for specific branches, file paths, or PR states

#### 2. Flux Image Automation Markers

**MUST: Use correct marker format inline in YAML**
- Standard format: `{"$imagepolicy": "<policy-namespace>:<policy-name>"}`
- Tag-specific: `{"$imagepolicy": "<policy-namespace>:<policy-name>:tag"}`
- Name-specific: `{"$imagepolicy": "<policy-namespace>:<policy-name>:name"}`
- Digest-specific: `{"$imagepolicy": "<policy-namespace>:<policy-name>:digest"}`
- Markers are INLINE COMMENTS in YAML files (not annotations or separate fields)

**MUST: Configure ImageUpdateAutomation with write permissions**
- Flux needs write access to Git repository for automated updates
- Use `--read-write-key` flag during bootstrap if using GitHub
- Configure commit author details for audit trail

**SHOULD NOT: Store image pull secrets in Git**
- Encrypt secrets using Mozilla SOPS or Sealed Secrets
- Reference encrypted secrets in ImageRepository `secretRef`
- Document secret encryption workflow

#### 3. Task Security (git-clone, kaniko)

**MUST: Use SSH or basic-auth for private repositories**
- Bind SSH directory workspace for private Git authentication
- Bind basic-auth workspace for username/password auth
- Prefer SSH over basic-auth when possible

**MUST: Secure dockerconfig for registry authentication**
- Bind dockerconfig workspace for Kaniko registry auth
- Use Kubernetes Secret with Docker config JSON
- Never expose Docker credentials in pipeline parameters

**MUST NOT: Run as root in production without justification**
- Kaniko requires running as root (runAsUser: 0)
- Document security context requirements
- Consider OpenShift SCCs and security contexts

#### 4. General Security Considerations

**MUST NOT: Expose Tekton Dashboard to public internet**
- Use port-forwarding or internal network access only
- Implement OAuth/OIDC if external access required (Tekton dashboard RBAC is limited)
- See: https://github.com/tektoncd/pipeline/issues/7940

**MUST: Use minimum necessary permissions**
- EventListener service account should only have permissions to create TaskRuns/PipelineRuns
- Flux image automation should only write to specific Git paths
- Use namespace-scoped Tasks over ClusterTasks when possible

**SHOULD: Enable audit logging**
- Flux supports commit message templates for change tracking
- Tekton PipelineRuns include execution logs
- Monitor webhook delivery failures for potential attacks

### Additional Notes

#### Tekton Hub Deprecation
- **Critical**: Tekton Hub is deprecated and shutting down in 2026
- Public hub instance: https://hub.tekton.dev shuts down January 7, 2026
- Repository archived: https://github.com/tektoncd/hub archived February 7, 2026
- **Alternative**: Use Tekton Catalog directly from GitHub
- Tasks available at: `kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/<task-name>/<version>/<task-name>.yaml`

#### Image Tag Sorting for Flux Automation
- Flux doesn't support selecting latest image by build time (rate limiting)
- Use sortable tags: timestamp-based (`date +%F.%H%M%S`) or serial numbers
- Recommended format: `<branch>-<sha1>-<timestamp>`
- Guide: https://fluxcd.io/flux/guides/sortable-image-tags/

#### Resource Version Notes
- git-clone task v0.9: requires Tekton Pipelines v0.38.0 or later
- kaniko task v0.7: requires Tekton Pipelines v0.43.0 or later
- Both tasks marked as deprecated in annotations but actively maintained
- Minimum pipeline versions specified in `tekton.dev/pipelines.minVersion` annotation
- /home/gmn/projects/tekton/docs/tekton-k3s-cicd-runbook.md (verify + notes)
- /home/gmn/projects/tekton/flux/demo-app-image-automation.yaml
- /home/gmn/projects/tekton/flux/demo-app-kustomization.yaml
- /home/gmn/projects/tekton/flux/demo-app-source.yaml

## 2026-02-02T10:13:28Z Task: for-humans

Added ### Prerequisites and ### Quick Start subsections under ## For Humans in README.md.
- Prerequisites: concise list of assumptions (k3s, Tekton, Flux, Harbor, DNS/Ingress+TLS) with link to runbook
- Quick Start: minimal copy/paste verify commands (kubectl/tkn/flux/curl) and directory pointers
- Kept all existing content intact; inserted new sections before ### Architecture
- /home/gmn/projects/tekton/tekton-demo-app/k8s/deployment.yaml
- /home/gmn/projects/tekton/tekton/pipeline.yaml
- /home/gmn/projects/tekton/tekton/pipelinerun-manual.yaml
- /home/gmn/projects/tekton/tekton/triggers/ingress.yaml
- /home/gmn/projects/tekton/tekton/triggers/event-listener.yaml
- /home/gmn/projects/tekton/tekton/triggers/trigger-template.yaml
- /home/gmn/projects/tekton/tekton/triggers/rbac.yaml

## 2026-02-02T10:13:28Z Task: llm-preflight-step0

Added ### Pre-flight Checks and ### Step 0: Gather Inputs sections under ## For LLM Agents in README.md.

### Pre-flight Checks
- Copy/paste commands to verify prerequisites:
  - Tekton and Flux namespaces exist (`kubectl get namespace`)
  - EventListener exists (`kubectl get eventlistener`)
  - Flux controllers healthy (`flux check`)
  - Optional secret name checks (harbor-creds, github-webhook-secret, harbor-registry)
  - Optional demo pattern file path confirmation (`ls -la tekton/ tekton/triggers/ flux/`)

### Step 0: Gather Inputs
- Table format with exact questions an LLM should ask
- Variables: APP_NAME, GITHUB_REPO_EXISTS, GITHUB_USER, REGISTRY_PROJECT, DOMAIN, NAMESPACE
- Defaults: REGISTRY_PROJECT=personal, DOMAIN=germainleignel.com, NAMESPACE={{APP_NAME}}
- Bash validation snippet for APP_NAME DNS label (`^[a-z0-9-]+$`) and non-empty vars
- Kept existing Template Variables and Decision/Inputs block after the new sections

## 2026-02-02T10:13:28Z Task: llm-steps-1-6

Added Steps 1-6 under ## For LLM Agents in README.md:

### Step 1: GitHub Repo
- Covers both new repo from scratch and existing repo integration
- Provides copy/paste commands for manual repo creation
- Optional gh CLI commands marked as optional

### Step 2: App Repo Layout (Dockerfile + k8s/)
- Required file structure: Dockerfile, k8s/deployment.yaml, k8s/service.yaml, k8s/ingress.yaml
- Explicit Flux marker format: `# {"$imagepolicy": "flux-system:{{APP_NAME}}"}`
- Template edits shown as "copy file X -> new file Y and replace these fields"

### Step 3: Tekton (Pipeline + Triggers)
- Explicitly calls out Tekton Hub deprecation
- Uses Tekton Catalog raw URLs for git-clone (v0.9) and kaniko (v0.7) tasks
- Pipeline params grounded in tekton/pipeline.yaml: repo-url, revision=main default, image-repo default
- TriggerTemplate references: pvc workspace, dockerconfig secretName harbor-creds
- EventListener strategy subsection with Option A (add to existing) and Option B (dedicated listener)
- CEL filter example with repo full_name matching

### Step 4: Flux (GitOps + Image Automation)
- GitRepository from flux/demo-app-source.yaml: url + secretRef demo-app-git
- ImageRepository/ImagePolicy/ImageUpdateAutomation from flux/demo-app-image-automation.yaml
- Image policy filterTags pattern: '^[0-9a-f]{7,40}$' (git SHA matching)
- Policy alphabetical asc order for git SHA tags
- Update strategy: Setters

### Step 5: GitHub Webhook
- Webhook URL from ingress.yaml: https://tekton.germainleignel.com/github
- Secret name only: github-webhook-secret
- Push event configuration

### Step 6: End-to-End Verify
- Trigger build via git push
- Verify PipelineRun with tkn CLI
- Verify image pushed (git SHA and latest tags)
- Verify Flux image automation (ImageRepository, ImagePolicy, ImageUpdateAutomation)
- Verify Git commit from Flux
- Verify deployment in cluster
- Full flow verification commands

All secrets referenced by name only (no values or output commands).
All file paths and values grounded in actual repo YAML files.
