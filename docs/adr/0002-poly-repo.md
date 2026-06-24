# ADR-0002: Poly-repo layout

- **Status:** Accepted
- **Date:** 2026-06-06

## Context

The platform contains several distinct concerns: app code, Helm charts, GitOps wiring, CI templates,
infra. They could live in one monorepo or many.

## Decision

Use a **poly-repo** layout -- one repo per concern (`api`, `helm-charts`, `argocd`, `ci-templates`,
`postgres`, `gateway-proxy`, `rabbitmq`, `redis-cluster`, `platform-docs`) under the `CJs-Shipyard`
org. Each has its own history, README.

## Consequences

- ✅ Mirrors how many real orgs are structured; clean ownership and per-repo CI/access.
- ✅ Forces explicit contracts between repos (e.g. CI bumps `image.tag` in `helm-charts`) instead of
  implicit local coupling.
- ⚠️ A change can span repos (code → chart → Argo app), so the cross-repo flow has to be documented
  (see [conventions](../conventions.md)).
- ⚠️ Relative filesystem links across repos don't resolve on GitHub -- refer to siblings by name/URL.
