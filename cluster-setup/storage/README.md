# Storage Setup

BaumKubernetes uses **Longhorn** as the distributed storage backend with two
StorageClasses that target disks by label.

## StorageClasses

| Class | Disk Label | Backing | Typical Use |
|-------|-----------|---------|-------------|
| `longhorn-nvme` | `nvme` | NVMe SSD | Databases, Redis, ML model caches |
| `longhorn-ssd` | `ssd` | SATA/USB SSD | Media libraries, large file stores |

Both classes replicate to 2 nodes (`numberOfReplicas: "2"`).

---

## Install Longhorn

```bash
# Add Helm repo
helm repo add longhorn https://charts.longhorn.io
helm repo update

# Install into longhorn-system namespace
helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace

# Verify all pods are running
kubectl get pods -n longhorn-system -w
```

Access the Longhorn UI via port-forward (or add an ingress):

```bash
kubectl port-forward -n longhorn-system svc/longhorn-frontend 8080:80
# Open http://localhost:8080
```

---

## Label disks in Longhorn

Longhorn routes volumes to disks based on tags set in the UI or via API.

### In the Longhorn UI

1. Go to **Node** → click a node → **Edit node and disks**
2. For each disk, add the appropriate tag:
   - NVMe disks (RK1): add tag `nvme`
   - SSD disks (CM4-1, CM4-2): add tag `ssd`
3. Save

### Via kubectl (Longhorn API)

```bash
# Tag a disk on a node — replace NODE and DISK names with values from the UI
kubectl -n longhorn-system patch node.longhorn.io <NODE> \
  --type merge \
  --patch '{"spec":{"disks":{"<DISK-NAME>":{"tags":["nvme"]}}}}'
```

---

## Apply StorageClasses

```bash
kubectl apply -k cluster-setup/storage/
```

Verify:

```bash
kubectl get storageclass
# NAME             PROVISIONER          RECLAIMPOLICY
# longhorn (default) driver.longhorn.io  Delete
# longhorn-nvme    driver.longhorn.io   Delete
# longhorn-ssd     driver.longhorn.io   Delete
```

---

## Physical disk setup

Longhorn manages its own volume directories. Ensure each disk is mounted and
visible to the Longhorn node agent. By default Longhorn uses
`/var/lib/longhorn` — for a dedicated disk, mount it there:

```bash
# Format and mount (example for NVMe on RK1)
sudo parted /dev/nvme0n1 --script mklabel gpt mkpart primary ext4 0% 100%
sudo mkfs.ext4 /dev/nvme0n1p1
sudo mkdir -p /var/lib/longhorn
sudo mount /dev/nvme0n1p1 /var/lib/longhorn

# Persist
UUID=$(sudo blkid -s UUID -o value /dev/nvme0n1p1)
echo "UUID=$UUID  /var/lib/longhorn  ext4  defaults,noatime  0  2" | sudo tee -a /etc/fstab
```

Repeat for each CM4 SSD, mounting at `/var/lib/longhorn` on each node.
