# ADR-0005: RabbitMQ for async writes

- **Status:** Proposed (not implemented)
- **Date:** 2026-06-06

## Context

Today `write-api` commits synchronously to Postgres and `read-api` reads the same database. To move
toward a fuller CQRS shape -- and to absorb write spikes and decouple side effects — writes should be
able to propagate asynchronously.

## Decision

Introduce **RabbitMQ**: `write-api` publishes a domain event after committing; a consumer/projector
processes events asynchronously (e.g. updating a read model, triggering downstream work). The
`rabbitmq` repo exists as a placeholder for this.

## Consequences

- ✅ Decouples writes from downstream processing; smooths spikes; enables real read/write model
  separation later.
- ✅ Natural seam for future event-driven features.
- ⚠️ Adds a broker to run and reason about (delivery guarantees, idempotency, ordering).
- ⚠️ Introduces eventual consistency between the write commit and it being reflected on the read-side.
- 🟡 **Not built:** no broker, no chart, no Argo app, and no publish code in `api/` yet.
