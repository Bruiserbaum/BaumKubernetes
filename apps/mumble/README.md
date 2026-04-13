# Mumble

**Low-latency voice chat server** — open-source, end-to-end encrypted, very
lightweight. Uses Murmur (the server component).

**Source:** [mumble-voip/mumble](https://github.com/mumble-voip/mumble)
**Image:** `mumblevoip/mumble-server:latest`
**Namespace:** `mumble`

---

## Secrets required

```bash
kubectl create namespace mumble

kubectl create secret generic mumble-secret \
  --namespace mumble \
  --from-literal=MUMBLE_SUPERUSER_PASSWORD="$(openssl rand -hex 16)"
```

---

## Deploy

```bash
kubectl apply -k apps/mumble/
```

## Deploy via Portainer

> Create the namespace and secret first using the `kubectl` commands in **Secrets required** above.

1. In Portainer: **Kubernetes** → **Manifests** → **Deploy**
2. Select **Repository**
3. Enter:
   - Repository URL: `https://github.com/Bruiserbaum/BaumKubernetes`
   - Repository reference: `refs/heads/master`
   - Manifest path: `apps/mumble/manifest.yaml`
   - Namespace: `mumble`
4. Click **Deploy**

---

## Client apps

- [Mumble Desktop](https://www.mumble.info/downloads/) (Windows / macOS / Linux)
- [Mumla](https://f-droid.org/packages/se.lublin.mumla/) (Android)
- Connect to `mumble.example.com:64738`

---

## NodePort for UDP voice traffic

Mumble requires UDP port 64738 for voice. Since ingress-nginx only handles TCP/HTTP,
expose Mumble via a `NodePort` service and point your DNS/firewall to the node IP.

The manifest below uses NodePort 64738 for both TCP (signaling) and UDP (voice).

---

## Storage

| PVC | Class | Size | Content |
|-----|-------|------|---------|
| `mumble-pvc` | `longhorn-nvme` | 1Gi | SQLite database, SSL cert |
