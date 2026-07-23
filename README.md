# CJs-Shipyard — Platform Docs

Architecture diagrams, request-flow diagrams, and design decisions for **CJs-Shipyard**,
a local, production-style **GitOps Kubernetes** platform.

## What it is

CJs-Shipyard is a personal, portfolio-grade platform that runs a small product/inventory service
where reads and writes are handled by separate services (**CQRS-lite**) and each component of
the platform has its own repository (**poly-repo** structure).
- Kubernetes workloads are packaged with **Helm**.
- **Argo CD** keeps the cluster in sync with what's committed to Github.
- A local **k3d** cluster is standing in for a managed Kubernetes environment like EKS. 

The project exists to practice and demonstrate the *operational* side of software — GitOps, 
progressive delivery, least-privilege data access, and clean service boundaries.

> **Honesty note (it's a portfolio piece and playground):** some designed components are not built yet.
> What is **running today** is the read/write API split, PostgreSQL, pgAdmin, the **Envoy Gateway
> edge with the central Swagger UI**, and Argo CD. RabbitMQ, Redis,
> Vault/External-Secrets, and the observability stack are **designed but not deployed**.
> The colour-coded architecture diagram below is the source of truth for what is live.

## Tech stack & repositories

This table defines each layer/component of the platform, which repo owns it (if any), and whether it is
actually running in my cluster.

| Layer / component | Technology | Repo | Status |
|---|---|---|---|
| API services | Python 3.12 · FastAPI · SQLAlchemy 2 (async) · Alembic | `api` | ✅ Live |
| Database | PostgreSQL 16 (StatefulSet, `local-path` PVC) | `postgres` | ✅ Live |
| DB admin UI | pgAdmin 4 | `postgres` | ✅ Live |
| Ingress / edge | Envoy Gateway (Gateway API) | `helm-charts` | ✅ Live |
| API docs | Swagger UI (central page at `api.localhost`) | `api` | ✅ Live |
| Packaging | Helm (one chart per deployable) | `helm-charts` | ✅ Live |
| GitOps / CD | Argo CD (app-of-apps) | `argocd` | ✅ Live |
| CI | GitHub Actions + reusable workflows | `ci-templates` | ✅ Live |
| Image registry | GitHub Container Registry (GHCR) | — | ✅ Live |
| Local cluster | k3d (single-node today) | — | ✅ Live |
| Message queue | RabbitMQ (async write path) | `rabbitmq` | 🟡 Planned |
| Cache | Redis (read-side cache) | `redis-cluster` | 🟡 Planned |
| Secrets | Vault + External Secrets Operator | — | 🟡 Planned (manual today) |
| Observability | OpenTelemetry · Loki · Grafana | — | 🟡 Planned |

**Poly-repo** — every component is its own repository under the `CJs-Shipyard` org. Rows with a
`—` are platform / SaaS layers (registry, cluster, secrets, observability) with no dedicated repo.

## Diagrams

### 1. Architecture & live status

Overarching view of the platform — **color defines reality.** Apart from the client, every
component here lives in the local k3d `shipyard` cluster and is delivered by Argo CD;
Diagram 2 shows how.

```mermaid
flowchart TB
    classDef live fill:#d4f7d4,stroke:#2e7d32,color:#1b5e20;
    classDef planned fill:#f0f0f0,stroke:#9e9e9e,color:#616161,stroke-dasharray:5 5;
    classDef partial fill:#fff4cc,stroke:#f9a825,color:#7f6000;
    classDef ctrl fill:#e3f2fd,stroke:#1565c0,color:#0d47a1;

    subgraph legend["Legend"]
        direction LR
        lL["Live"]:::live
        lP["Designed, not deployed"]:::planned
        lS["Partial / stopgap"]:::partial
        lC["Control plane"]:::ctrl
    end

    secrets["Secrets · manual kubectl apply<br/>consumed by all live services"]:::partial
    client([Client / browser]):::live

    envoy["Envoy Gateway<br/>api · pgadmin · argocd .localhost"]:::live
    docs["Swagger UI · central API docs<br/>shows both write-api & read-api"]:::live
    write["write-api<br/>commands"]:::live
    read["read-api<br/>queries"]:::live
    pgadmin["pgAdmin"]:::live
    pg[("PostgreSQL 16")]:::live

    redis[("Redis cache")]:::planned
    mq{{RabbitMQ}}:::planned
    consumer["Consumer / projector"]:::planned

    argo[[Argo CD]]:::ctrl
    write -.->|publish event| mq

    client -->|HTTPS| envoy
    envoy -->|"api.localhost/read/*"| read
    envoy -->|"api.localhost/write/*"| write
    envoy -->|"api.localhost/docs"| docs
    envoy -->|pgadmin.localhost| pgadmin
    envoy -->|argocd.localhost| argo
    write -->|persist command| pg
    read -->|query on cache miss| pg
    pgadmin -->|admin queries| pg

    read -.->|check cache| redis
    mq -.-> consumer
    consumer -.->|update read model| pg

    %% layout only, invisible: dagre's network-simplex ranker minimises total edge
    %% length, so without these it slides read-api and redis off their intended rows.
    %% read ~~~ mq pins read-api level with write-api; mq ~~~ redis pins redis level
    %% with consumer.
    read ~~~ mq
    mq ~~~ redis

    linkStyle 1,2,3,4,5,6,7,8,9 stroke:#2e7d32,stroke-width:2px;
    linkStyle 0,10,11,12 stroke:#9e9e9e,stroke-width:1.5px,stroke-dasharray:5 5;
    linkStyle 13,14 stroke:none,stroke-width:0px,fill:none;
```

**Edges** (the legend covers node colors):

- **Solid green** — live request / data path
- **Grey dashed** — designed, not deployed

The planned Observability stack (OTel · Loki · Grafana) is omitted here for clarity; see the
table above and [observability.md](./docs/observability.md).

### 2. GitOps delivery flow

How a code change becomes a running pod — no manual `kubectl apply` of workloads. Steps are
numbered 1–8 in causal order. The flow forks at `ci-templates` into an image branch (4, 8) and
a chart branch (5, 6), which reconverge in the cluster. Same colour language as Diagram 1:
green = runs today, blue = Argo CD (control plane); dashed green = image pull.

```mermaid
flowchart TB
    classDef live fill:#d4f7d4,stroke:#2e7d32,color:#1b5e20;
    classDef ctrl fill:#e3f2fd,stroke:#1565c0,color:#0d47a1;

    dev([Developer]):::live

    repo["service repo<br/>(api/)"]:::live
    gha["repo-local GitHub Actions<br/>build-image.yml"]:::live
    tmpl[["ci-templates repo<br/>docker-build & helm-bump Actions"]]:::live
    ghcr[("GHCR<br/>image :short-sha")]:::live
    helm["helm-charts repo<br/>values.yaml (image.tag)"]:::live

    argo{{"Argo CD<br/>k3d · shipyard"}}:::ctrl
    k3d[("workloads<br/>k3d · shipyard")]:::live

    dev -->|1. git push| repo
    repo -->|2. triggers| gha
    gha -->|3. calls| tmpl
    tmpl -->|4. build & push| ghcr
    tmpl -->|5. commit tag bump| helm
    argo -->|6. detects change| helm
    argo -->|7. sync / apply| k3d
    ghcr -.->|8. image pull| k3d

    subgraph legend["Legend"]
        direction LR
        gL["Live / runs today"]:::live
        gC["Control plane"]:::ctrl
    end

    linkStyle 0,1,2,3,4 stroke:#2e7d32,stroke-width:2px;
    linkStyle 5,6 stroke:#1565c0,stroke-width:2px;
    linkStyle 7 stroke:#2e7d32,stroke-width:1.5px,stroke-dasharray:5 5;
```

## Where to look next

- [Architecture overview](./docs/architecture.md) — components, responsibilities, namespaces
- [Local setup guide](./docs/local-setup.md) — stand it up in your own cluster [beta]
- [Observability](./docs/observability.md) — the planned OTel / Loki / Grafana approach
- [Conventions](./docs/conventions.md) — how to contribute changes
- [Architecture Decision Records](./docs/adr/README.md) — why the platform is shaped this way
- [Docs index](./docs/README.md)
