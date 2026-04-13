# Uptime Kuma

**Self-hosted monitoring tool** — monitors services, sends alerts via Slack, Discord,
email, and more. Beautiful status page included.

**Source:** [louislam/uptime-kuma](https://github.com/louislam/uptime-kuma)
**Image:** `louislam/uptime-kuma:latest`
**Namespace:** `uptime-kuma`

---

## Deploy

```bash
kubectl apply -k apps/uptime-kuma/
```

No secrets required. Admin account is created on first visit.

## Deploy via Portainer

1. In Portainer: **Kubernetes** → **Manifests** → **Deploy**
2. Select **Repository**
3. Enter:
   - Repository URL: `https://github.com/Bruiserbaum/BaumKubernetes`
   - Repository reference: `refs/heads/master`
   - Manifest path: `apps/uptime-kuma/manifest.yaml`
   - Namespace: `uptime-kuma`
4. Click **Deploy**

---

## Access

Default ingress: `https://status.example.com`

On first visit, create your admin account immediately — Uptime Kuma has no
pre-auth protection until the first account is created.

---

## Storage

| PVC | Class | Size | Content |
|-----|-------|------|---------|
| `uptime-kuma-pvc` | `longhorn-nvme` | 5Gi | SQLite database + config |

---

## Monitoring other cluster services

Add monitors pointing to internal ClusterIP service names, e.g.:

| Monitor | URL |
|---------|-----|
| Authentik | `http://authentik.authentik.svc.cluster.local:9000` |
| Immich | `http://immich-server.immich.svc.cluster.local:2283` |
| Vaultwarden | `http://vaultwarden.vaultwarden.svc.cluster.local:80` |
