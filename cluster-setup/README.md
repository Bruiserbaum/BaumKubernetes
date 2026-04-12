# Cluster Setup — RK1 (Control Plane) + 2× CM4 (Workers)

This guide covers a bare-metal kubeadm cluster on BaumLab hardware.
Run the steps in order. Commands marked **All Nodes**, **CM4 Only**, or **RK1 Only**
indicate which machines each section applies to.

---

## All Nodes — Run on RK1, CM4-1, CM4-2

### 1. Fix Ubuntu Repos (Oracular EOL)

```bash
sudo tee /etc/apt/sources.list << 'EOF'
deb http://old-releases.ubuntu.com/ubuntu oracular main restricted universe multiverse
deb http://old-releases.ubuntu.com/ubuntu oracular-updates main restricted universe multiverse
deb http://old-releases.ubuntu.com/ubuntu oracular-security main restricted universe multiverse
deb http://old-releases.ubuntu.com/ubuntu oracular-backports main restricted universe multiverse
EOF
sudo apt-get update
```

### 2. Pre-flight Config

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/' /etc/fstab

sudo tee /etc/modules-load.d/k8s.conf << 'EOF'
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

sudo tee /etc/sysctl.d/k8s.conf << 'EOF'
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```

### 3. Install Docker CE + containerd

```bash
sudo apt-get remove -y docker.io docker-doc docker-compose podman-docker
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 4. Configure containerd for Kubernetes

```bash
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### 5. Install Kubernetes Tools

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable kubelet
```

---

## CM4 Nodes Only — Fix Memory Cgroups

The CM4 Ubuntu image ships with memory cgroups disabled. Kubernetes requires them.

```bash
# Find active cmdline
cat /boot/firmware/current/cmdline.txt

# Remove explicit disable and enable memory cgroups
sudo sed -i 's/cgroup_disable=memory//' /boot/firmware/current/cmdline.txt
sudo sed -i 's/$/ cgroup_enable=memory cgroup_memory=1/' /boot/firmware/current/cmdline.txt

# Mirror to new/ for OTA survival
sudo sed -i 's/cgroup_disable=memory//' /boot/firmware/new/cmdline.txt
sudo sed -i 's/$/ cgroup_enable=memory cgroup_memory=1/' /boot/firmware/new/cmdline.txt

sudo reboot

# After reboot — verify (should show 1 in the enabled column)
grep memory /proc/cgroups
```

---

## RK1 Only — Mount NVMe for Fast Storage

The RK1's NVMe is used as the fast storage tier (`local-path-fast`) for databases
and latency-sensitive workloads.

```bash
# Find the NVMe device
lsblk | grep nvme
# Typically: nvme0n1

# Partition and format (skip if already formatted)
sudo parted /dev/nvme0n1 --script mklabel gpt mkpart primary ext4 0% 100%
sudo mkfs.ext4 /dev/nvme0n1p1

# Create mount point and mount
sudo mkdir -p /mnt/nvme
sudo mount /dev/nvme0n1p1 /mnt/nvme

# Persist across reboots
UUID=$(sudo blkid -s UUID -o value /dev/nvme0n1p1)
echo "UUID=$UUID  /mnt/nvme  ext4  defaults,noatime  0  2" | sudo tee -a /etc/fstab

# Create provisioner directory
sudo mkdir -p /mnt/nvme/local-path
```

---

## CM4 Nodes — Mount SSD for Bulk Storage

The CM4 SSDs are used as the bulk storage tier (`local-path-bulk`) for media
libraries and large file stores.

```bash
# Find the SSD device
lsblk
# Typically: sda or mmcblk1 (USB SSD) or nvme0n1 (NVMe hat)

# Partition and format (replace sda with your device)
sudo parted /dev/sda --script mklabel gpt mkpart primary ext4 0% 100%
sudo mkfs.ext4 /dev/sda1

# Create mount point and mount
sudo mkdir -p /mnt/ssd
sudo mount /dev/sda1 /mnt/ssd

# Persist across reboots
UUID=$(sudo blkid -s UUID -o value /dev/sda1)
echo "UUID=$UUID  /mnt/ssd  ext4  defaults,noatime  0  2" | sudo tee -a /etc/fstab

# Create provisioner directory
sudo mkdir -p /mnt/ssd/local-path
```

---

## RK1 Only — Initialize Control Plane

```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --cri-socket unix:///run/containerd/containerd.sock

# Set up kubeconfig
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install Flannel CNI
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# Watch nodes come Ready
watch kubectl get nodes

# Get join command for CM4s
kubeadm token create --print-join-command
```

---

## CM4 Nodes Only — Join Cluster

```bash
sudo kubeadm join <RK1-IP>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --cri-socket unix:///run/containerd/containerd.sock
```

---

## RK1 Only — Storage Classes + Portainer

### 1. Install local-path-provisioner

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml

# Make local-path the default (fallback for PVCs without an explicit class)
kubectl patch storageclass local-path \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Verify provisioner is running
kubectl get pods -n local-path-storage
```

### 2. Label nodes for storage tiers

```bash
# Replace node names with your actual node names (kubectl get nodes)
kubectl label node <rk1-node-name>   storage-tier=fast
kubectl label node <cm4-1-node-name> storage-tier=bulk
kubectl label node <cm4-2-node-name> storage-tier=bulk
```

### 3. Configure provisioner paths and apply storage classes

Edit `cluster-setup/storage/local-path-config.yaml` and replace the placeholder
node names with your actual node names, then apply:

```bash
kubectl apply -k cluster-setup/storage/
```

This creates the `local-path-fast` and `local-path-bulk` StorageClasses and
configures the provisioner to use `/mnt/nvme/local-path` on the RK1 and
`/mnt/ssd/local-path` on CM4 nodes.

Verify:

```bash
kubectl get storageclass
# NAME                PROVISIONER             RECLAIMPOLICY
# local-path (default)  rancher.io/local-path   Delete
# local-path-bulk       rancher.io/local-path   Retain
# local-path-fast       rancher.io/local-path   Retain
```

### 4. Deploy Portainer

```bash
kubectl create namespace portainer
kubectl apply -n portainer -f https://downloads.portainer.io/ce2-21/portainer.yaml

# Watch until Running
kubectl get pods -n portainer -w
```

Access Portainer at `http://<any-node-ip>:30777`

---

## Verify Cluster Health

```bash
kubectl get nodes -o wide          # All three nodes Ready
kubectl get pods -A                # All system pods Running
kubectl get storageclass           # local-path default, fast + bulk present
kubectl get pods -n portainer      # Portainer Running
```

---

## Quick Reference

| Task | Command |
|------|---------|
| Regenerate join token | `kubeadm token create --print-join-command` |
| Drain node for maintenance | `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data` |
| Restore node after maintenance | `kubectl uncordon <node>` |
| Check resource usage | `kubectl top nodes` |
| Restart a deployment | `kubectl rollout restart deployment/<name> -n <namespace>` |
| Remove control plane taint (run workloads on RK1) | `kubectl taint nodes <rk1-name> node-role.kubernetes.io/control-plane:NoSchedule-` |
| Show node labels | `kubectl get nodes --show-labels` |
| Check PVC status | `kubectl get pvc -A` |
