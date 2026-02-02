# Tekton CI/CD on k3s (Homelab Runbook)

This documents the current CI/CD setup running on the k3s node `space`.

## Components

- Tekton Pipelines + Triggers: `tekton-pipelines` namespace
- Tekton Dashboard (local-only): `tekton-pipelines` namespace
- FluxCD: `flux-system` namespace
- Weave GitOps UI: `flux-system` namespace
- Registry: Harbor at `registry.germainleignel.com` (project: `personal`)

## URLs

- Demo app: `https://demo-app.germainleignel.com/` (protected by Traefik basic auth)
- Weave GitOps: `https://gitops.germainleignel.com/` (protected by Traefik basic auth)
- Tekton webhook endpoint (GitHub): `https://tekton.germainleignel.com/github`

## Namespaces

- `tekton-pipelines`: Tekton controllers, EventListener, pipeline definitions
- `flux-system`: Flux controllers, Weave GitOps
- `demo-app`: Demo app workload

## Secrets (Names Only)

Do not commit secret values to git.

Tekton:
- `tekton-pipelines/harbor-creds`: Docker config mounted for Kaniko at `/kaniko/.docker/config.json`
- `tekton-pipelines/github-webhook-secret`: GitHub webhook signature secret (key: `secretToken`)

Flux:
- `flux-system/demo-app-git`: Git credentials for Flux image automation pushing to GitHub
- `flux-system/harbor-registry`: Registry credentials for ImageRepository scanning

Demo app:
- `demo-app/regcred`: Image pull secret for Harbor

## Pipelines

Pipeline:
- `tekton-pipelines/demo-app-pipeline`

Behavior:
- On GitHub push to `main`, Tekton builds and pushes:
  - `registry.germainleignel.com/personal/demo-app:<git-sha>`
  - `registry.germainleignel.com/personal/demo-app:latest`

## Flux CD (Deployment)

Flux resources in `flux-system`:
- Git source: `GitRepository/demo-app`
- CD: `Kustomization/demo-app` applies `./k8s` from the demo repo
- Image automation:
  - `ImageRepository/demo-app`
  - `ImagePolicy/demo-app`
  - `ImageUpdateAutomation/demo-app`

The demo repo uses a Flux image policy marker in `k8s/deployment.yaml`:
`# {"$imagepolicy": "flux-system:demo-app"}`

## Verify

Tekton core:
```bash
kubectl get pods -n tekton-pipelines
tkn pipeline list -n tekton-pipelines
tkn pipelinerun list -n tekton-pipelines
```

Flux:
```bash
flux check
flux get sources git -n flux-system
flux get kustomizations -n flux-system
kubectl get imagepolicy -n flux-system
kubectl get imageupdateautomation -n flux-system
```

Demo app:
```bash
kubectl get deploy -n demo-app
kubectl get ingress -n demo-app
curl -I https://demo-app.germainleignel.com/
```

## Troubleshooting Notes

- Tekton Triggers interceptors: if EventListener using `github`/`cel` crashloops with caBundle errors, ensure `interceptors.yaml` is applied.
- Pod Security Admission: if TaskRuns are blocked, check namespace labels on `tekton-pipelines` (PSA enforcement level).
