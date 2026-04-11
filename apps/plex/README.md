# Plex

**Media server** — stream movies, TV, and music to any device with the Plex app.

**Source:** [plexinc/pms-docker](https://github.com/plexinc/pms-docker)
**Image:** `lscr.io/linuxserver/plex:latest`
**Namespace:** `plex`

---

## Plex Claim Token

Required on first deployment to link the server to your Plex account.

1. Log in at [plex.tv/claim](https://www.plex.tv/claim/)
2. Copy the token (valid 4 minutes)
3. Create secret:

```bash
kubectl create namespace plex

kubectl create secret generic plex-secret \
  --namespace plex \
  --from-literal=PLEX_CLAIM="claim-xxxxxxxxxxxxxxxxxxxx"
```

The claim is only used for initial registration; the secret can be deleted after first boot.

---

## Deploy

```bash
kubectl apply -k apps/plex/
```

---

## Network

Plex requires direct network access for local network discovery (GDM).
The manifest uses a `NodePort` service on port 32400 and a standard ingress for remote access.

For full local LAN discovery, add `hostNetwork: true` to the pod spec (see comment in deployment).

---

## Storage

| PVC | Class | Size | Content |
|-----|-------|------|---------|
| `plex-config-pvc` | `local-path-bulk` | 20Gi | Metadata, thumbnails, database |
| `plex-transcode-pvc` | `local-path-bulk` | 50Gi | Transcode buffer |
| `plex-movies-pvc` | `local-path-bulk` | 2Ti | Movies |
| `plex-tv-pvc` | `local-path-bulk` | 2Ti | TV shows |
| `plex-music-pvc` | `local-path-bulk` | 500Gi | Music |
