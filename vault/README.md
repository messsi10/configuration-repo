## Vault install (Helm)

This repo contains baseline Helm values to install HashiCorp Vault and use it as the source of truth for app secrets.

### Install Vault

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

kubectl create namespace vault || true
helm upgrade --install vault hashicorp/vault -n vault -f vault/values.yaml
```

### Install External Secrets Operator (ESO)

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

kubectl create namespace external-secrets || true
helm upgrade --install external-secrets external-secrets/external-secrets -n external-secrets -f external-secrets/values.yaml
```

### Vault bootstrap (one-time)

1. Initialize + unseal Vault (unless you have auto-unseal).
2. Enable KV v2 at `kv/` (or adjust `SecretStore.spec.provider.vault.path`).
3. Enable Kubernetes auth at `kubernetes/`.
4. Create a policy + role for ESO.

Example commands (adjust names/namespaces to your cluster):

```bash
# If you previously enabled the injector and hit a MutatingWebhookConfiguration conflict on upgrades,
# the simplest path for minikube/dev is to reinstall Vault with injector disabled (see vault/values.yaml):
# helm -n vault uninstall vault
# kubectl delete mutatingwebhookconfiguration vault-agent-injector-cfg --ignore-not-found
# helm upgrade --install vault hashicorp/vault -n vault -f vault/values.yaml
#
# Initialize + unseal (single-node, dev):
kubectl -n vault exec -it vault-0 -- vault operator init -key-shares=1 -key-threshold=1
kubectl -n vault exec -it vault-0 -- vault operator unseal

# IMPORTANT: Most "enable/configure" operations require a token with `sudo` capability
# (for dev this is usually the Initial Root Token from `vault operator init`).
# You can either open a shell and `vault login`, or pass VAULT_TOKEN per command.
#
# Option A (recommended): open a shell inside the pod:
# kubectl -n vault exec -it vault-0 -- sh
#   export VAULT_ADDR=http://127.0.0.1:8200
#   vault login <INITIAL_ROOT_TOKEN>
#
# Option B: pass token inline:
# ROOT_TOKEN="<INITIAL_ROOT_TOKEN>"
# kubectl -n vault exec -it vault-0 -- sh -c "export VAULT_TOKEN=$ROOT_TOKEN; vault auth list"

# Enable KV v2 at path kv/
kubectl -n vault exec -it vault-0 -- vault secrets enable -path=kv kv-v2

# Enable k8s auth
kubectl -n vault exec -it vault-0 -- vault auth enable kubernetes

# Configure k8s auth (cluster-specific values required)
kubectl -n vault exec -it vault-0 -- sh -c '
  vault write auth/kubernetes/config \
    token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
    kubernetes_host="https://kubernetes.default.svc:443" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
'
```

Create policy (example: read `kv/data/socketio-test-service` only):

```hcl
path "kv/data/socketio-test-service" {
  capabilities = ["read"]
}
```

Create role bound to the ServiceAccount used by the app namespace:

```bash
kubectl -n vault exec -it vault-0 -- vault policy write socketio-test-service -<<'HCL'
path "kv/data/socketio-test-service" {
  capabilities = ["read"]
}
HCL

kubectl -n vault exec -it vault-0 -- vault write auth/kubernetes/role/external-secrets-socketio-namespace \
  bound_service_account_names=external-secrets-vault \
  bound_service_account_namespaces=socketio-namespace \
  policies=socketio-test-service \
  ttl=1h
```

If you want **one Vault role for all services** (so you can add keys in Vault UI for any app without changing roles),
create a single policy + role:

```bash
kubectl -n vault exec -it vault-0 -- vault policy write eso-read-all -<<'HCL'
path "kv/data/*" {
  capabilities = ["read"]
}
path "kv/metadata/*" {
  capabilities = ["read"]
}
HCL

kubectl -n vault exec -it vault-0 -- vault write auth/kubernetes/role/external-secrets-any-namespace \
  bound_service_account_names=external-secrets-vault \
  bound_service_account_namespaces="*" \
  policies=eso-read-all \
  ttl=1h
```

### Put secrets into Vault

```bash
kubectl -n vault exec -it vault-0 -- vault kv put kv/socketio-test-service \
  TEST__LOGIN_VAR="login" \
  TEST_PASSWORD_VAR="password"
```

Then apply the manifests in `socketio-test-service/external-secrets/` to create a Kubernetes Secret `socketio-test-service-env`
that your Helm values reference via `valueFrom.secretKeyRef`.

