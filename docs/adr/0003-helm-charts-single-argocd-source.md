# ADR-0003: helm-charts as the single Argo CD source

- **Status:** Accepted
- **Date:** 2026-06-06

## Context

In a poly-repo setup, charts could live next to each service. Argo CD then has to track many source
repos, and "what's deployed" is scattered.

## Decision

Keep **all** Helm charts in one `helm-charts` repo, and make it the **single source of truth** Argo
CD deploys. Service repos build docker images; CI commits the resulting `image.tag` into
`helm-charts/charts/<svc>/values.yaml`. Argo `Application` manifests (in `argocd/`) all point at
`helm-charts`.

This makes it:
- a simpler structure for Argo CD to ingest
- more portable for if I ever want to enable outside people/developers to clone and launch my code on their local kubernetes clusters.

## Consequences

- ✅ One place answers "what is deployed and at what version" — `values.yaml` is the deploy record.
- ✅ A root **app-of-apps** in `argocd/` auto-discovers new `Application` manifests; adding a service needs no
  edit to the root app.
- ✅ Decouples "build an image" from "deploy an image" cleanly across repos.
- ⚠️ Requires a cross-repo push (CI needs a PAT with write on `helm-charts`).
