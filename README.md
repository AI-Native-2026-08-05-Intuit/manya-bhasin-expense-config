# manya-bhasin-expense-config

GitOps source of truth for expense-api. Argo CD Applications in the capstone
repo (`manya-bhasin-expense-tracking`) point here.

```
base/                 W5 D3 manifests (verbatim copies)
overlays/dev          namespace expense-dev, 1 replica, LOG_LEVEL=DEBUG
overlays/staging      namespace expense-staging, 2 replicas, LOG_LEVEL=INFO
overlays/prod         namespace expense-prod, 3 replicas, LOG_LEVEL=WARN
```

Namespaces are created by Argo CD `CreateNamespace=true`, not by `00-namespace.yaml`
(that file includes ResourceQuota/LimitRange, which the AppProject blacklists).

CI in the application repo opens image-bump PRs against `overlays/dev/kustomization.yaml`
via `_bump-config.yml`. No cluster credentials live in either repo.

```bash
kustomize build overlays/dev
kustomize build overlays/staging
kustomize build overlays/prod
```
