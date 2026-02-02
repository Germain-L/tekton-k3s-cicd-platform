# LLM-Optimized README for Tekton CI/CD Platform

## TL;DR

> **Create a README that enables both humans and LLM agents to onboard new applications to an existing Tekton + FluxCD CI/CD pipeline on k3s.**
>
> **Deliverables**:
> - README.md with "For Humans" and "For LLM Agents" sections
> - Clear decision trees for LLM automation
> - Copy-paste commands for common scenarios
> - Verification procedures at each step
> - Guardrails and scope boundaries
>
> **Estimated Effort**: Short (2-4 hours)
> **Parallel Execution**: NO - Sequential document creation
> **Critical Path**: Structure → Content → Verification → Review

---

## Context

### Original Request
Create a README similar to oh-my-opencode's installation guide to automate Tekton and k8s deployments for projects.

### Interview Summary
**Key Decisions**:
- **Primary focus**: Onboard new applications (not initial cluster setup)
- **Structure**: Both "For Humans" and "For LLM Agents" sections
- **Automation scope**: Both new apps from scratch AND existing app integration
- **Target**: Enable LLM agents to fully automate onboarding with minimal user input

**Repository Context**:
- Existing CI/CD platform: Tekton Pipelines + Triggers, FluxCD, Harbor registry
- Demo app: `tekton-demo-app` serves as reference implementation
- Work plan: `.sisyphus/plans/tekton-k3s-cicd-platform.md` (completed)
- Registry: `registry.germainleignel.com/personal`
- Domain: `*.germainleignel.com`

### Metis Review Findings
**Identified Gaps** (addressed in plan):
- Naming conventions: DNS-compliant kebab-case for app names
- Registry project: Default to `personal`, configurable per-app
- Secret management: Reuse existing secrets (`harbor-creds`, `harbor-registry`, `demo-app-git`)
- Domain pattern: `<app>.germainleignel.com` subdomain
- Template variables: `{{APP_NAME}}`, `{{GITHUB_USER}}`, `{{REGISTRY_PROJECT}}`
- Scope boundaries: NO cluster-wide changes, NO other apps affected
- Edge cases: Repo exists, namespace exists, Flux resources exist

---

## Work Objectives

### Core Objective
Create an LLM-optimized README that enables AI agents to onboard new applications to the CI/CD pipeline with minimal human intervention.

### Concrete Deliverables
1. README.md with dual-section structure (Humans + LLM Agents)
2. "For Humans" section: Overview, quick-start, architecture
3. "For LLM Agents" section: Step-by-step automation guide
4. Template variable substitution guide
5. Verification commands for each step
6. Guardrails and "MUST NOT" constraints
7. Error handling and rollback procedures

### Definition of Done
- [x] README.md created with both sections
- [x] LLM agent can follow instructions to onboard a new app
- [x] All template files referenced correctly
- [x] Verification commands are executable
- [x] Guardrails prevent cluster-wide modifications

### Must Have
- Clear decision tree for LLM (Step 0 questions)
- Copy-paste template instructions
- Executable verification commands
- Reference to existing `tekton-demo-app/` as template
- Pre-flight checklist for prerequisites

### Must NOT Have (Guardrails)
- NO cluster setup instructions (Tekton/Flux already installed)
- NO Harbor installation instructions
- NO modifications to `tekton-pipelines` or `flux-system` namespaces
- NO changes to other existing applications
- NO support for multiple environments (prod only)
- NO database or stateful service setup

---

## Verification Strategy

### Test Decision
- **Infrastructure exists**: YES (README is documentation)
- **User wants tests**: Manual verification only
- **QA approach**: Manual verification with specific commands

### Automated Verification (Agent-Executable)

**For README Structure**:
```bash
# Verify README exists and has required sections
grep -q "## For Humans" README.md && echo "Humans section: OK"
grep -q "## For LLM Agents" README.md && echo "LLM section: OK"
grep -q "Step 0:" README.md && echo "Decision tree: OK"
```

**For Template References**:
```bash
# Verify all referenced files exist
ls tekton-demo-app/Dockerfile
ls tekton-demo-app/k8s/deployment.yaml
ls tekton/pipeline.yaml
ls flux/demo-app-source.yaml
# Expected: All files exist
```

**For LLM Agent Test**:
```bash
# Simulate LLM following instructions (dry run)
# Extract commands from README and verify syntax
# This will be done manually by reviewer
```

---

## Execution Strategy

### Sequential Execution (Documentation Task)

```
Step 1: Create README structure
├── Define sections and headers
├── Add frontmatter and TL;DR
└── Set up template variable convention

Step 2: Write "For Humans" section
├── Architecture overview
├── Quick-start guide
├── Prerequisites checklist
└── Common commands reference

Step 3: Write "For LLM Agents" section
├── Pre-flight checks
├── Step 0: User questions
├── Step 1-6: Implementation steps
├── Verification procedures
└── Error handling

Step 4: Add guardrails and constraints
├── "MUST NOT" list
├── Scope boundaries
└── Safety checks

Step 5: Final review and polish
├── Verify all file references
├── Test command syntax
└── Check formatting
```

