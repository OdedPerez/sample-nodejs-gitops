# sample-nodejs-gitops

GitOps deployment state for [`sample-nodejs`](https://github.com/OdedPerez/sample-nodejs), watched by ArgoCD.

## Layout

```
staging/values.yaml       # environment values for staging
production/values.yaml    # environment values for production
argocd/
  application-staging.yaml
  application-prod.yaml
```

## Why a separate repo, but not a separate chart

The Helm chart itself (`helm/sample-nodejs/`) lives in the `sample-nodejs`
app repo, alongside the Dockerfile it packages and the code its probes and
templates are tied to — chart correctness is coupled to app correctness
(a probe path that doesn't match a real endpoint is a bug caught in the
same PR, not two).

What *does* live here is per-environment operational state: which image
tag is live in staging/production right now. That's a different concern
from the chart's structure — it changes on every release rather than every
app change, and updating it shouldn't require a Docker rebuild, a Trivy
scan, or an ESLint pass. Separating it into its own repo also means CI
never writes generated deployment-state commits back into the app repo's
own history.

Each `Application` in `argocd/` is a multi-source ArgoCD Application
(stable since Argo CD 2.6): one source pulls the chart from `sample-nodejs`,
the second (`ref: values`) pulls this repo's environment values file, and
`helm.valueFiles: ['$values/<env>/values.yaml']` merges them.

## How deployment state gets updated here

`sample-nodejs`'s `release.yml` pipeline pushes commits to this repo's
`staging/values.yaml` (on every merge to `main`) and `production/values.yaml`
(after the `production` GitHub Environment's manual approval) — bumping
`image.tag` to the version it just built, scanned, and pushed to GHCR.
ArgoCD watches this repo and reconciles the cluster automatically
(`syncPolicy.automated` with `prune` + `selfHeal`) — the git commit *is*
the deployment.

GHCR is **private**, so each environment's `values.yaml` also sets
`imagePullSecrets: [{name: ghcr-pull-secret}]` — a `kubernetes.io/dockerconfigjson`
Secret created once per namespace directly via `kubectl` (not git-tracked,
since it holds a real credential).

## Bootstrapping ArgoCD to watch this repo

```bash
kubectl apply -f argocd/application-staging.yaml
kubectl apply -f argocd/application-prod.yaml
kubectl -n argocd get applications
```
