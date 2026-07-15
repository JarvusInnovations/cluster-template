# Exposing services (Gateway API)

Public services are exposed through **Envoy Gateway** and the Kubernetes **Gateway API**.
This replaces the older ingress-nginx path (see [Legacy: ingress-nginx](#legacy-ingress-nginx)).

The design rests on one rule, and most of the sharp edges below are consequences of
getting it wrong:

> **A cluster decides *where* a service is reachable. An application decides *what* it
> serves. Neither declares the other's half.**

| The cluster owns | The application owns |
| --- | --- |
| `Gateway` — hostnames, TLS, certificate issuer, `gatewayClassName` | `HTTPRoute` — paths, backend Services, ports |
| `GatewayClass`, the shared data plane, the HTTP→HTTPS redirect | its own container ports and routing |

Hostnames, TLS, and issuers differ between clusters and are negotiated across every app
sharing the ingress — they are cluster facts. Paths and ports change when the app changes
and belong in the same commit as that change — they are app facts. An app cannot correctly
author cluster facts: it doesn't know the cluster's issuer name, cert Secret names, or
Gateway class, and guessing wrong fails in ways that are hard to see (see
[Failure modes](#failure-modes)).

## Exposing a service

**1. The cluster provides a `Gateway` named after the app**, in the app's namespace, with
one HTTPS listener per hostname:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-app
  namespace: my-app
  annotations:
    cert-manager.io/cluster-issuer: {{ cluster.cluster_issuer }}
spec:
  gatewayClassName: eg
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      hostname: my-app.{{ cluster.wildcard_hostname }}
      tls:
        mode: Terminate
        certificateRefs:
          - name: my-app-gw-tls        # cert-manager creates this from the annotation
      allowedRoutes:
        namespaces:
          from: Same
```

cert-manager sees the `cluster-issuer` annotation and issues the certificate into the
`certificateRefs` Secret automatically. No `Certificate` resource is written by hand.

**2. The application ships an `HTTPRoute`** that attaches to that Gateway by name and
declares **no hostnames**:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app
  namespace: my-app
spec:
  parentRefs:
    - name: my-app          # the Gateway the cluster provides
  # no hostnames — inherited from the Gateway's listeners
  rules:
    - matches:
        - path: { type: PathPrefix, value: / }
      backendRefs:
        - name: my-app
          port: 80
```

A route with no `hostnames` **inherits** them from the listeners it attaches to. So the
same route file works unchanged in every environment — the Gateway supplies the hostname —
and the app repo contains zero cluster facts. This is the mechanism that makes the
ownership split practical; it is a supported part of the Gateway API and behaves correctly
on Envoy Gateway.

When an app team ships routes from their own repository, this is all they ship: an
HTTPRoute, attached by convention to a Gateway named after the app.

## HTTP, HTTPS, and certificates

- **HTTP→HTTPS is handled once, globally.** A single redirect `HTTPRoute` on the shared
  HTTP listener 301s everything to HTTPS. Per-app HTTPRoutes attach only to their per-app
  HTTPS Gateway — they do **not** each carry a redirect.
- **ACME challenges bypass the redirect automatically.** cert-manager creates a short-lived
  HTTPRoute per challenge with an `Exact` path match on `/.well-known/acme-challenge/…`,
  which outranks the catch-all redirect under Gateway API precedence.
- **One LoadBalancer for all Gateways.** The `EnvoyProxy` resource sets
  `mergeGateways: true`, so every Gateway shares one Envoy data plane and one cloud
  LoadBalancer. Without it, each Gateway provisions its own LB and cost multiplies. Do not
  disable it.

## Failure modes

These are not hypothetical — each one cost real time on a production cluster.

!!! danger "Certificate issuance depends on **in-cluster** reachability"
    cert-manager runs an HTTP-01 self-check *from inside the cluster* before it asks the CA
    to validate. So a solver the cluster cannot reach from within will never issue a cert,
    even if it works perfectly from the public internet.

    This is what makes ingress-nginx-with-PROXY-protocol unusable for issuance once the
    in-cluster PROXY-header shim (hairpin-proxy) is removed: in-cluster traffic bypasses the
    load balancer that would add the header, so the backend drops the connection and the
    self-check fails silently. The general rule: **a component in the request path that
    needs a header only the load balancer supplies is unreachable from inside the cluster.**
    Envoy Gateway does not use PROXY protocol, so it is reachable in-cluster and its
    self-checks pass. Never enable `ClientTrafficPolicy.enableProxyProtocol` to recover
    client IPs without accounting for this.

!!! danger "Don't leave gateway certs pending across a renewal window"
    If you pre-create per-app Gateways (and their pending ACME challenges) but delay the DNS
    cutover, those challenges sit `pending` indefinitely. A pending challenge **holds its
    hostname's slot** in cert-manager's scheduler, which will not run two HTTP-01 challenges
    for the same hostname at once — so it can silently starve the *legacy* certificate's
    renewal for that same hostname. If you must pause mid-migration, remove the
    `cluster-issuer` annotation from the pre-created Gateways so no challenges exist to hold
    the slots.

!!! warning "A Challenge can outlive its Order and deadlock renewals"
    A cert-manager `Challenge` is owned by **both** its `Order` and the `ClusterIssuer`.
    Kubernetes garbage-collects a dependent only when *all* owners are gone, so a Challenge
    survives its Order's deletion as a half-orphan: still `processing`, no controller driving
    it, no GC path to remove it — and still holding its scheduler slot. Symptom:
    `kubectl get challenges -A` shows a challenge `pending` for days with a sibling for the
    same `dnsName` stuck with an empty status. Delete the stale one by name. The
    `CertChallengeStuck` alert (shipped with Grafana alerting) catches this.

!!! warning "`ListenerSet` is not usable yet"
    Envoy Gateway before v1.8 does not reconcile `ListenerSet` — it watches `XListenerSet`
    and logs `XListenerSet CRD not found, skipping XListenerSet watch`. A `ListenerSet`
    applies cleanly to the API server and is then **silently ignored**, leaving the hostname
    with no listener and nothing in any log to say why. Use per-app `Gateway` resources with
    `mergeGateways: true` until the cluster's Envoy Gateway is on v1.8+.

## Legacy: ingress-nginx

ingress-nginx remains in the manifests for compatibility with services that haven't moved
to the Gateway API yet, but it is being retired and **new services should not use it**.
The old flow — an `Ingress` with `kubernetes.io/ingress.class: nginx` — still functions
where it already exists. When migrating an existing Ingress, mirror its routing faithfully
in the HTTPRoute: multi-path Ingresses (e.g. `/api` to a backend, `/` to a frontend) and
redirect-only hosts must be reproduced as explicit HTTPRoute rules, or behavior changes
silently at cutover.