---

## TODOs

- [x] **1. Create README Structure and Frontmatter**

  **What to do**:
  - Create README.md with proper structure
  - Add TL;DR section
  - Define template variable convention (`{{APP_NAME}}`, etc.)
  - Set up section headers

  **Must NOT do**:
  - Skip the dual-section structure
  - Use vague placeholders without examples

  **Recommended Agent Profile**:
  - **Category**: `writing`
  - **Reason**: Documentation creation task
  - **Skills**: None required

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Blocked By**: None
  - **Blocks**: Task 2, 3, 4

  **References**:
  - Pattern: oh-my-opencode installation.md structure
  - Current README: `./README.md` (for context)

  **Acceptance Criteria**:
  ```bash
  # Verify file created
  test -f README.md && echo "README exists: OK"
  
  # Verify sections exist
  grep -q "## TL;DR" README.md
  grep -q "## For Humans" README.md
  grep -q "## For LLM Agents" README.md
  # Expected: All grep commands exit 0
  ```

  **Commit**: YES
  - Message: `docs: create README structure with dual-section layout`
  - Files: `README.md`

- [x] **2. Write "For Humans" Section**

  **What to do**:
  - Architecture diagram/description
  - Quick-start for common case (new Go app)
  - Prerequisites checklist
  - URLs and endpoints reference
  - File structure overview

  **Must NOT do**:
  - Include LLM-specific instructions here
  - Overwhelm with too much detail

  **Recommended Agent Profile**:
  - **Category**: `writing`

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Blocked By**: Task 1
  - **Blocks**: Task 5

  **References**:
  - Current README: `./README.md` (extract architecture, URLs, files)
  - Runbook: `docs/tekton-k3s-cicd-runbook.md`

  **Acceptance Criteria**:
  ```bash
  # Verify content
  grep -q "Architecture" README.md
  grep -q "Quick Start" README.md
  grep -q "Prerequisites" README.md
  # Expected: All sections present
  ```

  **Commit**: YES
  - Message: `docs: add For Humans section with quick-start guide`
  - Files: `README.md`

- [x] **3. Write "For LLM Agents" - Pre-flight and Step 0**

  **What to do**:
  - Pre-flight checks (verify cluster, Flux, Tekton)
  - Step 0: Ask user questions
    - App name (DNS-compliant validation)
    - GitHub user/org
    - Repo exists? (yes/no)
    - Language/framework
    - Registry project (default: personal)
  - Document decision tree with exact commands

  **Must NOT do**:
  - Skip validation steps
  - Assume inputs are valid

  **Recommended Agent Profile**:
  - **Category**: `writing`

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Blocked By**: Task 1
  - **Blocks**: Task 4

  **References**:
  - Pattern: oh-my-opencode "Step 0: Ask user about subscriptions"
  - Validation: DNS naming conventions

  **Acceptance Criteria**:
  ```bash
  # Verify sections
  grep -q "Pre-flight Checks" README.md
  grep -q "Step 0:" README.md
  grep -q "App name" README.md
  grep -q "GitHub" README.md
  # Expected: All present
  ```

  **Commit**: YES
  - Message: `docs: add LLM pre-flight checks and decision tree`
  - Files: `README.md`

- [x] **4. Write "For LLM Agents" - Steps 1-6**

  **What to do**:
  - Step 1: GitHub repo (create or verify)
  - Step 2: Generate app manifests (Dockerfile + k8s/)
  - Step 3: Copy/adapt Tekton resources
  - Step 4: Copy/adapt Flux resources
  - Step 5: Configure GitHub webhook
  - Step 6: End-to-end verification
  - Include exact template substitution commands

  **Must NOT do**:
  - Skip verification after each step
  - Forget to document variable substitution

  **Recommended Agent Profile**:
  - **Category**: `writing`

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Blocked By**: Task 3
  - **Blocks**: Task 5

  **References**:
  - Template files: `tekton-demo-app/`, `tekton/`, `flux/`
  - Pattern: `sed` or `envsubst` for variable substitution
  - Example: `sed 's/demo-app/{{APP_NAME}}/g' template.yaml`

  **Acceptance Criteria**:
  ```bash
  # Verify all steps documented
  grep -q "Step 1:" README.md
  grep -q "Step 2:" README.md
  grep -q "Step 3:" README.md
  grep -q "Step 4:" README.md
  grep -q "Step 5:" README.md
  grep -q "Step 6:" README.md
  # Expected: All steps present
  ```

  **Commit**: YES
  - Message: `docs: add LLM implementation steps 1-6`
  - Files: `README.md`

