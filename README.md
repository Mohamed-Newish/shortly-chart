# Shortly — Helm chart

A URL shortener with click analytics, packaged as a Helm chart and deployed to Kubernetes either with `helm install` or, the way it is actually run, by **ArgoCD reconciling this repository**.

```
Ingress (Traefik/NGINX)
  ├── /         →  frontend   Deployment ×2   (Nginx, static)
  └── /api      →  api        Deployment ×4   (Node/Express)
                                  │
                        ┌─────────┴─────────┐
                     db  StatefulSet     redis  StatefulSet
                     (Postgres 16,        (Redis 7,
                      1Gi PVC)             512Mi PVC)
```

## What it deploys

| Object | Name | Notes |
| --- | --- | --- |
| Deployment | `api` | Node/Express, port 3000, liveness `/health` + readiness `/ready` + startup probe, non-root `securityContext`, dedicated ServiceAccount |
| Deployment | `frontend` | Nginx serving the static UI on port 80 |
| StatefulSet | `db` | Postgres 16 with a headless Service for stable `db-0.db` DNS, `volumeClaimTemplates` → 1Gi PVC |
| StatefulSet | `redis` | Redis 7, 512Mi PVC |
| Service | `api`, `frontend`, `db`, `redis` | `db` is headless (`clusterIP: None`) |
| Ingress | `shortly` | `/` → frontend, `/api` and `/health` → api |
| ConfigMap | `api-config` | `PORT`, `REDIS_URL`, `DB_HOST`, `DB_NAME` |
| Secret | `api-secrets` | `DB_USER`, `DB_PASSWORD` via `stringData` |
| ServiceAccount | `api-sa` | `automountServiceAccountToken: false` — the app never calls the Kubernetes API |

## Install

```bash
helm lint .

# render locally first — never install to find out
helm template . | grep -E "^(kind|  name):"

helm install shortly . -n shortly --create-namespace

kubectl wait --for=condition=Ready pod --all -n shortly --timeout=180s
```

The images are public on Docker Hub (`obito991/shortly-api`, `obito991/shortly-frontend`), so no registry credentials or `imagePullSecrets` are needed. Postgres and Redis come from their official images.

If your cluster exposes the ingress controller on `localhost:8080`:

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8080/

curl -s -X POST http://localhost:8080/api/links -H 'Content-Type: application/json' -d '{"url":"https://helm.sh/docs/"}'

curl -s http://localhost:8080/api/links
```

## Values worth knowing

| Key | Default | What it does |
| --- | --- | --- |
| `api.image` / `frontend.image` | `obito991/shortly-*:v1` | fully-qualified, so any cluster can pull them |
| `api.replicas` | `4` | frontend has its own `frontend.replicas` |
| `api.resources` | 50m/64Mi → 250m/128Mi | requests and limits, rendered with `toYaml` |
| `postgres.storage` / `redis.storage` | `1Gi` / `512Mi` | PVC sizes; `storageClassName` is deliberately unset so the cluster default is used |
| `config.dbName` | `shortly` | becomes `DB_NAME` in the ConfigMap |
| `secrets.dbUser` / `secrets.dbPassword` | `shortly` / `change-me` | **placeholders** — see below |
| `api.autoscale.*`, `worker.*` | — | declared but unused; there is no HPA or CronJob template yet |

### Secrets

`values.yaml` holds a placeholder password only. Pass the real one at install time so it never reaches Git:

```bash
helm install shortly . -n shortly --create-namespace --set secrets.dbPassword="$(openssl rand -base64 24)"
```

Note that Postgres reads `POSTGRES_PASSWORD` **only when it initialises an empty data directory**. Changing the Secret later does not change the database's password — the PVC already exists. Either rotate with `ALTER USER`, or delete the PVC for a clean slate.

## Per-environment values

`values-dev.yaml` holds only the overrides, layered on top of `values.yaml` (later file wins, and `--set` beats both):

```bash
helm install shortly-dev . -n dev --create-namespace -f values-dev.yaml
```

One chart, two independent releases: 4 api replicas in `shortly`, 1 in `dev`.

> Both copies declare an Ingress with no `host:`, so they compete for the same routes. Give one of them a host, or disable the Ingress for dev, if you want to reach both.

## GitOps with ArgoCD

This chart is designed to be reconciled rather than installed by hand. ArgoCD renders it with `helm template` and owns the resulting objects — there is **no Helm release** in the cluster, so `helm list` stays empty.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: shortly
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/Mohamed-Newish/shortly-chart.git
    targetRevision: main
    path: .
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: shortly
  syncPolicy:
    automated:
      prune: true          # objects removed from Git are deleted from the cluster
      selfHeal: true       # drift in the cluster is reverted to what Git says
    syncOptions:
      - CreateNamespace=true
```

With `selfHeal: true`, changing the cluster by hand does not stick:

| what you do | what ArgoCD does |
| --- | --- |
| `kubectl scale deploy/api --replicas=1` | restores the replica count from Git, within seconds |
| edit a value in `api-config` | reverts it to the committed value |
| `kubectl delete svc/frontend` | recreates it (with a new ClusterIP) |
| delete a template from Git | `prune` removes the object from the cluster |

Deploying is then a commit:

```bash
sed -i 's/replicas: 4/replicas: 6/' values.yaml

git commit -am "api: scale to 6"

git push
```

## Notes

- `db` is a StatefulSet with a **headless** Service so replicas get stable identities (`db-0.db`). The app itself just uses `DB_HOST=db`.
- Short DNS names are namespace-relative, so the same chart in two namespaces wires each api to its own database with no config change.
- Deleting a release leaves the PVCs behind on purpose. Remove them explicitly for a clean slate:
  `kubectl delete pvc -n shortly --all`
