# Karakeep

**Bookmark manager** — save links, screenshots, and full-page archives with AI tagging.
Includes a Chrome extension and mobile apps.

**Source:** [karakeep-app/karakeep](https://github.com/karakeep-app/karakeep)
**Images:** `ghcr.io/karakeep-app/karakeep:latest`, `gcr.io/zenika-hub/alpine-chrome:123`, `getmeili/meilisearch:v1.6`
**Namespace:** `karakeep`

---

## Secrets required

```bash
kubectl create namespace karakeep

kubectl create secret generic karakeep-secret \
  --namespace karakeep \
  --from-literal=NEXTAUTH_SECRET="$(openssl rand -hex 32)" \
  --from-literal=MEILI_MASTER_KEY="$(openssl rand -hex 32)"
```

Optional OpenAI key for AI auto-tagging:
```bash
kubectl patch secret karakeep-secret -n karakeep \
  --type merge --patch '{"stringData":{"OPENAI_API_KEY":"sk-..."}}'
```

---

## Deploy

```bash
kubectl apply -k apps/karakeep/
```

## Deploy via Portainer

> Create the namespace and secret first using the `kubectl` commands in **Secrets required** above.

1. In Portainer: **Kubernetes** → **Manifests** → **Deploy**
2. Select **Repository**
3. Enter:
   - Repository URL: `https://github.com/Bruiserbaum/BaumKubernetes`
   - Compose path: `apps/karakeep/kustomization.yaml`
   - Namespace: `karakeep`
4. Click **Deploy**

---

## Storage

| PVC | Class | Size | Content |
|-----|-------|------|---------|
| `karakeep-data-pvc` | `longhorn-nvme` | 20Gi | Saved pages, screenshots |
| `karakeep-meilisearch-pvc` | `longhorn-nvme` | 5Gi | Search index |
