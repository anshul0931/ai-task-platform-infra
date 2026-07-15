# AI Task Processing Platform — Infrastructure Repository

Kubernetes manifests + ArgoCD Application definition for the AI Task
Processing Platform. Designed for k3s but works on any standard Kubernetes
cluster.

## Contents

```
k8s/
  00-namespace.yaml     Dedicated namespace
  01-secrets.yaml        Secret template (JWT, internal API key, Mongo creds)
  02-configmap.yaml       Non-secret shared config
  03-mongo.yaml             MongoDB StatefulSet + headless Service
  04-redis.yaml               Redis Deployment + Service
  05-backend.yaml                Backend Deployment + Service
  06-worker.yaml                    Worker Deployment (horizontally scalable)
  07-frontend.yaml                    Frontend Deployment + Service
  08-ingress.yaml                       Ingress routing / and /api
  09-hpa.yaml                             HPA for backend + worker
argocd/
  application.yaml       ArgoCD Application (auto-sync + self-heal)
```

Every Deployment includes `resources.requests`/`limits`, `readinessProbe`,
and `livenessProbe`, and runs as non-root (`securityContext.runAsNonRoot`),
per the assignment's Kubernetes requirements checklist.

## First-time cluster setup

```bash
# 1. Create the namespace and load secrets (EDIT 01-secrets.yaml first —
#    replace every REPLACE_WITH_* placeholder with a real generated value)
kubectl apply -f k8s/00-namespace.yaml
kubectl apply -f k8s/01-secrets.yaml
kubectl apply -f k8s/02-configmap.yaml

# 2. Everything else
kubectl apply -f k8s/

# 3. Point the image references at your real registry before applying, or
#    let CI (in the application repo) do it automatically via the GitOps
#    flow below.
```

## Installing ArgoCD (one-time, per cluster)

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Access the UI (port-forward for local/dev clusters)
kubectl port-forward svc/argocd-server -n argocd 8081:443

# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

Then register this application:

```bash
kubectl apply -f argocd/application.yaml
```

Edit `argocd/application.yaml`'s `repoURL` to point at wherever you actually
host this infrastructure repository before applying.

## GitOps flow

1. CI in the **application repository** builds/pushes images and then
   commits updated image tags into `k8s/*.yaml` **in this repo**.
2. ArgoCD (configured via `argocd/application.yaml` with
   `syncPolicy.automated: { prune: true, selfHeal: true }`) detects the
   commit and automatically reconciles the cluster to match — no manual
   `kubectl apply` needed after initial setup.
3. **Capture a screenshot of the ArgoCD dashboard** (`https://localhost:8081`
   if port-forwarded, showing the `ai-task-platform` Application as
   `Synced`/`Healthy`) for the submission deliverables — this can't be
   generated ahead of time since it requires an actual running cluster.

## Notes

- `01-secrets.yaml` is a **template**. Never commit real secret values to
  source control — in a real deployment, generate the Secret imperatively
  (`kubectl create secret generic ...`) or use a secrets manager (Sealed
  Secrets, External Secrets Operator, Vault) instead of committing
  `stringData` directly.
- Mongo and Redis run as single replicas here for simplicity. See
  `../ARCHITECTURE.md` §5 for the production HA recommendation (persistent
  volume + AOF for Redis; replica set for Mongo).
- For staging vs. production environments, introduce a Kustomize overlay
  structure (`base/` here + `overlays/staging`, `overlays/production`) — see
  `../ARCHITECTURE.md` §6.
