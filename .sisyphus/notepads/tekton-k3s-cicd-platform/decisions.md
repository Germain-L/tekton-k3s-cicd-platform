## 2026-02-01T11:53:32+01:00 Init
- CI: Tekton (Pipelines + Triggers) builds images.
- CD: FluxCD (plus Weave GitOps UI) deploys via GitOps.
- Tekton Dashboard: local-only via port-forward.
- Repos: one GitHub repo per app.

## 2026-02-01T12:05:00+01:00 Registry Path
- All built images must be pushed to: registry.germainleignel.com/personal
- This applies to the demo app and any future apps in the CI/CD pipeline.

## 2026-02-01T13:10:00+01:00 Demo Repo
- Demo app repository: https://github.com/Germain-L/tekton-demo-app (public, so Tekton can clone without credentials).
