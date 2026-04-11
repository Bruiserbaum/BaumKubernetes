# Vaultwarden

**Bitwarden-compatible password manager** — lightweight Rust implementation.

**Source:** [dani-garcia/vaultwarden](https://github.com/dani-garcia/vaultwarden)
**Image:** `vaultwarden/server:latest`
**Namespace:** `vaultwarden`

---

## Secrets required

```bash
kubectl create namespace vaultwarden

kubectl create secret generic vaultwarden-secret \
  --namespace vaultwarden \
  --from-literal=ADMIN_TOKEN="$(openssl rand -base64 48)"
```

---

## Deploy

```bash
# 1. Create secret (see above)
# 2. Edit configmap.yaml — set DOMAIN to your public URL
# 3. Edit ingress.yaml — set host
kubectl apply -k apps/vaultwarden/
```

---

## Configuration

| Key | Description |
|-----|-------------|
| `DOMAIN` | Public URL, e.g. `https://vault.example.com` — required for WebAuthn |
| `ADMIN_TOKEN` | Hashed or raw token for `/admin` panel |
| `SIGNUPS_ALLOWED` | `true` for initial setup, then set to `false` |

### Disable signups after creating your account

```bash
kubectl set env deployment/vaultwarden -n vaultwarden SIGNUPS_ALLOWED=false
```

---

## SSO with Authentik (optional)

Vaultwarden SSO requires the `SSO_ENABLED` feature (available in recent builds).
In Authentik, create an **OAuth2/OpenID Connect** provider, then:

```bash
kubectl patch configmap vaultwarden-config -n vaultwarden --patch '
data:
  SSO_ENABLED: "true"
  SSO_AUTHORITY: "https://authentik.example.com/application/o/vaultwarden/"
  SSO_CLIENT_ID: "vaultwarden"
'
kubectl create secret generic vaultwarden-sso \
  --namespace vaultwarden \
  --from-literal=SSO_CLIENT_SECRET="<client secret from Authentik>"
```

---

## Storage

| PVC | Class | Size | Content |
|-----|-------|------|---------|
| `vaultwarden-pvc` | `local-path-fast` | 5Gi | Vault database + attachments |
