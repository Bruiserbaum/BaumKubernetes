# Crafty Controller

**Minecraft server manager** — web-based GUI for managing multiple Minecraft server
instances (Java and Bedrock), with one-click install, backups, and scheduling.

**Source:** [crafty-controller/crafty-4](https://gitlab.com/crafty-controller/crafty-4)
**Image:** `registry.gitlab.com/crafty-controller/crafty-4:latest`
**Namespace:** `crafty`

---

## Deploy

```bash
kubectl apply -k apps/crafty/
```

No secrets required. Admin credentials set on first launch.

## Deploy via Portainer

1. In Portainer: **Kubernetes** → **Manifests** → **Deploy**
2. Select **Repository**
3. Enter:
   - Repository URL: `https://github.com/Bruiserbaum/BaumKubernetes`
   - Repository reference: `refs/heads/master`
   - Manifest path: `apps/crafty/manifest.yaml`
   - Namespace: `crafty`
4. Click **Deploy**

---

## Access

- Admin UI: `https://crafty.example.com` (port 8443 HTTPS)
- Minecraft Java: NodePort 25565–25567 (direct to node IP)
- Minecraft Bedrock: NodePort 19132 UDP

---

## Minecraft ports

Minecraft clients connect directly via NodePort (not through ingress). Point your
DNS / firewall to the cluster node IP:

| Server type | Port |
|------------|------|
| Java Edition | 25565 (default), 25566, 25567 |
| Bedrock Edition | 19132 UDP |

---

## Storage

| PVC | Class | Size | Content |
|-----|-------|------|---------|
| `crafty-data-pvc` | `longhorn-ssd` | 50Gi | Server jars, worlds, backups, configs |
