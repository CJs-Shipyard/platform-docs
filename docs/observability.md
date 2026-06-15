# Observability

> **Status: 🟡 planned.** Nothing in this doc is deployed yet. The `observability` namespace is
> reserved (an Argo CD `AppProject` exists) but the namespace itself isn't created and no workloads
> run. This is the intended shape, written down early so it can be filled in later.

## Intended approach

Three signals, kept in the dedicated `observability` namespace, separate from `shipyard` so their
RBAC and blast radius stay scoped:

| Signal | Tool | Where it's emitted |
|--------|------|--------------------|
| Traces | OpenTelemetry SDK → OTel Collector | embedded in `read-api` / `write-api` app code |
| Logs | Loki (via Promtail/agent) | container stdout |
| Metrics | Prometheus (later) | app + cluster |
| Dashboards | Grafana | queries Loki / Prometheus / traces |

## Principles

- **Instrumentation lives in app code.** The OTel SDK is added in `api/`, not bolted on at the
  cluster edge — so traces carry real application context.
- **Deployed the same way as everything else.** Charts in `helm-charts/`, an Argo `Application`
  under `argocd/apps/observability/`, reconciled by the root app-of-apps. No special path.
- **Scoped namespace.** Everything lands in `observability`, governed by its own `AppProject`.

## To bring it online (future)

1. Add OTel SDK + exporter config to the API services in `api/`.
2. Add charts (OTel Collector, Loki, Grafana) under `helm-charts/charts/`.
3. Drop `Application` manifests into `argocd/apps/observability/`.
4. Update the root [README](../README.md) Diagram 2 to move these from planned → running.
