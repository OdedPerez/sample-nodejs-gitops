# sample-nodejs-gitops

Helm chart, per-environment values, and ArgoCD manifests for
[`sample-nodejs`](https://github.com/OdedPerez/sample-nodejs), watched by ArgoCD.

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

This repo owns everything ArgoCD needs to deploy the app — the Helm chart,
per-environment values, and the `Application`/`AppProject` manifests —
kept separate from `sample-nodejs`'s own source/CI concerns. For the full
reasoning (why a separate repo, why the chart lives here rather than in
the app repo), see [`sample-nodejs`'s `SUBMISSION.md`](https://github.com/OdedPerez/sample-nodejs/blob/main/SUBMISSION.md#gitops--argocd).

## How deployment state gets updated here

`sample-nodejs`'s `release.yml` pipeline bumps `staging/values.yaml` and
`production/values.yaml`'s `image.tag` here on every release (production
gated behind manual approval); ArgoCD reconciles the cluster automatically
from there. GHCR is **private**, so each environment's `values.yaml` also
sets `imagePullSecrets: [{name: ghcr-pull-secret}]`, a Secret created once
per namespace directly via `kubectl` (not git-tracked).

## Bootstrapping ArgoCD to watch this repo

```bash
kubectl apply -f argocd/appproject-staging.yaml
kubectl apply -f argocd/appproject-prod.yaml
kubectl apply -f argocd/application-staging.yaml
kubectl apply -f argocd/application-prod.yaml
kubectl apply -f argocd/application-monitoring.yaml   # optional: Prometheus/Grafana
kubectl -n argocd get applications
```
