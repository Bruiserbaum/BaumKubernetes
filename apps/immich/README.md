# Immich

**High-performance photo and video backup** — self-hosted Google Photos alternative
with mobile apps, face recognition, and object detection.

**Source:** [immich-app/immich](https://github.com/immich-app/immich)
**Images:** `ghcr.io/immich-app/immich-server:release`, `ghcr.io/immich-app/immich-machine-learning:release`
**Namespace:** `immich`

---

## What's deployed

| Component | Purpose |
|-----------|---------|
| `immich-server` | Main API + web UI (port 2283) |
| `immich-machine-learning` | Face recognition / CLIP model server |
| `redis` | Job queue / cache |
| `postgres` | pgvecto-rs — vector similarity for smart search |

---

## Secrets required

```bash
kubectl create namespace immich

kubectl create secret generic immich-secret \
  --namespace immich \
  --from-literal=DB_PASSWORD="$(openssl rand -hex 32)"
```

---

## Deploy

```bash
# 1. Create secret
# 2. Edit configmap.yaml — set IMMICH_DOMAIN
# 3. Edit ingress.yaml — set host
kubectl apply -k apps/immich/
```

## Deploy via Portainer

> Create the namespace and secret first using the `kubectl` commands in **Secrets required** above.

1. In Portainer: **Kubernetes** → **Manifests** → **Deploy**
2. Select **Repository**
3. Enter:
   - Repository URL: `https://github.com/Bruiserbaum/BaumKubernetes`
   - Compose path: `apps/immich/kustomization.yaml`
   - Namespace: `immich`
4. Click **Deploy**

---

## Storage

| PVC | Class | Size | Content |
|-----|-------|------|---------|
| `immich-uploads-pvc` | `longhorn-ssd` | 500Gi | Photo/video library |
| `immich-postgres-pvc` | `longhorn-nvme` | 20Gi | Database |
| `immich-ml-cache-pvc` | `longhorn-nvme` | 10Gi | ML model cache |

Adjust sizes in `pvc.yaml` for your library size. For very large libraries,
consider an NFS-backed PV for `immich-uploads-pvc` — see
[cluster-setup/storage/README.md](../../cluster-setup/storage/README.md).

---

## SSO with Authentik (optional)

In Authentik, create an **OAuth2/OpenID Connect** provider for Immich.
Then in the Immich web UI: **Administration → OAuth → Enable OAuth**.

| Immich field | Value |
|-------------|-------|
| Issuer URL | `https://authentik.example.com/application/o/immich/` |
| Client ID | from Authentik |
| Client Secret | from Authentik |
| Scope | `openid profile email` |

---

## Mobile apps

- iOS: [App Store](https://apps.apple.com/app/immich/id1613945652)
- Android: [Play Store](https://play.google.com/store/apps/details?id=app.alextran.immich)
- Point to `https://immich.example.com`
