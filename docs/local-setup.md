# Local setup

How another dev stands CJs-Shipyard up in their own cluster. This is the GitOps path; for a
chart-only quick loop see `helm-charts/README.md` (Path B).

## Prerequisites

| Tool | Version (known-good) | Why |
|------|----------------------|-----|
| Docker | recent | k3d runs the cluster in Docker |
| k3d | 5.x | local Kubernetes (k3s in Docker) |
| kubectl | 1.29+ | talk to the cluster |
| helm | 3.x+ | render/validate charts (Argo deploys them for you) |
| argocd CLI | optional | nicer than `kubectl` for app status |

You also need a GitHub **PAT** with `read:packages` to pull the private GHCR images, and the
gitignored `secret.yaml` files for each chart (DB creds, pgAdmin login).

## 1. Create the cluster

```bash
k3d cluster create shipyard \
  --k3s-arg "--disable=traefik@server:0" \
  --port "80:80@loadbalancer" \
  --port "443:443@loadbalancer"

kubectl cluster-info
kubectl get nodes
```

The `80:80` mapping is what makes the `*.localhost` hostnames resolve. Traefik is **disabled**
because Envoy Gateway is the platform edge ([ADR-0007](./adr/0007-envoy-gateway-edge.md)) — if
Traefik is running it claims host port 80 first via its LoadBalancer Service, and Envoy's never
binds. If you have a pre-ADR-0007 cluster (Traefik-based), recreate it: `k3d cluster delete
shipyard` first — **this wipes the Postgres PVC**; the schema is rebuilt by Alembic, but back up
any data you care about.

## 2. Bootstrap Argo CD (day-0, run once)

From the `argocd/` repo:

```bash
# Argo CD itself (umbrella chart wrapping upstream argo-cd)
helm dependency build install/argo-cd
helm install argocd install/argo-cd -n argocd --create-namespace

# Let Argo pull the private CJs-Shipyard repos (fill in a PAT first; gitignored)
cp install/repo-creds.example.yaml install/repo-creds-secret.yaml
kubectl apply -f install/repo-creds-secret.yaml
```

## 3. Provide secrets (manual for now)

Vault/ESO is not wired up yet, so the app secrets and the GHCR pull secret are applied by hand:

```bash
kubectl create namespace shipyard

# App secrets (from the gitignored per-chart manifests in helm-charts)
kubectl apply -f ../helm-charts/charts/postgres/secrets/secret.yaml  -n shipyard
kubectl apply -f ../helm-charts/charts/write-api/secrets/secret.yaml -n shipyard
kubectl apply -f ../helm-charts/charts/read-api/secrets/secret.yaml  -n shipyard
kubectl apply -f ../helm-charts/charts/pgadmin/secrets/secret.yaml   -n shipyard

# GHCR pull secret (private images)
kubectl -n shipyard create secret docker-registry github-cjs-shipyard-creds \
  --docker-server=ghcr.io \
  --docker-username=<github-user> \
  --docker-password=<pat-with-read:packages>
```

> Do not commit any of these. The `*secret.yaml` files are gitignored across the repos.

## 4. Point Argo at the charts (start the app-of-apps)

```bash
kubectl apply -f root-app.yaml
```

Argo recursively reconciles everything under `argocd/apps/` — `envoy-gateway` and Postgres first
(wave 0), then the Gateway and the APIs (wave 1), then `swagger-ui` (wave 2). Adding a service
later is just dropping a new `Application` manifest in; no edit here.

Once the `envoy-gateway` app is Synced (Gateway API CRDs exist), re-run the install chart to add
the Argo CD UI route — it is capability-gated, so the day-0 `helm install` skipped it:

```bash
helm upgrade argocd install/argo-cd -n argocd
```

## 5. Verify

```bash
kubectl get applications -n argocd          # all should be Synced/Healthy
kubectl get pods -n shipyard                # postgres, write-api, read-api, pgadmin, swagger-ui Running
kubectl get pods -n envoy-gateway-system    # envoy-gateway controller + envoy proxy Running
kubectl get gateway,httproute -n shipyard   # Gateway Programmed, routes Accepted
```

Then browse:

- http://api.localhost — central Swagger UI (dropdown: write-api / read-api)
- http://api.localhost/write/docs and http://api.localhost/read/docs — per-service docs
- http://pgadmin.localhost
- http://argocd.localhost — Argo CD UI (after the `helm upgrade` in step 4)

## Current limitations / not deployed yet

- **Single-node** k3d cluster (no agent nodes).
- **No RabbitMQ or Redis** — writes are synchronous and reads uncached.
- **No observability stack** — the `observability` namespace isn't created yet.
- **Secrets are manual** — Vault + External Secrets Operator are scaffolded but inert.
- The `postgres` Argo app may show **OutOfSync/Healthy** (benign StatefulSet drift).
