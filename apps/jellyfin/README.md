# Jellyfin

**Open-source media server** — stream movies, TV, and music to any device.

**Source:** [jellyfin/jellyfin](https://github.com/jellyfin/jellyfin)
**Image:** `lscr.io/linuxserver/jellyfin:latest`
**Namespace:** `jellyfin`

---

## Deploy

```bash
# Edit configmap.yaml — set TZ, PUID, PGID
# Edit pvc.yaml — adjust media PVC sizes or swap to NFS PV
# Edit ingress.yaml — set host
kubectl apply -k apps/jellyfin/
```

No secrets required for basic deployment.

## Deploy via Portainer

1. In Portainer: **Kubernetes** → **Manifests** → **Deploy**
2. Select **Repository**
3. Enter:
   - Repository URL: `https://github.com/Bruiserbaum/BaumKubernetes`
   - Compose path: `apps/jellyfin/kustomization.yaml`
   - Namespace: `jellyfin`
4. Click **Deploy**

---

## Storage

| PVC | Class | Size | Content |
|-----|-------|------|---------|
| `jellyfin-config-pvc` | `local-path-bulk` | 20Gi | Config, metadata, transcodes |
| `jellyfin-movies-pvc` | `local-path-bulk` | 2Ti | Movies library |
| `jellyfin-tv-pvc` | `local-path-bulk` | 2Ti | TV shows library |
| `jellyfin-music-pvc` | `local-path-bulk` | 500Gi | Music library |

### NFS alternative (recommended for large libraries)

Instead of local-path PVCs for media, use a static NFS PV:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jellyfin-movies-nfs
spec:
  capacity:
    storage: 10Ti
  accessModes: [ReadWriteMany]
  nfs:
    server: 192.168.1.x
    path: /mnt/media/movies
  persistentVolumeReclaimPolicy: Retain
```

---

## Hardware transcoding

To enable hardware acceleration on ARM (RK3588 / CM4), add device mounts to `deployment.yaml`:

```yaml
# RK3588 (RK1) — Rockchip MPP
securityContext:
  privileged: true
volumeMounts:
  - name: dri
    mountPath: /dev/dri
volumes:
  - name: dri
    hostPath:
      path: /dev/dri
```

Then in Jellyfin → Dashboard → Playback → Hardware Acceleration: select **Video4Linux2 (V4L2)**.

---

## LAN discovery

Jellyfin uses UDP broadcasts for local network discovery. To support this,
add a `hostNetwork: true` line to the pod spec or expose UDP ports via NodePort.
