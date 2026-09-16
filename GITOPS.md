# GitOps (expense-api)

Argo CD reconciles
[AI-Native-2026-08-05-Intuit/manya-bhasin-expense-config](https://github.com/AI-Native-2026-08-05-Intuit/manya-bhasin-expense-config)
(`overlays/{dev,staging,prod}` on branch `config-1`). The curriculum's
`uptimecrew/expense-config` does not exist, so that org repo is the source of
truth.

## Repo layout (who owns what)

| Repo | Holds |
| --- | --- |
| `manya-bhasin-expense-config` (GitOps) | `base/`, `overlays/{dev,staging,prod}`, `argocd/projects`, `argocd/applications`, `argocd/applicationsets`, `argocd-system/` |
| `manya-bhasin-expense-tracking` (app) | application source, `Dockerfile`, CI workflows, this doc, `manifests/observability/` |

The Argo CD objects and the W5 D3 workload manifests moved **out** of the app
repo in W6 D2; the brief keeps them in the GitOps repo. `manifests/observability/`
stays in the app repo because the W5 D5 Sloth drift gate in
`.github/workflows/observability.yml` diffs
`manifests/observability/expense-api-prometheusrule.yaml` against freshly
generated output in the same checkout.

`scripts/k8s-up.sh` no longer applies workload manifests. With `GITOPS_DIR`
unset it brings up the cluster + ingress and stops, leaving the workloads to
Argo CD.

CI after W6 D2 only opens a PR on the GitOps repo (`_bump-config.yml`); it does
not roll out with cluster credentials.

Empty `clusterResourceWhitelist` plus `CreateNamespace=true` is invalid on
Argo CD 2.11: the controller injects a Namespace object, then the project
denies it (`resource :Namespace is not permitted in project expense`). The
AppProject therefore allow-lists **only** `Namespace` as a cluster kind.

## Where the image comes from

W6 D1's `_build-and-push.yml` pushes
`625397071689.dkr.ecr.us-east-1.amazonaws.com/uptimecrew/expense-api:<git-sha>`
(plus a `:main` tag). The overlays therefore set `images[].newName` to that ECR
registry: a bare `uptimecrew/expense-api` resolves to Docker Hub and 404s.
`_bump-config.yml` rewrites only `newTag`.

The dev/staging/prod overlays currently pin `70ff4abd…`, which is the newest
image actually in ECR (`:main` resolves to the same digest). The app repo's
`main` HEAD is ahead of it because that CI run was cancelled at the
`build-test` timeout, so no image exists for the newer commit.

ECR is private, so a k3d cluster needs credentials out-of-band — never in Git:

```bash
# 12-hour token; re-run when pulls start failing with 401
kubectl create secret docker-registry ecr-creds \
  --docker-server=625397071689.dkr.ecr.us-east-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password="$(aws ecr get-login-password --region us-east-1)" \
  -n expense-dev
```

Alternatively, side-load the exact ref (the base Deployment uses
`imagePullPolicy: IfNotPresent`, so containerd serves the local copy):

```bash
k3d image import 625397071689.dkr.ecr.us-east-1.amazonaws.com/uptimecrew/expense-api:70ff4abdf369b6a941fc5db679e504f48c5ad8f2 -c expense-dev
```

The ECR images are `linux/arm64`, built by the self-hosted runner on an Apple
Silicon host, which matches the k3d nodes here. An `amd64` image would start
and immediately exit with `exec format error`.

## Known blocker: Argo CD 2.11.7 vs Kubernetes 1.35

With `ServerSideApply=true`, Argo CD 2.11.7 fails to diff the Deployment on
this cluster:

```
ComparisonError: failed to calculate diff: error calculating structured merge diff:
error building typed value from live resource: .status.terminatingReplicas:
field not declared in schema
```

`status.terminatingReplicas` is a Kubernetes 1.33+ Deployment field that
2.11.7's bundled schema does not know, so every Application sits
`Unknown`/`Progressing`. The assignment pins **both** Argo CD `v2.11.7` and
`ServerSideApply=true`, so the fix is the cluster, not the manifests: recreate
k3d on a Kubernetes version from the 2.11 era
(`k3d cluster create … --image rancher/k3s:v1.29.6-k3s2`). Dropping
`ServerSideApply` would clear the error but breaks a graded requirement.

Docker Desktop also needs more than the default 8 GB before all three envs can
be Healthy at once — dev + staging + prod each run Postgres, Redis, Mongo, and
Kafka alongside the API, and the first attempt evicted pods under
`DiskPressure`/memory pressure.

## Why pin the Argo CD install to `v2.11.7`

`stable` is a moving tag. Pinning the install YAML means every laptop and CI
run the same CRDs, ApplicationSet controller, and notification templates. A
mid-week `stable` bump has broken ApplicationSet generators and notification
ConfigMap keys before.

## Why two project roles instead of cluster-admin for the team

`developers` can sync **dev and staging** only and cannot delete Applications.
`releasers` can sync prod (and run Application actions) but still cannot delete.
Neither role can widen `sourceRepos` / destinations / resource allow-lists —
those live on the AppProject. One `cluster-admin` Slack group would let a
compromised laptop prune prod or point an Application at `kube-system`.

OIDC groups `uptimecrew:expense-engineers` and
`uptimecrew:expense-engineers-releasers` document intent; they are inert until
the shared cluster wires Dex/OIDC (W6 D3).

## Why a matrix ApplicationSet on a single k3d cluster

The inner generators are (env list) × (clusters labelled
`uptimecrew.example.internal/tier=workload`). Today that product is three
Applications on one cluster. Tomorrow a second labelled cluster does not
require a new Application YAML — only a Secret label. A list-only generator
would bake in `https://kubernetes.default.svc` and have to be rewritten for
every cluster.

Sync waves: Application annotation `dev=0`, `staging=1`, `prod=2`.
`CreateNamespace=true` creates the env Namespace (overlay Git must not include
a Namespace object, or the project allow-list fights the sync).

## Why `on-sync-succeeded` is omitted

Every healthy sync would page the deploy channel. Nobody reads that firehose,
and real failures drown. Notifications are `on-sync-failed` and
`on-health-degraded` only, selector `team=expense`. Incoming webhooks are not
Slack Bot tokens, so the ConfigMap uses `service.webhook.slack` with
`url: $slack-token` and recipient `webhook:slack`. An incoming webhook posts
only to the channel it was created for, so there is no `channel:` field to
set: ours is **`#argocd-notifications`** (the brief calls the same team channel
`expense-deploys`).
The URL is Secret `argocd-notifications-secret` key `slack-token`, never Git.

## Task 2 rogue Application (PR evidence)

`kubectl apply -f argocd/applications/expense-api-rogue.yaml` is accepted by the
API server (no validating webhook). The Application controller then sets:

```
InvalidSpecError: application destination server 'https://kubernetes.default.svc'
and namespace 'kube-system' do not match any of the allowed destinations
in project 'expense'
```

The Application was deleted afterward. `kube-system` is not in `spec.destinations`.

## `preserveResourcesOnDeletion` (Task 3)

`ApplicationSet.spec.syncPolicy.preserveResourcesOnDeletion: true`.

Ran 2026-09-15T16:53Z on k3d `expense-dev`:

1. Patched the list generator to drop `staging`.
2. `expense-api-staging` **remained** (`kubectl -n argocd get app` still listed it).
3. Restored `argocd/applicationsets/expense-api-envs.yaml`; the ApplicationSet
   adopted staging again.

Without the flag, removing the list entry would garbage-collect staging.

k3d cannot Healthy-run three full postgres/kafka/mongo stacks. After the
ApplicationSet applied, staging/prod pods were **Evicted**. `expense-api-dev`
was **Synced + Healthy** before the matrix fan-out. The three Application
objects exist (`expense-api-dev|staging|prod`).

## Drift / selfHeal (Task 4)

Ran 2026-09-15T16:53Z:

```
kubectl -n expense-dev scale deployment expense-api --replicas=5
```

`spec.replicas` went `2 → 5 → 3` (HPA `minPods=2` / live pressure). While the
Application status was `Unknown`, the controller **skipped auto-sync**:

```
time="2026-09-15T16:53:34Z" level=info msg="Comparing app state (cluster: https://kubernetes.default.svc, namespace: expense-dev)" application=argocd/expense-api-dev
time="2026-09-15T16:53:34Z" level=info msg="Skipping auto-sync: application status is Unknown" application=argocd/expense-api-dev
time="2026-09-15T16:53:34Z" level=info msg="Reconciliation completed" application=argocd/expense-api-dev dest-namespace=expense-dev
```

Earlier the same day (before ApplicationSet fan-out) `expense-api-dev` was
`Synced` / `Healthy` with both API pods `1/1`. Re-run the scale after staging/prod
are deleted or the node has spare memory to capture a selfHeal that writes
replicas back while status is Synced.

Drift experiment timestamp: **2026-09-15T16:53:34Z**.

## argocd-author Skill audit

`/argocd-author expense-api --namespace-prefix expense --strategy canary --secrets-mode eso`
was not available in this Cursor session (no `argocd-author` skill on disk).
Audit was against the lab AppProject / Application / ApplicationSet snippets
and the Skill quirks called out in the assignment.

| Skill-style output | Decision |
| --- | --- |
| `spec.project: default` on Applications | **Rejected.** Every Application must use `expense` so allow-lists and RBAC apply. `default` is unrestricted. |
| Omit `resources-finalizer.argocd.argoproj.io` | **Rejected.** Without the finalizer, deleting the Application orphans Deployment/Service in the env namespace. |
| Scaffold an Argo Rollouts `Rollout` CR for `--strategy canary` | **Accepted as a W6 D5 stub, not applied today.** Rollouts is not installed. Leave a commented Rollout with `# W6 D5 lands this` rather than applying a CRD the cluster does not have. |

## Apply order (k3d)

Run these from a clone of the GitOps repo (`config-1`), which is where
`argocd/` and `argocd-system/` live after W6 D2.

```bash
# Task 1
kubectl apply -f argocd/projects/expense.yaml -n argocd
kubectl apply -f argocd/applications/expense-api-dev.yaml -n argocd

# Task 2 deny path (expect rejection), then delete
kubectl apply -f argocd/applications/expense-api-rogue.yaml -n argocd
kubectl delete application expense-api-rogue -n argocd --ignore-not-found

# Task 3: label in-cluster destination, then ApplicationSet owns all three envs
kubectl -n argocd label secret -l argocd.argoproj.io/secret-type=cluster \
  uptimecrew.example.internal/tier=workload --overwrite
kubectl apply -f argocd/applicationsets/expense-api-envs.yaml -n argocd
kubectl delete application expense-api-dev -n argocd   # ApplicationSet recreates it

# Task 4 notifications (webhook never committed)
kubectl -n argocd create secret generic argocd-notifications-secret \
  --from-literal=slack-token="$SLACK_WEBHOOK_URL" --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f argocd-system/notifications-cm.yaml -n argocd
kubectl -n argocd rollout restart deploy/argocd-notifications-controller
```

GitOps repo token for CI: GitHub secret `GITOPS_REPO_TOKEN` with `contents: write`
on `AI-Native-2026-08-05-Intuit/manya-bhasin-expense-config` only. It is not set
on the app repo yet, so the `_bump-config` checkout step fails until it is
added; no other job needs it.
