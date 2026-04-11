# BaumKubernetes

Kubernetes manifests for the full BaumLab self-hosted stack. Each app is an independent,
namespaced deployment with `kustomize` support. Designed for a small ARM64 cluster
(Turing Pi 2 with RK1 / CM4 modules) running **K3s**, but compatible with any standard
Kubernetes distribution.

---

## Architecture

```
Cluster
├── ingress-nginx          ← reverse proxy / TLS termination
├── cert-manager           ← automatic Let's Encrypt certificates
├── storage
│   ├── local-path (fast)  ← NVMe node (databases, caches, ML models)
│   └── local-path (bulk)  ← SATA node (media libraries, large files)
└── apps/
    ├── authentik          ← SSO / identity provider (deploy first)
    ├── ai-stack           ← Ollama · LibreChat · AnythingLLM · n8n
    ├── immich             ← photo/video backup
    ├── jellyfin / plex    ← media servers
    ├── vaultwarden        ← password manager
    └── … 18 apps total
```

### Node Labels

Assign these labels to cluster nodes to match storage-tier placement:

```bash
# NVMe node (fast databases, AI, caches)
kubectl label node <fast-node-name> storage-tier=fast

# SATA / bulk node (media, large files)
kubectl label node <bulk-node-name> storage-tier=bulk
```

---

## Prerequisites

