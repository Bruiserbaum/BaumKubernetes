# Heimdall

**Application dashboard** — bookmark and launch all your self-hosted services from
a single, clean homepage.

**Source:** [linuxserver/Heimdall](https://github.com/linuxserver/Heimdall)
**Image:** `lscr.io/linuxserver/heimdall:latest`
**Namespace:** `heimdall`

---

## Deploy

```bash
kubectl apply -k apps/heimdall/
```

No secrets required.

## Deploy via Portainer

1. In Portainer: **Kubernetes** → **Manifests** → **Deploy**
2. Select **Repository**
3. Enter:
   - Repository URL: `https://github.com/Bruiserbaum/BaumKubernetes`
   - Compose path: `apps/heimdall/kustomization.yaml`
   - Namespace: `heimdall`
4. Click **Deploy**

---

## Access

Default ingress: `https://home.example.com`

After first launch, add application tiles via the web UI. Heimdall supports
enhanced app integrations (live stats) for Sonarr, Radarr, Plex, and more.

---

## Storage

| PVC | Class | Size | Content |
|-----|-------|------|---------|
| `heimdall-config-pvc` | `longhorn-nvme` | 2Gi | Dashboard config + icons |
