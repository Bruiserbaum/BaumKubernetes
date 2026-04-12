# Storage Setup

BaumKubernetes uses two `StorageClass` resources backed by `local-path-provisioner`.
Path selection is controlled by a `ConfigMap` that maps each node to its physical
storage device.

## Hardware layout

| Node | Device | Mount | StorageClass |
|------|--------|-------|--------------|
| RK1 (control plane) | NVMe SSD | `/mnt/nvme` | `local-path-fast` |
| CM4-1 (worker) | SATA/USB SSD | `/mnt/ssd` | `local-path-bulk` |
| CM4-2 (worker) | SATA/USB SSD | `/mnt/ssd` | `local-path-bulk` |

## StorageClasses

| Class | Node Label | Backing | Typical Use |
|-------|-----------|---------|-------------|
| `local-path-fast` | `storage-tier=fast` | NVMe `/mnt/nvme` | Databases, Redis, ML caches |
| `local-path-bulk` | `storage-tier=bulk` | SSD `/mnt/ssd` | Media libraries, large files |

`WaitForFirstConsumer` binding ensures PVCs only bind once a pod is scheduled, so
the provisioner creates the volume on the correct node automatically.

---

## Physical storage setup

Run these on each node **before** deploying any apps.

### RK1 — NVMe at `/mnt/nvme`

```bash
# Find the NVMe device (typically nvme0n1)
lsblk | grep nvme

# Partition and format (skip if already done)
sudo parted /dev/nvme0n1 --script mklabel gpt mkpart primary ext4 0% 100%
sudo mkfs.ext4 /dev/nvme0n1p1

# Mount
sudo mkdir -p /mnt/nvme
sudo mount /dev/nvme0n1p1 /mnt/nvme

# Persist across reboots
UUID=$(sudo blkid -s UUID -o value /dev/nvme0n1p1)
echo "UUID=$UUID  /mnt/nvme  ext4  defaults,noatime  0  2" | sudo tee -a /etc/fstab

# Create provisioner directory
sudo mkdir -p /mnt/nvme/local-path
```

### CM4-1 and CM4-2 — SSD at `/mnt/ssd`

```bash
# Find the SSD device (typically sda, mmcblk1, or nvme0n1 depending on hat)
lsblk

# Partition and format (replace sda with your device)
sudo parted /dev/sda --script mklabel gpt mkpart primary ext4 0% 100%
sudo mkfs.ext4 /dev/sda1

# Mount
sudo mkdir -p /mnt/ssd
sudo mount /dev/sda1 /mnt/ssd

# Persist across reboots
UUID=$(sudo blkid -s UUID -o value /dev/sda1)
echo "UUID=$UUID  /mnt/ssd  ext4  defaults,noatime  0  2" | sudo tee -a /etc/fstab

# Create provisioner directory
sudo mkdir -p /mnt/ssd/local-path
```

---

## Label nodes

```bash
# Replace with your actual node names (kubectl get nodes)
kubectl label node <rk1-node-name>   storage-tier=fast
kubectl label node <cm4-1-node-name> storage-tier=bulk
kubectl label node <cm4-2-node-name> storage-tier=bulk

# Verify
kubectl get nodes --show-labels | grep storage-tier
```

---

## Configure and apply

1. Edit `cluster-setup/storage/local-path-config.yaml` — replace the three placeholder
   node names with your actual node names from `kubectl get nodes`.

2. Apply everything:

```bash
kubectl apply -k cluster-setup/storage/
```

3. Verify:

```bash
kubectl get storageclass
# NAME                  PROVISIONER             RECLAIMPOLICY
# local-path (default)  rancher.io/local-path   Delete
# local-path-bulk       rancher.io/local-path   Retain
# local-path-fast       rancher.io/local-path   Retain

kubectl get configmap local-path-config -n local-path-storage -o yaml
```
