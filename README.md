# CJs-Shipyard — Platform Docs

Architecture diagrams, request-flow diagrams, and design decisions for **CJs-Shipyard**,
a local, production-style **GitOps Kubernetes** platform.

## What it is

CJs-Shipyard is a personal, portfolio-grade platform that runs a small product/inventory service
the way a real shop would: reads and writes handled by separate services (**CQRS-lite**), each
component in its own repository (**poly-repo**), workloads packaged with **Helm**, **Argo CD**
keeping the cluster in sync with what's committed to Git, and a local **k3d** cluster standing in
for a managed Kubernetes environment. It exists to practice and demonstrate the *operational* side of
software — GitOps, progressive delivery, least-privilege data access, and clean service boundaries —
on hardware you already own, with no cloud bill.

> **Honesty note (it's a portfolio piece):** several designed components are not built yet.
> What is **running today** is the read/write API split, PostgreSQL, pgAdmin, Traefik ingress, and
> Argo CD. The **Envoy Gateway edge + central Swagger UI** ([ADR-0007](./docs/adr/0007-envoy-gateway-edge.md),
> [ADR-0008](./docs/adr/0008-central-api-docs.md)) are implemented in `helm-charts`/`argocd` but
> awaiting cluster Instantiation. RabbitMQ, Redis, Vault/External-Secrets, and the observability stack
> are **designed but not deployed**. Diagram 2 below is the source of truth for what is live.

## Tech stack

| Layer | Technology | Status |
|-------|-----------|--------|
| API services | Python 3.12 · FastAPI · SQLAlchemy 2 (async) · Alembic | ✅ Live |
| Database | PostgreSQL 16 (StatefulSet, `local-path` PVC) | ✅ Live |
| DB admin UI | pgAdmin 4 | ✅ Live |
| Packaging | Helm (one chart per deployable) | ✅ Live |
| GitOps / CD | Argo CD (app-of-apps) | ✅ Live |
| CI | GitHub Actions + reusable `ci-templates` | ✅ Live |
| Image registry | GitHub Container Registry (GHCR) | ✅ Live |
| Local cluster | k3d (single-node today) | ✅ Live |
| Ingress / edge | Traefik (k3d built-in) | ✅ Live (until edge cutover) |
| Edge gateway | Envoy Gateway (Gateway API, replaces the Nginx `gateway-proxy` plan — ADR-0007) | 🟡 Implemented, cutover pending |
| API docs | Swagger UI (central page at `api.localhost` — ADR-0008) | 🟡 Implemented, cutover pending |
| Message queue | RabbitMQ (async write path) | 🟡 Planned |
| Cache | Redis (read-side cache) | 🟡 Planned |
| Secrets | Vault + External Secrets Operator | 🟡 Planned (manual today) |
| Observability | OpenTelemetry · Loki · Grafana | 🟡 Planned |

## Diagrams

### 1. Designed client traffic flow (target architecture)

The full intended request path. Several nodes here are **not deployed yet** — see Diagram 2 for the
honest live view.

```mermaid
flowchart LR
    client([Client])
    envoy["Envoy Gateway (api.localhost)"]
    docs["Swagger UI (central API docs)"]
    write["write-api (commands)"]
    read["read-api (queries)"]
    mq{{RabbitMQ}}
    consumer["Consumer / projector"]
    redis[("Redis cache")]
    pg[("PostgreSQL")]

    client --> envoy
    envoy -->|"/write/*"| write
    envoy -->|"/read/*"| read
    envoy -->|"/"| docs
    write -->|publish event| mq
    mq --> consumer
    consumer --> pg
    write --> pg
    read --> pg
    read <-->|cache| redis
```

### 2. Live status (what is actually running)

Verified against the running k3d cluster. The legend distinguishes what is deployed from what is
only designed.

```mermaid
flowchart TB
    classDef running fill:#d4f7d4,stroke:#2e7d32,color:#1b5e20;
    classDef planned fill:#f5f5f5,stroke:#9e9e9e,color:#616161,stroke-dasharray:5 5;
    classDef partial fill:#fff4cc,stroke:#f9a825,color:#7f6000;

    client([Client / browser])
    traefik["Traefik ingress (*.localhost)"]
    write[write-api]
    read[read-api]
    pgadmin[pgAdmin]
    pg[("PostgreSQL 16")]
    argo[[Argo CD]]
    secrets["Secrets: manual kubectl apply"]

    envoy["Envoy Gateway + central Swagger UI (cutover pending)"]
    mq{{RabbitMQ}}
    redis[("Redis")]
    obs["Observability: OTel / Loki / Grafana"]

    client --> traefik
    traefik --> write
    traefik --> read
    traefik --> pgadmin
    write --> pg
    read --> pg
    pgadmin --> pg
    argo -. reconciles .-> write
    argo -. reconciles .-> read
    argo -. reconciles .-> pg
    argo -. reconciles .-> pgadmin
    secrets -. consumed by .-> write
    secrets -. consumed by .-> read
    secrets -. consumed by .-> pg

    subgraph legend["Legend"]
        l1["Running"]:::running
        l2["Designed — not deployed"]:::planned
        l3["Partial / stopgap"]:::partial
    end

    class client,traefik,write,read,pgadmin,pg,argo running;
    class mq,redis,obs planned;
    class secrets,envoy partial;
```

### 3. GitOps delivery flow

How a code change becomes a running pod, with no manual `kubectl apply` of workloads.

```mermaid
flowchart LR
    dev([Developer])
    repo["Service repo (api/)"]
    gha["GitHub Actions (build-image.yml)"]
    tmpl[["ci-templates: docker-build.yml + helm-bump.yml"]]
    ghcr[("GHCR image :short-sha")]
    helm["helm-charts values.yaml (image.tag)"]
    argo{{Argo CD}}
    k3d[("k3d cluster (shipyard ns)")]

    dev -->|git push| repo
    repo -->|triggers| gha
    gha -->|calls| tmpl
    tmpl -->|build & push| ghcr
    tmpl -->|commit tag bump| helm
    argo -->|detects change| helm
    argo -->|sync / apply| k3d
    ghcr -. pulled by .-> k3d
```

## Repositories

Each sibling repo is independent (poly-repo). The org is `CJs-Shipyard`.

| Repo | Role |
|------|------|
| `api` | FastAPI `read-api` + `write-api`, shared SQLAlchemy models, Alembic migrations |
| `helm-charts` | Helm charts for every deployable — the single source of truth Argo CD deploys |
| `argocd` | GitOps wiring: app-of-apps, `Application`/`AppProject` manifests, day-0 bootstrap |
| `ci-templates` | Reusable GitHub Actions workflows (`docker-build`, `helm-bump`) |
| `postgres` | Local Postgres + pgAdmin via docker-compose; provisions per-service DB roles |
| `gateway-proxy` | Nginx edge gateway — *superseded by Envoy Gateway ([ADR-0007](./docs/adr/0007-envoy-gateway-edge.md)); will not be built* |
| `rabbitmq` | Message queue — *planned* |
| `redis-cluster` | Read cache — *planned* |

## Where to look next

- [Architecture overview](./docs/architecture.md) — components, responsibilities, namespaces
- [Local setup guide](./docs/local-setup.md) — stand it up in your own cluster
- [Observability](./docs/observability.md) — the planned OTel / Loki / Grafana approach
- [Conventions](./docs/conventions.md) — how to contribute changes
- [Architecture Decision Records](./docs/adr/README.md) — why the platform is shaped this way
- [Docs index](./docs/README.md)
