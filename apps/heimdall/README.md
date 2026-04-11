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

---

## Access

Default ingress: `https://home.example.com`

After first launch, add application tiles via the web UI. Heimdall supports
enhanced app integrations (live stats) for Sonarr, Radarr, Plex, and more.

---

## Storage

| PVC | Class | Size | Content |
|-----|-------|------|---------|
| `heimdall-config-pvc` | `local-path-fast` | 2Gi | Dashboard config + icons |