- [x] **5. Add Guardrails, Error Handling, and Final Polish**

  **What to do**:
  - Add "MUST NOT" guardrails section
  - Document scope boundaries
  - Add error handling for common failures
  - Include rollback procedures
  - Final formatting and review
  - Verify all file references are correct

  **Must NOT do**:
  - Skip the guardrails (critical for safety)
  - Leave TODOs or placeholders

  **Recommended Agent Profile**:
  - **Category**: `writing`

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Blocked By**: Task 2, 4
  - **Blocks**: None

  **References**:
  - Guardrails from Metis review
  - Error patterns from runbook troubleshooting

  **Acceptance Criteria**:
  ```bash
  # Verify guardrails
  grep -q "MUST NOT" README.md
  grep -q "Guardrails" README.md
  grep -q "Rollback" README.md
  
  # Verify file references exist
  grep -o '`[^`]*\.yaml`' README.md | tr -d '`' | while read f; do test -f "$f" && echo "OK: $f" || echo "MISSING: $f"; done
  # Expected: All referenced files exist
  ```

  **Commit**: YES
  - Message: `docs: add guardrails, error handling, and finalize README`
  - Files: `README.md`

---

## Commit Strategy

| After Task | Message | Files | Verification |
|------------|---------|-------|--------------|
| 1 | `docs: create README structure with dual-section layout` | README.md | grep sections |
| 2 | `docs: add For Humans section with quick-start guide` | README.md | grep content |
| 3 | `docs: add LLM pre-flight checks and decision tree` | README.md | grep steps |
| 4 | `docs: add LLM implementation steps 1-6` | README.md | grep all steps |
| 5 | `docs: add guardrails, error handling, and finalize README` | README.md | verify refs |

---

## Success Criteria

### Verification Commands
```bash
# README exists and is readable
cat README.md | head -50

# All required sections present
grep -c "## For Humans" README.md  # Expected: 1
grep -c "## For LLM Agents" README.md  # Expected: 1
grep -c "Step 0:" README.md  # Expected: >=1
grep -c "Step 6:" README.md  # Expected: >=1

# All file references valid
grep -oE '[a-zA-Z0-9_-]+\.(yaml|md|Dockerfile)' README.md | sort -u | while read f; do test -f "$f" && echo "✓ $f" || echo "✗ MISSING: $f"; done

# Guardrails present
grep -q "MUST NOT" README.md && echo "Guardrails: OK"
grep -q "tekton-pipelines" README.md && echo "Namespace refs: OK"
grep -q "flux-system" README.md && echo "Flux refs: OK"
```

### Final Checklist
- [x] README has "For Humans" section with quick-start
- [x] README has "For LLM Agents" section with steps 0-6
- [x] Template variable convention documented (`{{APP_NAME}}`, etc.)
- [x] All referenced files exist in repo
- [x] Guardrails prevent cluster-wide modifications
- [x] Verification commands are executable
- [x] Error handling and rollback documented
- [x] Formatting is consistent and readable

---

## Template Variable Convention

Document these in the README:

| Variable | Description | Example | Validation |
|----------|-------------|---------|------------|
| `{{APP_NAME}}` | DNS-compliant app name | `my-app` | `^[a-z0-9-]+$` |
| `{{GITHUB_USER}}` | GitHub username or org | `Germain-L` | From git config or user input |
| `{{REGISTRY_PROJECT}}` | Harbor project name | `personal` | Default: `personal` |
| `{{DOMAIN}}` | Base domain | `germainleignel.com` | Default: `germainleignel.com` |
| `{{NAMESPACE}}` | K8s namespace | `my-app` | Default: same as `{{APP_NAME}}` |

---

## File References to Include

### Template Files (for LLM "Copy & Adapt")

**From `tekton-demo-app/`:**
- `Dockerfile` - Multi-stage build template
- `k8s/namespace.yaml` - Namespace definition
- `k8s/deployment.yaml` - Deployment with Flux marker
- `k8s/service.yaml` - Service definition
- `k8s/ingress.yaml` - Ingress with TLS
- `k8s/kustomization.yaml` - Kustomize base
- `flux/kustomization.yaml` - Flux Kustomization

**From `tekton/`:**
- `pipeline.yaml` - Build pipeline
- `triggers/trigger-template.yaml` - PipelineRun template
- `triggers/trigger-binding.yaml` - GitHub payload binding
- `triggers/event-listener.yaml` - Webhook receiver
- `triggers/rbac.yaml` - EventListener permissions
- `triggers/ingress.yaml` - Webhook ingress

**From `flux/`:**
- `demo-app-source.yaml` - GitRepository source
- `demo-app-kustomization.yaml` - Kustomization
- `demo-app-image-automation.yaml` - Image automation resources

---

## Next Steps After Completion

1. **Test with Real LLM**: Have an LLM agent follow the README to onboard a test app
2. **Gather Feedback**: Ask users what was unclear
3. **Iterate**: Update based on real-world usage
4. **Add More Templates**: Support Node.js, Python, etc.
5. **Automation Script**: Consider creating a shell script that implements the LLM steps
