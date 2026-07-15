# Ingress (legacy)

Public services are now exposed through the Gateway API — see
[Exposing services (Gateway API)](gateway.md).

ingress-nginx and the `Ingress` resource remain in the manifests for compatibility with
services that predate the migration, but they are being retired. **New services should use
the Gateway API.** When moving an existing `Ingress` to an `HTTPRoute`, reproduce its
routing exactly — see the migration note at the end of the Gateway page.
