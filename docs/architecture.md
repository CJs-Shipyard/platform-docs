# Architecture overview

CJs-Shipyard is a small **CQRS-lite** product/inventory platform deployed via **GitOps** onto a
local **k3d** cluster. The pieces are deliberately split across repos and packaged as Helm charts so
the *operational* shape mirrors a real environment.

> Status legend: ✅ live · 🟡 planned. See the root [README](../README.md) Diagram 2 for the
> current live view.

## Request flow (today)

```
client → Envoy Gateway (api.localhost)
           ├── /write/*  → write-api   (prefix stripped, ROOT_PATH=/write)
           ├── /read/*   → read-api    (prefix stripped, ROOT_PATH=/read)
           └── /         → swagger-ui  (central docs: dropdown of both OpenAPI specs)
         Envoy Gateway (pgadmin.localhost) → pgAdmin
         Envoy Gateway (argocd.localhost)  → Argo CD UI
```

The edge is Envoy Gateway ([ADR-0007](./adr/0007-envoy-gateway-edge.md)); central docs are
[ADR-0008](./adr/0008-central-api-docs.md). The designed flow further adds a RabbitMQ async write
path and a Redis read cache (see [ADR-0005](./adr/0005-rabbitmq-async-writes.md) and
[ADR-0006](./adr/0006-redis-read-caching.md)).

## Components

### `write-api` ✅
FastAPI command side — `POST /products`, `PATCH /products/{id}/inventory`, `DELETE /products/{id}`.
Commits synchronously to Postgres (for now). **Owns the schema**: it is the only service that runs
`alembic upgrade head` (on container startup), connecting as the read/write `write_api` role.

### `read-api` ✅
FastAPI query side — `GET /products`, `GET /products/{id}`. Read-only; never migrates and never
commits. Connects as the least-privilege `read_api` role. Imports the **same** `shared/models`
package as `write-api`, so there is no schema duplication.

### Shared data model ✅
ORM models live once in `api/shared/models` (currently `Product` + `Inventory`). Migrations are an
Alembic project at the `api/` repo root. Both services build from the same package; only `write-api`
applies migrations.

### `postgres` ✅
PostgreSQL 16 StatefulSet with a `local-path` PVC. An initdb script provisions two login roles —
`write_api` (USAGE/CREATE on `public`, owns its tables) and `read_api` (read-only, auto-granted
SELECT on future tables via `ALTER DEFAULT PRIVILEGES`). This is the least-privilege seam between
the two services.

### `pgadmin` ✅
pgAdmin 4 web UI for inspecting the database. Persistence off by default (local ergonomics).

### Ingress / edge ✅
**Envoy Gateway** ([ADR-0007](./adr/0007-envoy-gateway-edge.md)): the controller runs in
`envoy-gateway-system`; the shared `shipyard-gateway` Gateway (listeners for `api.localhost`,
`pgadmin.localhost`, and `argocd.localhost`) lives in `shipyard`, and each service chart ships an
`HTTPRoute` that attaches to it. Traefik is disabled at cluster create; the previously designed
Nginx `gateway-proxy` is **superseded** and will not be built. The Argo CD UI route is the one
`HTTPRoute` outside `shipyard` — it ships with the day-0 install chart (`argocd/install/argo-cd`)
and is admitted by a namespace-selector on the `http-argocd` listener.

### `swagger-ui` ✅
Central API docs ([ADR-0008](./adr/0008-central-api-docs.md)): one Swagger UI at
`http://api.localhost/` with a dropdown for the write-api and read-api OpenAPI specs, served
same-origin via the gateway's `/write` and `/read` path routes.

### Argo CD ✅
Runs the GitOps loop. A root **app-of-apps** watches `argocd/apps/` and reconciles every
`Application`/`AppProject` it finds. Sync waves order startup: `postgres` (wave 0) before the APIs
(wave 1), because the APIs connect as roles Postgres' initdb creates.

### Planned components 🟡
- **RabbitMQ** — async write path (`write-api` → queue → consumer/projector).
- **Redis** — read-side cache for `read-api`.
- **CAS server (PKI auth)** — mTLS at the Envoy edge, client cert forwarded via
  `X-Forwarded-Client-Cert` and authorized through `extAuth` (design in
  [ADR-0007](./adr/0007-envoy-gateway-edge.md)).
- **Vault + External Secrets Operator** — replace the manual secret step with GitOps secrets.
- **Observability** — OpenTelemetry SDK in app code, Loki for logs, Grafana for dashboards.

## Namespaces

| Namespace | Contents | Status |
|-----------|----------|--------|
| `shipyard` | `write-api`, `read-api`, `postgres`, `pgadmin`, `swagger-ui`, `shipyard-gateway` | ✅ live |
| `envoy-gateway-system` | Envoy Gateway controller (`edge` AppProject) | ✅ live |
| `observability` | metrics / logs / traces stack | 🟡 reserved (AppProject exists; namespace not yet created) |
| `argocd` | Argo CD control plane | ✅ live |

## Known reconciliation notes

- Secrets are applied **by hand** today (`kubectl apply` of gitignored `secret.yaml` manifests).
  The Vault/ESO scaffolding is committed but inert until the CRDs are installed.
