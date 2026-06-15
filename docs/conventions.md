# Conventions

How changes flow through CJs-Shipyard. Keep these intact — they're what make the GitOps loop work.

## Core rules

- **Helm templates, not raw YAML.** Every deployable is a chart in `helm-charts/`. Don't hand-write
  manifests to apply directly.
- **Declarative / GitOps only.** Don't `kubectl apply` workloads to a running cluster — change the
  chart or the Argo `Application` and let Argo reconcile. The *only* manual steps should be the
  day-0 `install/` bootstrap and the (temporary) hand-applied secrets.
- **Per-repo Dockerfiles.** Each service owns its build under `build/Dockerfile`.
- **CI is centralized.** Image build/push and Helm tag bumps come from reusable workflows in
  `ci-templates/` (referenced as `…/ci-templates/.github/workflows/<name>.yml@main`). Don't inline
  CI logic per repo.
- **Observability in app code.** Instrumentation (OTel SDK) belongs in `api/`, not the edge.

## Adding or changing a service

1. **Code** in `api/` (or a new component repo).
2. **Chart** in `helm-charts/charts/<service>/` — copy `write-api/` as the template; keep the
   `_helpers.tpl` naming shape so in-cluster DNS resolves.
3. **Argo `Application`** in `argocd/apps/shipyard/<service>.yaml` *only if it's a new deployable*
   (existing services already track their chart path).
4. **Shared CI** changes go in `ci-templates/`, never copied per-repo.

## Secrets

No chart embeds credentials. Each references a `Secret` by name (`existingSecret`); the real
`secret.yaml` is gitignored (`*secret.yaml`). Applied by hand today; Vault + External Secrets
Operator is the planned replacement (see [ADR index](./adr/README.md)).

## Validating a chart before commit

```bash
helm lint charts/<name>
helm template test charts/<name> | kubectl apply --dry-run=server -f -
```

Treat the **server-side dry-run** as the real gate — a chart can lint clean but still fail the API.
