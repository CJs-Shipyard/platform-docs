# ADR-0001: CQRS-lite read/write split

- **Status:** Accepted (async propagation still pending — see ADR-0005/0006)
- **Date:** 2026-06-06

## Context

A single CRUD service is simple and intuitive, but it couples read and write scaling, permissions, and
deploy cadence. This is a portfolio platform -- demonstrating clean command/query separation is part
of the point.

This idea was inspired by this great NDC conference talk https://youtu.be/WRg13Ze_UpY?si=kog6baOhyYAgALgx

## Decision

Split the API into two FastAPI services over **one shared Postgres**:

- `write-api` -- commands; owns the schema (the only service that runs `alembic upgrade head`);
  connects as a read/write role.
- `read-api` -- queries; never migrates, never commits; connects as a read-only role.

Schema is defined **once** in a shared `shared/models` package both services import. "CQRS-lite"
because the read and write sides share a database rather than maintaining separate read models.

## Consequences

- ✅ Independent scaling, deploy, and least-privilege DB access per side.
- ✅ No model duplication; a schema change is picked up by both services from one source.
- ⚠️ Shared DB means the read side sees writes immediately but also shares write contention. The
  fuller CQRS shape (async event propagation, separate read model) is **not built yet** -- writes are
  synchronous to Postgres. The planned RabbitMQ/Redis work will enable asynchronicity. 
