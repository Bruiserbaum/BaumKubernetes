# Nextcloud

**File sync and collaboration platform** — files, calendars, contacts, and office docs.

> **Note:** Nextcloud AIO (All-in-One) requires Docker socket access and does not
> run well on Kubernetes. This manifest uses the standard `nextcloud` image with a
> separate MariaDB and Redis deployment instead.

**Source:** [nextcloud/server](https://github.com/nextcloud/server)
**Images:** `nextcloud:latest`, `mariadb:11`, `redis:7-alpine`
**Namespace:** `nextcloud`

---

## Secrets required

```bash
kubectl create namespace nextcloud

kubectl create secret generic nextcloud-secret \
  --namespace nextcloud \
  --from-literal=MYSQL_ROOT_PASSWORD="$(openssl rand -hex 32)" \
  --from-literal=MYSQL_PASSWORD="$(openssl rand -hex 32)" \
  --from-literal=NEXTCLOUD_ADMIN_PASSWORD="$(openssl rand -hex 16)"
```

---

## Deploy

```bash
# 1. Create secrets
# 2. Edit configmap.yaml — set NEXTCLOUD_TRUSTED_DOMAINS to your domain
# 3. Edit ingress.yaml — set host
kubectl apply -k apps/nextcloud/
```

## Deploy via Portainer

> Create the namespace and secret first using the `kubectl` commands in **Secrets required** above.

1. In Portainer: **Kubernetes** → **Manifests** → **Deploy**
2. Select **Repository**
3. Enter:
   - Repository URL: `https://github.com/Bruiserbaum/BaumKubernetes`
   - Repository reference: `refs/heads/master`
   - Manifest path: `apps/nextcloud/manifest.yaml`
   - Namespace: `nextcloud`
4. Click **Deploy**

---

## Post-deploy configuration

```bash
# Install recommended apps via CLI
kubectl exec -n nextcloud deployment/nextcloud -- php occ app:install calendar
kubectl exec -n nextcloud deployment/nextcloud -- php occ app:install contacts

# Set up cron (run every 5 minutes for background jobs)
kubectl exec -n nextcloud deployment/nextcloud -- php occ background:cron
```

---

## Storage

| PVC | Class | Size | Content |
|-----|-------|------|---------|
| `nextcloud-data-pvc` | `longhorn-ssd` | 500Gi | User files |
| `nextcloud-mysql-pvc` | `longhorn-nvme` | 20Gi | MariaDB database |
| `nextcloud-redis-pvc` | `longhorn-nvme` | 2Gi | Redis session cache |
