# Calibre-Web

**eBook library browser** — browse, read, and download books from a Calibre library.
Supports OPDS for e-reader apps (KOReader, PocketBook, etc.).

**Source:** [janeczku/calibre-web](https://github.com/janeczku/calibre-web)
**Image:** `lscr.io/linuxserver/calibre-web:latest`
**Namespace:** `calibre-web`

---

## Deploy

```bash
kubectl apply -k apps/calibre-web/
```

No secrets required.

---

## Initial setup

On first launch, Calibre-Web asks for the path to your Calibre library
(the folder containing `metadata.db`). Set it to `/books` (mapped from the PVC).

Default admin credentials: `admin` / `admin123` — **change immediately**.

---

## Storage

| PVC | Class | Size | Content |
|-----|-------|------|---------|
| `calibre-web-config-pvc` | `local-path-fast` | 2Gi | App config + database |
| `calibre-web-books-pvc` | `local-path-bulk` | 200Gi | Calibre library (books) |

If your book collection is on a NAS, use an NFS PV for `calibre-web-books-pvc`
and point it at your existing Calibre library folder.

---

## OPDS access

Calibre-Web exposes an OPDS catalog at `/opds`. Point your e-reader app to:
`https://calibre.example.com/opds`
