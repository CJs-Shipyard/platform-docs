# ADR-0004: Local-first on k3d

- **Status:** Accepted
- **Date:** 2026-06-06

## Context

A production-style Kubernetes platform usually means a cloud bill. The goal here is to practice the
operational patterns (GitOps, Helm, ingress, least privilege) without one for now.

In the future, I will probably deploy to a low cost or free cloud provider to practice terraform.

## Decision

Run everything on **k3d** (k3s in Docker) locally. Charts target k3d conventions: Traefik ingress
with `*.localhost` hosts, the `local-path` storage class for PVCs, and `IfNotPresent` image pull so
`k3d image import` works as a fallback.

## Consequences

- ✅ Free, fast to recreate, runs on my laptop; the whole stack tears down with one command.
- ✅ Patterns transfer to managed Kubernetes -- the charts/GitOps wiring is cloud-agnostic.
- ⚠️ Currently **single-node** (1 server, 0 agents), so multi-node scheduling/affinity isn't
  exercised yet. It will be easy to add additional nodes.
- ⚠️ Local conveniences (Traefik built-in, `*.localhost`, manual secrets) stand in for things a real
  cluster does differently (Nginx edge, real DNS/TLS, Vault/ESO).
