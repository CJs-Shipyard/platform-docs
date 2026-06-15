# Docs

Deeper material behind the [project README](../README.md). These are intentionally short -- a
skeleton to grow as the platform does.

| Doc | What's in it |
|-----|--------------|
| [architecture.md](./architecture.md) | Component responsibilities, request flow, namespace layout |
| [local-setup.md](./local-setup.md) | Prerequisites and steps to stand the platform up yourself |
| [observability.md](./observability.md) | The planned OTel / Loki / Grafana approach |
| [conventions.md](./conventions.md) | How changes flow through the repos |
| [adr/](./adr/README.md) | Architecture Decision Records -- one lightweight file per decision |

## Live vs. planned

This documentation separates **implemented** from **planned** throughout. As components come
online, update the status markers here and in the root README's Diagram 2 (the live view). The
quickest way to refresh the live picture:

```bash
kubectl get applications -n argocd
kubectl get pods,svc,ingress -n shipyard -o wide
```
