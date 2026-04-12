# Automatic Ripping Machine (ARM)

**Optical disc ripper** — automatically detects inserted discs and rips them to
MKV/MP4/FLAC. Supports Blu-ray, DVD, and CD.

**Source:** [automatic-ripping-machine/automatic-ripping-machine](https://github.com/automatic-ripping-machine/automatic-ripping-machine)
**Image:** `automaticrippingmachine/automatic-ripping-machine:latest`
**Namespace:** `arm`

---

## Requirements

- A physical optical drive accessible as `/dev/sr0` on the Kubernetes node
- The pod must run **privileged** and be scheduled on the node with the drive
- Label that node: `kubectl label node <node-name> has-optical-drive=true`

---

## Deploy

```bash
# Label the node with the optical drive
kubectl label node <node-name> has-optical-drive=true

kubectl apply -k apps/automatic-ripping-machine/
```

## Deploy via Portainer

> Label the node with `has-optical-drive=true` first (see above).

1. In Portainer: **Kubernetes** → **Manifests** → **Deploy**
2. Select **Repository**
3. Enter:
   - Repository URL: `https://github.com/Bruiserbaum/BaumKubernetes`
   - Compose path: `apps/automatic-ripping-machine/kustomization.yaml`
   - Namespace: `arm`
4. Click **Deploy**

---

## Storage

| PVC | Class | Size | Content |
|-----|-------|------|---------|
| `arm-media-pvc` | `local-path-bulk` | 1Ti | Ripped output files |
| `arm-config-pvc` | `local-path-bulk` | 5Gi | ARM config, logs |

---

## Web UI

Access the ARM web interface at `https://arm.example.com` to monitor ripping jobs,
configure output formats, and manage the library.

---

## Notes

- ARM runs **privileged** to access `/dev/sr0` — only deploy on a trusted, dedicated node
- Handbrake is included for transcoding to MP4/MKV
- MakeMKV handles Blu-ray decryption (license key required for Blu-ray support)
