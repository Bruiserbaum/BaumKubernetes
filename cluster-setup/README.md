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

## RK1 Only — Mount NVMe for Longhorn

The RK1's NVMe is used as the fast storage tier (`longhorn-nvme`). Mount it at
`/var/lib/longhorn` so Longhorn uses it as its primary data directory on this node.

```bash
# Find the NVMe device
lsblk | grep nvme
# Typically: nvme0n1

# Partition and format (skip if already formatted)
sudo parted /dev/nvme0n1 --script mklabel gpt mkpart primary ext4 0% 100%
sudo mkfs.ext4 /dev/nvme0n1p1

# Mount at Longhorn's data directory
sudo mkdir -p /var/lib/longhorn
sudo mount /dev/nvme0n1p1 /var/lib/longhorn

# Persist across reboots
UUID=$(sudo blkid -s UUID -o value /dev/nvme0n1p1)
echo "UUID=$UUID  /var/lib/longhorn  ext4  defaults,noatime  0  2" | sudo tee -a /etc/fstab
```

After mounting, tag the disk `nvme` in the Longhorn UI (Node → Edit node and disks).

---

## CM4 Nodes — Mount SSD for Longhorn

The CM4 SSDs are used as the bulk storage tier (`longhorn-ssd`). Mount at
`/var/lib/longhorn` on each CM4.

```bash
# Find the SSD device
lsblk
# Typically: sda or mmcblk1 (USB SSD) or nvme0n1 (NVMe hat)

# Partition and format (replace sda with your device)
sudo parted /dev/sda --script mklabel gpt mkpart primary ext4 0% 100%
sudo mkfs.ext4 /dev/sda1

# Mount at Longhorn's data directory
sudo mkdir -p /var/lib/longhorn
sudo mount /dev/sda1 /var/lib/longhorn

# Persist across reboots
UUID=$(sudo blkid -s UUID -o value /dev/sda1)
echo "UUID=$UUID  /var/lib/longhorn  ext4  defaults,noatime  0  2" | sudo tee -a /etc/fstab
```

After mounting, tag the disk `ssd` in the Longhorn UI (Node → Edit node and disks).

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

### 1. Install Longhorn

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace

# Watch until all pods are Running
kubectl get pods -n longhorn-system -w
```

### 2. Label disks for storage tiers

In the Longhorn UI (port-forward to `svc/longhorn-frontend`), go to
**Node → Edit node and disks** and tag each disk:

- RK1 NVMe disk → tag `nvme`
- CM4-1 SSD disk → tag `ssd`
- CM4-2 SSD disk → tag `ssd`

See `cluster-setup/storage/README.md` for details and the kubectl alternative.

### 3. Apply StorageClasses

```bash
kubectl apply -k cluster-setup/storage/
```

Verify:

```bash
kubectl get storageclass
# NAME               PROVISIONER          RECLAIMPOLICY
# longhorn (default) driver.longhorn.io   Delete
# longhorn-nvme      driver.longhorn.io   Delete
# longhorn-ssd       driver.longhorn.io   Delete
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
kubectl get storageclass           # longhorn default, nvme + ssd present
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