| Tool | Min Version | Notes |
|------|-------------|-------|
| [K3s](https://k3s.io) | v1.30+ | Recommended distro for ARM clusters |
| [kubectl](https://kubernetes.io/docs/tasks/tools/) | v1.30+ | |
| [kustomize](https://kustomize.io) | v5+ | Built into `kubectl apply -k` |
| [Helm](https://helm.sh) | v3.14+ | Used for ingress-nginx, cert-manager |

---

## Quick Start

### 1. Install K3s

```bash
# On the control-plane node
curl -sfL https://get.k3s.io | sh -

# On worker nodes (replace TOKEN and SERVER_IP)
curl -sfL https://get.k3s.io | K3S_URL=https://<SERVER_IP>:6443 K3S_TOKEN=<TOKEN> sh -
```

### 2. Label nodes

```bash
kubectl label node <fast-node> storage-tier=fast
kubectl label node <bulk-node> storage-tier=bulk
```

### 3. Set up cluster infrastructure

```bash
# Storage classes
kubectl apply -k cluster-setup/storage/

# ingress-nginx
kubectl apply -k cluster-setup/ingress-nginx/

# cert-manager
kubectl apply -k cluster-setup/cert-manager/
```

### 4. Deploy apps

Deploy **authentik first** — other apps optionally use it for SSO.

```bash
# Create secrets (see each app's README for required values)
kubectl apply -k apps/authentik/

# Then any other app
kubectl apply -k apps/vaultwarden/
kubectl apply -k apps/immich/
# …
```

---

## App Index

| App | Namespace | Description | Source |
|-----|-----------|-------------|--------|
| [ai-stack](apps/ai-stack/) | `ai-stack` | Ollama · LibreChat · AnythingLLM · n8n | [LibreChat](https://github.com/danny-avila/LibreChat) · [Ollama](https://github.com/ollama/ollama) · [n8n](https://github.com/n8n-io/n8n) |
| [authentik](apps/authentik/) | `authentik` | SSO / identity provider | [goauthentik/authentik](https://github.com/goauthentik/authentik) |
| [automatic-ripping-machine](apps/automatic-ripping-machine/) | `arm` | Optical disc ripper | [automatic-ripping-machine](https://github.com/automatic-ripping-machine/automatic-ripping-machine) |
| [calibre-web](apps/calibre-web/) | `calibre-web` | eBook library browser | [janeczku/calibre-web](https://github.com/janeczku/calibre-web) |
| [crafty](apps/crafty/) | `crafty` | Minecraft server manager | [crafty-controller](https://gitlab.com/crafty-controller/crafty-4) |
| [heimdall](apps/heimdall/) | `heimdall` | Application dashboard | [linuxserver/heimdall](https://github.com/linuxserver/Heimdall) |
| [immich](apps/immich/) | `immich` | Photo & video backup | [immich-app/immich](https://github.com/immich-app/immich) |
| [jellyfin](apps/jellyfin/) | `jellyfin` | Media server | [jellyfin/jellyfin](https://github.com/jellyfin/jellyfin) |
| [karakeep](apps/karakeep/) | `karakeep` | Bookmark manager | [karakeep-app/karakeep](https://github.com/karakeep-app/karakeep) |
| [mailcow](apps/mailcow/) | `mailcow` | Full email stack | [mailcow/mailcow-dockerized](https://github.com/mailcow/mailcow-dockerized) |
| [meshcentral](apps/meshcentral/) | `meshcentral` | Remote device management | [ylianst/MeshCentral](https://github.com/ylianst/MeshCentral) |
| [mumble](apps/mumble/) | `mumble` | Voice chat server | [mumble-voip/mumble](https://github.com/mumble-voip/mumble) |
| [nextcloud](apps/nextcloud/) | `nextcloud` | File sync & collaboration | [nextcloud/server](https://github.com/nextcloud/server) |
| [nginx-proxy-manager](apps/nginx-proxy-manager/) | `npm` | GUI reverse proxy | [NginxProxyManager/nginx-proxy-manager](https://github.com/NginxProxyManager/nginx-proxy-manager) |
| [plex](apps/plex/) | `plex` | Media server | [plexinc/pms-docker](https://github.com/plexinc/pms-docker) |
| [uptime-kuma](apps/uptime-kuma/) | `uptime-kuma` | Uptime monitoring | [louislam/uptime-kuma](https://github.com/louislam/uptime-kuma) |
| [vaultwarden](apps/vaultwarden/) | `vaultwarden` | Bitwarden-compatible password manager | [dani-garcia/vaultwarden](https://github.com/dani-garcia/vaultwarden) |
| [wordpress](apps/wordpress/) | `wordpress` | CMS | [wordpress/wordpress](https://github.com/WordPress/WordPress) |

---

## Storage

K3s ships with the **local-path provisioner** by default. This repo adds two named
`StorageClass` resources that target nodes by label:

| StorageClass | Node Label | Use Case |
|--------------|-----------|----------|
| `local-path-fast` | `storage-tier=fast` | Databases, caches, ML models |
| `local-path-bulk` | `storage-tier=bulk` | Media libraries, large files |

For distributed storage across nodes, replace with
[Longhorn](https://longhorn.io) — see [cluster-setup/storage/README.md](cluster-setup/storage/README.md).

---

## Secrets

Secrets are **never committed**. Each app has a `secret.example.yaml` showing required
keys. Create the actual secret before deploying:

```bash
# Generate a strong key
openssl rand -hex 32

# Create a secret manually
kubectl create secret generic vaultwarden-secret \
  --namespace vaultwarden \
  --from-literal=ADMIN_TOKEN="$(openssl rand -hex 32)"
```

Or edit `secret.example.yaml`, fill in values, rename to `secret.yaml` (gitignored), and:

```bash
kubectl apply -f apps/vaultwarden/secret.yaml
```

---

## Part of BaumLab

| Project | Description |
|---------|-------------|
| **[BaumConfigure](https://github.com/Bruiserbaum/BaumConfigure)** | OS image builder for ARM/x64 nodes |
| **[BaumDocker](https://github.com/Bruiserbaum/BaumDocker)** | Docker Compose / Swarm version of this stack |
| **[BaumKubernetes](https://github.com/Bruiserbaum/BaumKubernetes)** | *(this repo)* Kubernetes manifests |
| **[BaumLaunch](https://github.com/Bruiserbaum/BaumLaunch)** | WinGet GUI with system tray updater |

---

## License

Apache License 2.0. See LICENSE for details.
