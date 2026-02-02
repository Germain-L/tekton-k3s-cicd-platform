# tekton (k3s homelab CI/CD)

This repo contains the manifests + runbook for a personal CI/CD setup on k3s:

- CI: Tekton (Pipelines + Triggers)
- CD: FluxCD (GitRepository + Kustomization) + Flux image automation
- Registry: Harbor (`registry.germainleignel.com/personal`)

See: `docs/tekton-k3s-cicd-runbook.md`

## If you are an LLM

```text
To onboard a new app: create a new GitHub repo with a Dockerfile + k8s/ (Deployment/Service/Ingress) that references registry.germainleignel.com/personal/<app> and includes the Flux marker comment # {"$imagepolicy":"flux-system:<app>"} in k8s/deployment.yaml; copy/adapt this repo's tekton/ + tekton/triggers/ (pipeline + EventListener/TriggerTemplate/TriggerBinding/Ingress) and set the GitHub webhook URL to https://tekton.germainleignel.com/github with secret tekton-pipelines/github-webhook-secret, wire GitOps + image automation from flux/ (GitRepository/Kustomization/ImageRepository/ImagePolicy/ImageUpdateAutomation) with secrets flux-system/demo-app-git + flux-system/harbor-registry and demo-app/regcred + tekton-pipelines/harbor-creds (names only), then verify with kubectl + tkn + flux + curl; see docs/tekton-k3s-cicd-runbook.md.
```

## Architecture

Flow:

1) GitHub push to `main`
2) GitHub webhook -> Tekton Triggers EventListener
3) Tekton PipelineRun:
   - clone repo
   - build/push with Kaniko
4) Flux:
   - scans registry tags
   - updates `k8s/deployment.yaml` image tag in the demo repo
   - applies Kustomize manifests to the cluster

## Repos

- Demo app repo: https://github.com/Germain-L/tekton-demo-app
- This repo: cluster-side YAML and operational notes

## URLs

- Tekton webhook endpoint: `https://tekton.germainleignel.com/github`
- Weave GitOps UI: `https://gitops.germainleignel.com/` (basic auth)
- Demo app: `https://demo-app.germainleignel.com/` (basic auth)

## Files

- Tekton pipeline + manual run: `tekton/`
- Tekton triggers resources: `tekton/triggers/`
- Flux CD resources: `flux/`
- Runbook: `docs/tekton-k3s-cicd-runbook.md`

## Cluster Objects (namespaces)

- `tekton-pipelines`: Tekton controllers, pipeline, triggers, EventListener
- `flux-system`: Flux controllers, Weave GitOps, image automation resources
- `demo-app`: demo workload

## Secrets (names only)

No secrets are stored in this repo.

- `tekton-pipelines/harbor-creds`: Kaniko docker config mounted at `/kaniko/.docker/config.json`
- `tekton-pipelines/github-webhook-secret`: GitHub webhook signature secret (key `secretToken`)
- `flux-system/demo-app-git`: Git credentials for Flux to push automation commits
- `flux-system/harbor-registry`: Registry credentials for Flux ImageRepository scans
- `demo-app/regcred`: imagePullSecret for Harbor

## Verify

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

## Notes

- Tekton Triggers needs both `release.yaml` and `interceptors.yaml` (otherwise github/cel interceptors will fail).
- If TaskRuns are blocked by PodSecurity, check the `tekton-pipelines` namespace PSA labels.
