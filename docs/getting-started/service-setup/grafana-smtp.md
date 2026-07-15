# Grafana SMTP & certificate alerting

The template ships certificate alert rules (in `k8s-common/grafana/default-values.yaml`)
that email the cluster's operators when a certificate stops renewing. Grafana sends that
mail through **Postmark**, and the credential, sender, and recipient list come from a
`grafana-smtp` Secret in the `grafana` namespace.

Until this Secret exists, Grafana still runs and the rules still evaluate — they just
can't deliver. Creating it turns delivery on.

## Why this is a Secret, and why Grafana rather than Alertmanager

Postmark's server token is **both** the SMTP username and the password. Prometheus
Alertmanager can read the SMTP password from a file but has no way to read the username
from one, so on a public GitOps repo the token would end up committed in plaintext.
Grafana reads both from a Secret as environment variables, exposing nothing — which is
why alerting lives in Grafana here.

The recipient list lives in the Secret too: it's usually personal addresses, and this
keeps them out of a public repo.

## Keys

| Key | Value |
| --- | --- |
| `user` | Postmark **server token** |
| `password` | the **same** Postmark server token |
| `from-address` | a verified Postmark sender for this cluster (e.g. `alerts@example.org`) |
| `from-name` | the sender name shown in alert emails — **name the cluster** so its origin is obvious at a glance, e.g. `CodeForPhilly — Live` |
| `recipients` | semicolon-separated list, e.g. `a@example.org;b@example.org` |

The `from-address` must be a verified sender signature (or under a verified domain) in the
Postmark server, or Postmark rejects the mail. Set `from-name` distinctly per cluster —
when several clusters share one Postmark server and email the same people, it's the only
thing in the inbox that says which cluster fired.

## Create it

Sealed into the cluster repo (recommended — survives a cluster rebuild):

```bash
TOKEN='<postmark-server-token>'
kubectl create secret generic grafana-smtp \
    --namespace grafana \
    --dry-run=client -o json \
    --from-literal=user="$TOKEN" \
    --from-literal=password="$TOKEN" \
    --from-literal=from-address='alerts@example.org' \
    --from-literal=from-name='CodeForPhilly — Live' \
    --from-literal=recipients='a@example.org;b@example.org' \
  | kubeseal --format yaml \
      --controller-namespace sealed-secrets --controller-name sealed-secrets \
  > grafana/grafana-smtp.sealed.yaml
```

Commit the sealed file. For a quick non-GitOps setup, drop the `kubeseal` pipe and apply
the plain Secret directly instead — but never commit an unsealed one.

## Verify delivery

Don't assume a rendered config delivers. After Grafana rolls out, fire a real test through
its own notifier (a rendered-but-undeliverable pipeline is exactly the silent failure the
alerts exist to prevent):

```bash
# Grafana's admin password lives in the grafana-initial-admin Secret.
# Test the provisioned contact point with its real, env-expanded recipients:
kubectl -n grafana exec deploy/grafana -- \
  wget -qO- --post-data='{"receivers":[{"name":"cluster-alerts-email",
    "grafana_managed_receiver_configs":[{"name":"cluster-alerts-email","type":"email",
    "settings":{"addresses":"<your-address>","singleEmail":false}}]}],
    "alert":{"annotations":{"summary":"alerting pipeline test"}}}' \
  --header='Content-Type: application/json' \
  "http://admin:$ADMIN_PW@localhost:3000/api/alertmanager/grafana/config/api/v1/receivers/test"
```

A `status: "ok"` and an email in the inbox means the pipeline works end to end.
