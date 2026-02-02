## 2026-02-01T11:53:32+01:00 Init
- k3s single-node; default StorageClass is local-path.
- Ingress controller is Traefik; wildcard DNS *.germainleignel.com points to router and 443 is forwarded.
- cert-manager is installed and working.
- Harbor registry is already present (hosts include harbor.germainleignel.com and registry.germainleignel.com).

## 2026-02-01T12:18:30+01:00 Tekton Task Sources
- If Tekton Hub raw URLs are unreachable, use Tekton Catalog OCI bundles in `taskRef.bundle`.
- Known good bundles: `ghcr.io/tektoncd/catalog/upstream/tasks/git-clone:0.9`, `ghcr.io/tektoncd/catalog/upstream/tasks/kaniko:0.6`.

## 2026-02-01T12:20:30+01:00 Registry Path
- The Harbor project `personal` exists under `registry.germainleignel.com/personal` and is already in use by the cluster.

## 2026-02-01T12:32:00+01:00 Tekton Bundles
- Tekton tasks are referenced via OCI bundles because `api.hub.tekton.dev` DNS is unavailable from the cluster.

## 2026-02-01T12:35:30+01:00 Triggers Interceptors
- Tekton Triggers requires applying both `triggers/.../release.yaml` and `triggers/.../interceptors.yaml`; otherwise EventListener with `github`/`cel` interceptors crashloops (caBundle missing).

## 2026-02-01T12:40:30+01:00 Pod Security Admission
- Namespace `tekton-pipelines` was labeled `pod-security.kubernetes.io/enforce=restricted` by default, which blocked TaskRun pods.
- Changed it to `enforce=baseline` so Tekton TaskRuns can run.

## 2026-02-01T12:45:30+01:00 Traefik Basic Auth Pattern
- Traefik Middleware CRD exists as `traefik.io/v1alpha1`.
- Existing pattern: `Middleware.spec.basicAuth.secret: <secretName>` with secret key `users` (htpasswd content, base64-encoded).
- Ingress uses annotation `traefik.ingress.kubernetes.io/router.middlewares: <namespace>-<middlewareName>@kubernetescrd`.

## 2026-02-01T13:10:00+01:00 Weave GitOps TLS
- `gitops.germainleignel.com` uses cert-manager HTTP-01; during issuance the challenge returned 502 until the solver could be reached.
- Once issued, `https://gitops.germainleignel.com/` returns 401 (Traefik basic auth) as expected.

## 2026-02-01T13:10:00+01:00 Tekton Registry Credentials
- Kaniko Task expects the docker config mounted at `/kaniko/.docker/config.json`.
- `tekton-pipelines/harbor-creds` was created as type Opaque with key `config.json` and contains auth for `registry.germainleignel.com`.

## 2026-02-01T13:10:00+01:00 GitHub Webhook
- GitHub webhook endpoint for Tekton Triggers: `https://tekton.germainleignel.com/github`.
- Signature secret stored in `tekton-pipelines/github-webhook-secret` (key `secretToken`).
