# Storage Setup

BaumKubernetes uses two `StorageClass` resources that target nodes by label,
replacing the GlusterFS fast/bulk tier split from BaumDocker.

## StorageClasses

| Class | Node Label | Backing | Typical Use |
|-------|-----------|---------|-------------|
| `local-path-fast` | `storage-tier=fast` | NVMe | Databases, Redis, ML model caches |
| `local-path-bulk` | `storage-tier=bulk` | SATA / HDD | Media libraries, large file uploads |

K3s ships with `local-path-provisioner` by default. These classes are thin wrappers
that add node affinity via `volumeBindingMode: WaitForFirstConsumer`.

## Label your nodes

```bash
kubectl label node <nvme-node-name>  storage-tier=fast
kubectl label node <sata-node-name>  storage-tier=bulk
```

Check labels:

```bash
kubectl get nodes --show-labels | grep storage-tier
```

## Apply

```bash
kubectl apply -k cluster-setup/storage/
```

---

## Alternative: Longhorn (recommended for HA)

[Longhorn](https://longhorn.io) provides distributed block storage with replication,
snapshots, and a web UI. It is the recommended replacement for GlusterFS on K8s.

### Install via Helm

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace \
  --set defaultSettings.defaultDataPath="/mnt/longhorn"
```

### Create tiered StorageClasses

```yaml
# fast tier — NVMe nodes only
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-fast
provisioner: driver.longhorn.io
parameters:
  numberOfReplicas: "1"
  nodeSelector: "storage-tier=fast"
  dataLocality: strict-local
```

After switching to Longhorn, change `storageClassName` in any PVC from
`local-path-fast` / `local-path-bulk` to `longhorn-fast` / `longhorn-bulk`.

---

## NFS for media libraries

For large media collections shared between multiple pods (Jellyfin + ARM + Plex), NFS
is more practical than block storage:

```bash
# On the storage node
sudo apt install nfs-kernel-server
echo "/mnt/media  *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports
sudo exportfs -ra
```

Then use a PV + PVC with `nfs` driver instead of local-path-bulk for media apps.
See each media app README for the NFS PV template.
