# WordPress

**Content management system** — WordPress with MariaDB backend.

**Source:** [WordPress/WordPress](https://github.com/WordPress/WordPress)
**Images:** `wordpress:latest`, `mariadb:11`
**Namespace:** `wordpress`

---

## Secrets required

```bash
kubectl create namespace wordpress

kubectl create secret generic wordpress-secret \
  --namespace wordpress \
  --from-literal=MYSQL_ROOT_PASSWORD="$(openssl rand -hex 32)" \
  --from-literal=MYSQL_PASSWORD="$(openssl rand -hex 32)" \
  --from-literal=WORDPRESS_AUTH_KEY="$(openssl rand -hex 32)" \
  --from-literal=WORDPRESS_SECURE_AUTH_KEY="$(openssl rand -hex 32)" \
  --from-literal=WORDPRESS_LOGGED_IN_KEY="$(openssl rand -hex 32)" \
  --from-literal=WORDPRESS_NONCE_KEY="$(openssl rand -hex 32)"
```

---

## Deploy

```bash
kubectl apply -k apps/wordpress/
```

---

## Storage

| PVC | Class | Size | Content |
|-----|-------|------|---------|
| `wordpress-pvc` | `local-path-fast` | 20Gi | wp-content (uploads, themes, plugins) |
| `wordpress-mysql-pvc` | `local-path-fast` | 10Gi | MariaDB database |
