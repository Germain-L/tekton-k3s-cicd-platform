## 2026-02-01T11:53:32+01:00 Init
- Plan contains 27 unchecked boxes; only 13 are execution tasks, rest are acceptance checklists.
- Any public ingress must have authentication.

## 2026-02-01T12:18:30+01:00 Tekton Hub DNS
- `api.hub.tekton.dev` is not resolvable from this environment (DNS lookup fails via 10.255.255.254).
- Prefer Tekton Catalog OCI bundles (e.g. `ghcr.io/tektoncd/catalog/upstream/tasks/...`) instead of `api.hub.tekton.dev` raw installs.

## 2026-02-01T13:20:00+01:00 Flux Image Automation Template
- `ImageUpdateAutomation.spec.git.commit.messageTemplate` no longer supports `.Updated`; use `.Changed` or a static message.
