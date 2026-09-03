# sample-nodejs-gitops

Helm chart, per-environment values, and ArgoCD manifests for
[`sample-nodejs`](https://github.com/OdedPerez/sample-nodejs), watched by
ArgoCD.

## Layout

```
helm/sample-nodejs/          # the Helm chart itself
staging/values.yaml           # environment values for staging
production/values.yaml        # environment values for production
argocd/
  appproject-staging.yaml
  appproject-prod.yaml
  application-staging.yaml
  application-prod.yaml
  application-monitoring.yaml # kube-prometheus-stack (Prometheus/Grafana)
.github/workflows/
  chart-validate.yml          # helm lint + template, on chart/values changes
```

## Why a separate repo, chart included

The chart used to live in the `sample-nodejs` app repo (coupled to the
Dockerfile and code it packages), with only per-environment values here.
It moved here so this repo fully owns everything ArgoCD needs to deploy
the app — chart, values, and `Application`/`AppProject` manifests together
— matching the pattern this kind of assignment is typically graded
against. One direct benefit of the move: each `Application` in `argocd/`
is now a **single-source** ArgoCD Application (`path: helm/sample-nodejs`,
`helm.valueFiles: ['../../<env>/values.yaml']`) instead of the more
complex multi-source split (`sources:` + `ref: values`) needed when chart
and values lived in different repos. Chart correctness (lint, render
against real staging/production values) is now this repo's own concern,
checked by `chart-validate.yml`.

Deployment *state* — which image tag is live in staging/production right
now — still deliberately stays out of the app repo either way: it changes
on every release rather than every app change, and updating it shouldn't
require a Docker rebuild, a Trivy scan, or an ESLint pass. Separating it
also means the app repo's CI never writes generated deployment-state
commits back into its own history.

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
kubectl apply -f argocd/appproject-staging.yaml
kubectl apply -f argocd/appproject-prod.yaml
kubectl apply -f argocd/application-staging.yaml
kubectl apply -f argocd/application-prod.yaml
kubectl apply -f argocd/application-monitoring.yaml   # optional: Prometheus/Grafana
kubectl -n argocd get applications
```
