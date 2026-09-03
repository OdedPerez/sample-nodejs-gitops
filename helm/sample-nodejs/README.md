# Helm chart - `sample-nodejs`

Helm chart to deploy the `sample-nodejs` service to Kubernetes. This is the
deliverable for the **Helm** part of the challenge.

## Layout
```
helm/
└── sample-nodejs/
    ├── Chart.yaml              # chart + app version
    ├── values.yaml             # all defaults (HA features ON)
    ├── values-minikube.yaml    # single-node overlay (HA features OFF)
    ├── .helmignore
    └── templates/
        ├── _helpers.tpl        # shared naming/labels
        ├── deployment.yaml     # probes, resources, security, env-from cm/secret
        ├── service.yaml
        ├── ingress.yaml
        ├── configmap.yaml
        ├── secret.yaml
        ├── serviceaccount.yaml
        ├── hpa.yaml
        ├── pdb.yaml
        ├── networkpolicy.yaml   # opt-in
        ├── servicemonitor.yaml  # opt-in
        └── NOTES.txt
```

## Why a Deployment (not a StatefulSet)
The service is **stateless** - replicas are interchangeable, hold no per-pod
data, and need no stable network identity or ordered startup/shutdown. A
Deployment gives fast horizontal scaling, zero-downtime rolling updates
(`maxUnavailable: 0`), and lets the HPA scale freely. A StatefulSet earns its
complexity for workloads with stable identities and per-replica persistent
volumes (databases, brokers) - none of which apply to this app.

## Requirements coverage
- **Liveness + readiness probes** (+ a startup probe) → `deployment.yaml`,
  hitting the app's own `/live` and `/ready` endpoints.
- **Service + Ingress** → `service.yaml` (ClusterIP), `ingress.yaml` (class,
  host, optional TLS).
- **Resource requests *and* limits** for CPU + memory → `deployment.yaml` /
  `values.yaml`.
- **ConfigMap + Secret**, injected via `envFrom`; a `checksum/*` pod-template
  annotation rolls pods automatically when config/secret content changes.
  Secret supports `existingSecret` so real secrets can come from Sealed
  Secrets / External Secrets / Vault instead of being committed in Git. (The
  app itself only reads `PORT` - the ConfigMap/Secret keys are illustrative,
  added to demonstrate the wiring end-to-end.)
- **Extras**: HPA, PodDisruptionBudget, topologySpreadConstraints, a
  dedicated ServiceAccount (`automountServiceAccountToken: false`), opt-in
  NetworkPolicy + Prometheus Operator ServiceMonitor, and a hardened
  container security context (non-root, read-only root filesystem, all
  capabilities dropped, seccomp `RuntimeDefault`, with a small `emptyDir`
  mounted at `/tmp` since the root filesystem is read-only).

## Deploy

Multi-node cluster (defaults, HA on):
```bash
helm upgrade --install sample-nodejs helm/sample-nodejs \
  -n sample-nodejs --create-namespace
```

Single-node minikube (HA features off - see overlay). The image is built
from the [`sample-nodejs`](https://github.com/OdedPerez/sample-nodejs)
app repo's Dockerfile; the `helm` commands below run from this
(`sample-nodejs-gitops`) repo's root:
```bash
minikube addons enable ingress
# from a checkout of sample-nodejs:
eval $(minikube docker-env) && docker build -t sample-nodejs:local .
# from a checkout of sample-nodejs-gitops:
helm upgrade --install sample-nodejs helm/sample-nodejs \
  -f helm/sample-nodejs/values-minikube.yaml \
  -n sample-nodejs --create-namespace
```

## Validate
```bash
helm lint helm/sample-nodejs
helm template sample-nodejs helm/sample-nodejs
```

## Staging & production

Not deployed manually with `helm upgrade` - [ArgoCD](https://argo-cd.readthedocs.io/)
manages both, combining this chart with the `staging/`/`production/`
values files at the root of this same repo via a single-source
`Application`. See the root [README](../../README.md) for the values/deploy
flow and the CI/CD pipeline (in the separate `sample-nodejs` app repo) that
feeds it.
