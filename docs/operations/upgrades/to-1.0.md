# To 1.0.x

If upgrading from a version earlier than v0.20.x, follow the notes for upgrading [to v0.20.x](./to-0.20.md) instead.

## Deployment

Before deploying an upgrade to v1.0.x, delete existing `ingress-nginx` jobs to prevent errors about immutable fields being changed:

```bash
kubectl -n ingress-nginx delete jobs ingress-nginx-admission-create ingress-nginx-admission-patch
```
