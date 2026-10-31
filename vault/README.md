# Ledger regression Vault bootstrap

The Helm configuration enables the Vault Agent Injector and grants the Vault
server service account Kubernetes TokenReview permission. Helm does not enable
Vault's Kubernetes auth method or create Vault policies and roles. Run the
following once using an already authenticated Vault administrator CLI session;
do not put an admin or root token in this repository or in Kubernetes
application configuration.

## Configure Kubernetes authentication and the read-only role

From the repository root, run this where `vault` CLI is authenticated to the
existing Vault installation. First inspect `vault auth list`; only enable the
method if `kubernetes/` is not already mounted:

```sh
vault auth enable kubernetes
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc"
vault policy write ledger-regression vault/policies/ledger-regression.hcl
vault write auth/kubernetes/role/ledger-regression \
  bound_service_account_names=ledger-vault \
  bound_service_account_namespaces=regression \
  token_policies=ledger-regression \
  ttl=1h
```

Skip `vault auth enable kubernetes` if the mount already exists. The Vault
chart's `server.authDelegator.enabled` setting provides the Vault server
service account permission required for TokenReview. Do not change or rewrite
the existing `secret/ledger/database` value.

## Provide the public Vault CA certificate to regression

The Ledger pod needs only the public CA certificate, not the Vault TLS private
key. The following copies only `ca.crt` from the existing `vault-tls` Secret
into a Secret in the regression namespace; it does not print the certificate
or copy the private key. Run after the regression namespace exists:

```sh
tmp_ca="$(mktemp)"
trap 'rm -f "$tmp_ca"' EXIT
kubectl get secret vault-tls -n vault -o jsonpath='{.data.ca\.crt}' \
  | base64 -d > "$tmp_ca"
test -s "$tmp_ca" || { echo 'vault-tls does not contain a non-empty ca.crt'; exit 1; }
kubectl create secret generic vault-ca -n regression \
  --from-file=ca.crt="$tmp_ca" --dry-run=client -o yaml \
  | kubectl apply -f -
```

Sync the existing Vault Argo CD Application first so the injector is available,
then sync `regression-ledger`. The regression pod will use the
`ledger-vault` service account, authenticate as `ledger-regression`, retrieve
`secret/data/ledger/database`, and load the rendered `application.yaml` before
Java starts.
