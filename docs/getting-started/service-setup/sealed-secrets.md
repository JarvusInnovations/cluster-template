# Sealed Secrets

The [sealed-secrets](https://github.com/bitnami-labs/sealed-secrets) controller will automatically generate a public+private keypair for each cluster that will be used to encrypt and decrypt secrets.

After any new cluster is provisioned, the generated keypair should be backed up externally (i.e. to a shared VaultWarden/BitWarden collection). With the keypair backed up, secrets can be [decrypted locally](https://github.com/bitnami-labs/sealed-secrets#can-i-decrypt-my-secrets-offline-with-a-backup-key) or [restored to a new cluster](https://github.com/bitnami-labs/sealed-secrets#can-i-bring-my-own-pre-generated-certificates) in the future.

## Back up generated keys

[From the `sealed-secrets` README.md](https://github.com/bitnami-labs/sealed-secrets#can-i-decrypt-my-secrets-offline-with-a-backup-key):

```bash
kubectl get secret \
    -n sealed-secrets \
    -l sealedsecrets.bitnami.com/sealed-secrets-key \
    -o yaml \
> cluster-sealed-secrets-master.key
```

!!! warning "Sensitive data!"

    Be sure to keep this file secure and delete from your working directory after uploading it to a secure credentials vault for backup.

    **Do not commit this file to source control**

## Expose the public certificate

The controller serves its public certificate at `/v1/cert.pem`, which local `kubeseal`
clients can fetch instead of pulling it from the cluster each time. Expose it through the
Gateway API — see [Exposing services](../../development/features/gateway.md) for the full
pattern and the ownership split.

Provide a `Gateway` and an `HTTPRoute` (leave the chart's own `ingress` disabled):

=== "gateway.yaml"

    ```yaml
    apiVersion: gateway.networking.k8s.io/v1
    kind: Gateway
    metadata:
      name: sealed-secrets
      namespace: sealed-secrets
      annotations:
        cert-manager.io/cluster-issuer: {{ cluster.cluster_issuer }}
    spec:
      gatewayClassName: eg
      listeners:
        - name: https
          protocol: HTTPS
          port: 443
          hostname: sealed-secrets.{{ cluster.wildcard_hostname }}
          tls:
            mode: Terminate
            certificateRefs:
              - name: sealed-secrets-gw-tls
          allowedRoutes:
            namespaces:
              from: Same
    ---
    apiVersion: gateway.networking.k8s.io/v1
    kind: HTTPRoute
    metadata:
      name: sealed-secrets
      namespace: sealed-secrets
    spec:
      parentRefs:
        - name: sealed-secrets
      rules:
        # Only the public cert is served — not the whole controller surface.
        - matches:
            - path: { type: Exact, value: /v1/cert.pem }
          backendRefs:
            - name: sealed-secrets
              port: 8080
    ```

Once deployed, local `kubeseal` clients can be configured to use it by setting the `SEALED_SECRETS_CERT` environment variable:

```bash
export SEALED_SECRETS_CERT=https://sealed-secrets.{{ cluster.wildcard_hostname }}/v1/cert.pem
```
