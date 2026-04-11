# Nginx Proxy Manager

**GUI reverse proxy** — manage proxy hosts, redirects, and SSL certificates via a
web interface. Useful for routing traffic to services outside the cluster.

> **On Kubernetes:** For in-cluster routing, use **ingress-nginx** (already set up in
> `cluster-setup/ingress-nginx/`). Deploy NPM only if you need to proxy external
> services or non-Kubernetes hosts from the same interface.

**Source:** [NginxProxyManager/nginx-proxy-manager](https://github.com/NginxProxyManager/nginx-proxy-manager)
**Images:** `jc21/nginx-proxy-manager:latest`, `jc21/mariadb-aria:latest`
**Namespace:** `npm`

---

## Secrets required

```bash
kubectl create namespace npm

kubectl create secret generic npm-secret \
  --namespace npm \
  --from-literal=DB_ROOT_PASSWORD="$(openssl rand -hex 32)" \
  --from-literal=DB_PASSWORD="$(openssl rand -hex 32)"
```

---

## Deploy

```bash
kubectl apply -k apps/nginx-proxy-manager/
```

---

## Access

- Admin UI: `https://npm.example.com` (or NodePort 81)
- Default login: `admin@example.com` / `changeme`
- **Change the password immediately on first login**

---

## Ports

NPM needs ports 80 and 443 for proxied traffic. On Kubernetes, these are served
by ingress-nginx. NPM is deployed here as a NodePort service so it can bind
different ports if needed, or access is strictly through the admin UI ingress.

---

## Storage

| PVC | Class | Size | Content |
|-----|-------|------|---------|
| `npm-data-pvc` | `local-path-fast` | 5Gi | Proxy configs, SSL certs |
| `npm-mysql-pvc` | `local-path-fast` | 5Gi | MariaDB database |
