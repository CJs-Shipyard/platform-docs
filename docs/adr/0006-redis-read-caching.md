# ADR-0006: Redis for read caching

- **Status:** Proposed (not implemented)
- **Date:** 2026-06-06

## Context

`read-api` currently hits Postgres on every query. For hot, read-heavy paths a cache reduces latency
and database load.

## Decision

Introduce **Redis** as a read-side cache fronting `read-api` queries (cache-aside: check Redis, fall
back to Postgres on miss, populate on read). Invalidation is driven by writes -- ideally off the
RabbitMQ events from [ADR-0005](./0005-rabbitmq-async-writes.md). The `redis-cluster` repo exists as
a placeholder.

## Consequences

- ✅ Lower read latency and reduced Postgres load on frequently requested keys.
- ✅ Pairs naturally with event-driven invalidation once RabbitMQ lands.
- ⚠️ Cache invalidation is the hard part -- stale reads if writes and invalidation drift.
- ⚠️ Another stateful kubernetes resource to run and secure.
- 🟡 **Not built:** no Redis deployment, no chart, no Argo app, and no cache code in `read-api` yet.
