# MeshCentral

**Remote device management** — open-source remote desktop, file transfer, and
terminal access for all your managed nodes.

**Source:** [ylianst/MeshCentral](https://github.com/ylianst/MeshCentral)
**Image:** `ghcr.io/ylianst/meshcentral:latest`
**Namespace:** `meshcentral`

---

## Deploy

```bash
# Edit configmap.yaml — set HOSTNAME to your domain
kubectl apply -k apps/meshcentral/
```

No secrets required for basic deployment. Admin account created on first visit.

## Deploy via Portainer

1. In Portainer: **Kubernetes** → **Manifests** → **Deploy**
2. Select **Repository**
3. Enter:
   - Repository URL: `https://github.com/Bruiserbaum/BaumKubernetes`
   - Repository reference: `refs/heads/master`
   - Manifest path: `apps/meshcentral/manifest.yaml`
   - Namespace: `meshcentral`
4. Click **Deploy**

---

## Configuration

MeshCentral is configured via environment variables mapped to its `config.json`.
Key settings in `configmap.yaml`:

| Env Var | Description |
|---------|-------------|
| `HOSTNAME` | Public FQDN, e.g. `mesh.example.com` |
| `ALLOW_NEW_ACCOUNTS` | `true` for initial setup, then `false` |
| `REVERSE_PROXY` | Set to ingress controller IP |
| `REVERSE_PROXY_TLS_PORT` | `443` |

---

## Agent installation (on managed nodes)

After deploying, go to **My Mesh → Add Agent**, download the installer for your
platform, and run it on each node you want to manage.

The agent connects back to `https://mesh.example.com` over WebSocket.

---

## Storage

| PVC | Class | Size | Content |
|-----|-------|------|---------|
| `meshcentral-data-pvc` | `longhorn-nvme` | 5Gi | Agent data, recordings, config |
