# Authentik

**SSO / Identity Provider** — deploy this first. Other apps (Immich, Vaultwarden,
LibreChat, Karakeep) optionally use Authentik for single sign-on via OIDC/OAuth2.

**Source:** [goauthentik/authentik](https://github.com/goauthentik/authentik)
**Image:** `ghcr.io/goauthentik/server:latest`
**Namespace:** `authentik`

---

## What's deployed

| Component | Purpose |
|-----------|---------|
| `authentik-server` | API + web UI (port 9000 HTTP, 9443 HTTPS) |
| `authentik-worker` | Background task runner |
| `postgresql` | User/config database |
| `redis` | Session cache / task queue |

---

## Secrets required

Create before deploying:

```bash
kubectl create namespace authentik

kubectl create secret generic authentik-secret \
  --namespace authentik \
  --from-literal=AUTHENTIK_SECRET_KEY="$(openssl rand -hex 32)" \
  --from-literal=PG_PASS="$(openssl rand -hex 32)"
```

See `secret.example.yaml` for all keys.

---

## Deploy

```bash
# 1. Create secrets (see above)
# 2. Edit configmap.yaml — set AUTHENTIK_HOST to your domain
# 3. Edit ingress.yaml — set host to your domain
kubectl apply -k apps/authentik/
```

---

## Initial setup

1. Browse to `https://authentik.example.com/if/flow/initial-setup/`
2. Set admin username and password
3. Create applications and providers for SSO integrations

---

## SSO integration for other apps

After deploying Authentik, configure providers for:

| App | Provider type | Docs |
|-----|--------------|------|
| Immich | OAuth2 | [Immich OAuth docs](https://immich.app/docs/administration/oauth) |
| Vaultwarden | OIDC (SSO feature) | [Vaultwarden SSO](https://github.com/dani-garcia/vaultwarden/wiki/SSO) |
| LibreChat | OpenID Connect | [LibreChat SSO](https://www.librechat.ai/docs/configuration/pre_composed_env/oauth) |
| Karakeep | OIDC | [Karakeep docs](https://docs.karakeep.app/Installation/authentication) |

---

## Ingress

Default: `https://authentik.example.com`
Update `host:` in `ingress.yaml` before deploying.

---

## Storage

| PVC | Class | Size | Content |
|-----|-------|------|---------|
| `authentik-postgres-pvc` | `local-path-fast` | 10Gi | PostgreSQL data |
| `authentik-redis-pvc` | `local-path-fast` | 2Gi | Redis data |
| `authentik-media-pvc` | `local-path-fast` | 5Gi | Uploaded avatars/icons |
