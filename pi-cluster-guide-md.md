# Raspberry Pi 5 Secure Cluster Setup Guide

## Overview

This guide walks you through building a 3-node Raspberry Pi 5 cluster with full disk encryption, PoE power, NVMe storage, and Kubernetes orchestration for running a private GitLab/Gitea, Docker registry, and development workloads.

**Total Cost**: ~$510 (without UPS) | ~$690 (with UPS)  
**Power Consumption**: ~45W total cluster  
**Setup Time**: 2-3 days (including testing)

---

## Table of Contents

1. [Hardware Requirements](#hardware-requirements)
2. [Network Architecture](#network-architecture)
3. [Pre-Installation Preparation](#pre-installation-preparation)
4. [Phase 1: Base System Setup](#phase-1-base-system-setup)
5. [Phase 2: NVMe & LUKS Encryption](#phase-2-nvme--luks-encryption)
6. [Phase 3: K3s Kubernetes Installation](#phase-3-k3s-kubernetes-installation)
7. [Phase 4: Storage Layer (Longhorn)](#phase-4-storage-layer-longhorn)
8. [Phase 5: Core Services Deployment](#phase-5-core-services-deployment)
9. [Phase 6: Security Hardening](#phase-6-security-hardening)
10. [Maintenance & Operations](#maintenance--operations)
11. [Troubleshooting](#troubleshooting)

---

## Hardware Requirements

### Per Node (3x Total)

| Component | Specification | Price | Source |
|-----------|--------------|-------|--------|
| Raspberry Pi 5 | 8GB RAM model | $80 | Official distributors |
| PoE+NVMe HAT | Waveshare M.2 HAT+ or GeeekPi P33 | $40 | Amazon/AliExpress |
| NVMe SSD | 512GB M.2 2280 (NVMe 1.3+) | $35 | Samsung/Crucial/WD |
| Case | Ventilated, HAT-compatible | $12 | Argon/GeeekPi |
| Ethernet Cable | Cat6, 3-6ft | $3 | Various |

**Recommended SSDs**:
- Samsung 980 (500GB) - Excellent performance, ~$35
- Crucial P3 (512GB) - Budget-friendly, ~$30
- WD Blue SN570 (500GB) - Reliable, ~$35

### Shared Infrastructure

| Component | Specification | Price | Notes |
|-----------|--------------|-------|-------|
| PoE+ Switch | 802.3at, 4+ ports, 25W/port | Existing | Ensure adequate power budget |
| UPS | 1500VA, 900W | $180 | CyberPower/APC (optional but recommended) |
| Network Router | Gigabit, DHCP | Existing | Configure static IPs |

### Power Budget Calculation

- Per Pi 5 with NVMe under load: ~15W
- Total cluster: 45W
- PoE switch overhead: ~10%
- **Minimum PoE budget**: 50W across 3 ports
- **Recommended**: 75W (25W per port)

---

## Network Architecture

### IP Address Assignment

```
Control Plane:
- pi-node1: 192.168.1.10

Worker Nodes:
- pi-node2: 192.168.1.11
- pi-node3: 192.168.1.12

Reserved IPs:
- Tang Server (optional): 192.168.1.9
- VIP for services: 192.168.1.20-29
```

### Network Diagram

```
Internet
    |
Router (192.168.1.1)
    |
PoE+ Switch
    |
    +---- UPS (Battery Backup)
    |
    +---- pi-node1 (192.168.1.10) - Control Plane
    +---- pi-node2 (192.168.1.11) - Worker
    +---- pi-node3 (192.168.1.12) - Worker
    +---- Management PC
```

### Port Requirements

**Open on firewall/router** (internal network):
- 22: SSH (management)
- 6443: Kubernetes API
- 10250: Kubelet API
- 2379-2380: etcd (control plane only)
- 30000-32767: NodePort services

---

## Pre-Installation Preparation

### 1. Download Required Software

```bash
# On your management PC

# Raspberry Pi Imager
wget https://downloads.raspberrypi.org/imager/imager_latest_amd64.deb

# Or from: https://www.raspberrypi.com/software/

# Download Raspberry Pi OS Lite (64-bit)
# Latest version from official site
```

### 2. Prepare SD Cards

You'll need temporary SD cards for initial setup (can reuse for all 3 nodes):

```bash
# Flash Raspberry Pi OS Lite (64-bit) using Imager
# Enable SSH in advanced options
# Set hostname: pi-node1 (change for each node)
# Set username/password
# Configure WiFi (optional, for initial setup)
```

### 3. Create Secure Credentials

```bash
# Generate SSH key pair for remote LUKS unlock
ssh-keygen -t ed25519 -f ~/.ssh/pi_cluster_unlock -C "cluster-unlock"

# Generate strong LUKS passphrases (24+ chars)
# Store in password manager: Bitwarden, 1Password, KeePass

# Example generation:
openssl rand -base64 32
```

### 4. Document Your Setup

Create a spreadsheet/note with:
- Node names and MAC addresses
- IP addresses
- LUKS passphrases (encrypted storage)
- SSH keys locations
- Network topology

---

## Phase 1: Base System Setup

### Node 1: Initial Boot and Configuration

```bash
# Insert SD card, connect to PoE switch (temporary power)
# SSH into node (find IP via router DHCP, or use raspberrypi.local)

ssh pi@192.168.1.x  # Initial DHCP IP

# Update system
sudo apt update && sudo apt upgrade -y

# Set static IP
sudo nmcli con mod "Wired connection 1" \
  ipv4.addresses 192.168.1.10/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "192.168.1.1,8.8.8.8" \
  ipv4.method manual

sudo nmcli con up "Wired connection 1"

# Set hostname
sudo hostnamectl set-hostname pi-node1

# Install essential packages
sudo apt install -y \
  vim git curl wget \
  cryptsetup cryptsetup-initramfs \
  dropbear-initramfs \
  open-iscsi nfs-common \
  net-tools htop iotop

# Enable cgroup for Kubernetes
sudo sed -i '$ s/$/ cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory/' \
  /boot/firmware/cmdline.txt

# Reboot to apply changes
sudo reboot
```

### Verify NVMe Detection

```bash
# After reboot, check NVMe detection
lsblk
# Should see: nvme0n1

# Check NVMe info
sudo nvme list
sudo nvme id-ctrl /dev/nvme0n1
```

---

## Phase 2: NVMe & LUKS Encryption

### Step 1: Partition NVMe Drive

```bash
# Create partition table
sudo parted /dev/nvme0n1 --script mklabel gpt

# Create boot partition (512MB)
sudo parted /dev/nvme0n1 --script mkpart boot fat32 1MiB 512MiB
sudo parted /dev/nvme0n1 --script set 1 boot on

# Create root partition (remaining space)
sudo parted /dev/nvme0n1 --script mkpart root ext4 512MiB 100%

# Verify
lsblk /dev/nvme0n1
```

### Step 2: Setup LUKS Encryption

```bash
# Format root partition with LUKS
# You'll be prompted for passphrase (use strong one!)
sudo cryptsetup luksFormat --type luks2 \
  --cipher aes-xts-plain64 \
  --key-size 512 \
  --hash sha256 \
  --iter-time 5000 \
  --use-random \
  /dev/nvme0n1p2

# Verify LUKS header
sudo cryptsetup luksDump /dev/nvme0n1p2

# Open encrypted partition
sudo cryptsetup luksOpen /dev/nvme0n1p2 cryptroot

# Create filesystems
sudo mkfs.vfat -F 32 -n BOOT /dev/nvme0n1p1
sudo mkfs.ext4 -L rootfs /dev/mapper/cryptroot

# Get UUIDs (save these!)
sudo blkid /dev/nvme0n1p1
sudo blkid /dev/nvme0n1p2
sudo blkid /dev/mapper/cryptroot
```

### Step 3: Mount and Migrate System

```bash
# Create mount points
sudo mkdir -p /mnt/nvme-root
sudo mkdir -p /mnt/nvme-boot

# Mount partitions
sudo mount /dev/mapper/cryptroot /mnt/nvme-root
sudo mkdir -p /mnt/nvme-root/boot/firmware
sudo mount /dev/nvme0n1p1 /mnt/nvme-root/boot/firmware

# Copy system to NVMe (this takes 10-15 minutes)
sudo rsync -axHAWXS --numeric-ids --info=progress2 \
  --exclude=/mnt \
  --exclude=/proc \
  --exclude=/sys \
  --exclude=/dev \
  --exclude=/tmp \
  / /mnt/nvme-root/

# Recreate system directories
sudo mkdir -p /mnt/nvme-root/{proc,sys,dev,tmp,mnt}
```

### Step 4: Configure Boot for Encryption

```bash
# Update /etc/fstab on new root
BOOT_UUID=$(sudo blkid -s UUID -o value /dev/nvme0n1p1)
ROOT_UUID=$(sudo blkid -s UUID -o value /dev/mapper/cryptroot)

sudo tee /mnt/nvme-root/etc/fstab << EOF
UUID=$ROOT_UUID  /               ext4    defaults,noatime  0  1
UUID=$BOOT_UUID  /boot/firmware  vfat    defaults          0  2
tmpfs            /tmp            tmpfs   defaults,noatime,mode=1777  0  0
EOF

# Update cmdline.txt for encrypted root
LUKS_UUID=$(sudo blkid -s UUID -o value /dev/nvme0n1p2)

# Backup original
sudo cp /mnt/nvme-root/boot/firmware/cmdline.txt \
     /mnt/nvme-root/boot/firmware/cmdline.txt.backup

# Update to use encrypted root
sudo sed -i "s|root=[^ ]*|root=/dev/mapper/cryptroot cryptdevice=UUID=$LUKS_UUID:cryptroot|" \
  /mnt/nvme-root/boot/firmware/cmdline.txt

# Add required kernel modules
echo "dm_crypt" | sudo tee -a /mnt/nvme-root/etc/initramfs-tools/modules
echo "aes" | sudo tee -a /mnt/nvme-root/etc/initramfs-tools/modules
```

### Step 5: Configure Remote Unlock (Dropbear SSH)

```bash
# Configure crypttab
echo "cryptroot UUID=$LUKS_UUID none luks,initramfs" | \
  sudo tee /mnt/nvme-root/etc/crypttab

# Setup Dropbear for remote unlock
sudo mkdir -p /mnt/nvme-root/etc/dropbear-initramfs/

# Copy your public key for unlock
cat ~/.ssh/pi_cluster_unlock.pub | \
  sudo tee /mnt/nvme-root/etc/dropbear-initramfs/authorized_keys

# Configure static IP for initramfs
sudo tee -a /mnt/nvme-root/etc/initramfs-tools/initramfs.conf << EOF
DEVICE=eth0
IP=192.168.1.10::192.168.1.1:255.255.255.0:pi-node1:eth0:off
EOF

# Configure Dropbear
sudo tee /mnt/nvme-root/etc/dropbear-initramfs/config << EOF
DROPBEAR_OPTIONS="-p 22 -s -j -k -I 60"
EOF

# Chroot and rebuild initramfs
sudo mount --bind /dev /mnt/nvme-root/dev
sudo mount --bind /proc /mnt/nvme-root/proc
sudo mount --bind /sys /mnt/nvme-root/sys

sudo chroot /mnt/nvme-root /bin/bash << 'CHROOT_EOF'
update-initramfs -u
CHROOT_EOF

# Unmount
sudo umount /mnt/nvme-root/sys
sudo umount /mnt/nvme-root/proc
sudo umount /mnt/nvme-root/dev
sudo umount /mnt/nvme-root/boot/firmware
sudo umount /mnt/nvme-root

# Close LUKS
sudo cryptsetup luksClose cryptroot
```

### Step 6: Update Firmware Boot Order

```bash
# Edit boot configuration
sudo nano /boot/firmware/config.txt

# Add at the end:
# Boot from NVMe
dtparam=pciex1
dtparam=nvme

# Save and exit

# Update bootloader to try NVMe first
sudo rpi-eeprom-config --edit
# Change BOOT_ORDER to: 0xf416 (NVMe->SD->USB->Network)
# Save and exit

# Reboot to test
sudo reboot
```

### Step 7: First Encrypted Boot & Unlock

```bash
# On your management PC
# Wait ~30 seconds after reboot for Dropbear to start

# Connect to unlock prompt
ssh -i ~/.ssh/pi_cluster_unlock root@192.168.1.10

# You'll see a minimal shell
# Unlock the drive:
cryptroot-unlock

# Enter your LUKS passphrase
# System will continue booting

# Wait 1 minute, then SSH normally
ssh pi@192.168.1.10
```

### Step 8: Verify and Clean Up

```bash
# Check boot device
lsblk
# / should be mounted from /dev/mapper/cryptroot

df -h
# Verify NVMe is in use

# Remove SD card (power off first)
sudo poweroff

# Remove SD card, power on via PoE only
# System should boot from NVMe with remote unlock
```

---

## Phase 3: K3s Kubernetes Installation

### Control Plane Node (pi-node1)

```bash
# Disable swap (required for K8s)
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Install K3s as server
curl -sfL https://get.k3s.io | sh -s - server \
  --write-kubeconfig-mode 644 \
  --disable traefik \
  --disable servicelb \
  --node-name pi-node1 \
  --node-ip 192.168.1.10 \
  --cluster-cidr=10.42.0.0/16 \
  --service-cidr=10.43.0.0/16

# Wait for node to be ready
sudo k3s kubectl get nodes

# Save token for worker nodes
sudo cat /var/lib/rancher/k3s/server/node-token > ~/k3s-token
cat ~/k3s-token
# Copy this token securely

# Setup kubectl access
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
chmod 600 ~/.kube/config

# Test
kubectl get nodes
kubectl get pods -A
```

### Worker Nodes (pi-node2, pi-node3)

**Repeat Phase 1 and Phase 2 for each worker node**, then:

```bash
# On pi-node2 (192.168.1.11)
# And pi-node3 (192.168.1.12)

# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Install K3s as agent
export K3S_URL="https://192.168.1.10:6443"
export K3S_TOKEN="<paste token from control plane>"

curl -sfL https://get.k3s.io | sh -s - agent \
  --node-name pi-node2 \  # Change for node3
  --node-ip 192.168.1.11   # Change for node3

# Verify (back on control plane)
kubectl get nodes
# Should show all 3 nodes as Ready
```

### Label Nodes

```bash
# On control plane
kubectl label node pi-node1 node-role.kubernetes.io/control-plane=true
kubectl label node pi-node2 node-role.kubernetes.io/worker=true
kubectl label node pi-node3 node-role.kubernetes.io/worker=true

# Verify
kubectl get nodes --show-labels
```

---

## Phase 4: Storage Layer (Longhorn)

### Prerequisites

```bash
# On ALL nodes
sudo apt-get install -y open-iscsi nfs-common
sudo systemctl enable iscsid
sudo systemctl start iscsid
sudo systemctl status iscsid
```

### Install Longhorn

```bash
# On control plane
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.5.3/deploy/longhorn.yaml

# Wait for deployment (5-10 minutes)
kubectl get pods -n longhorn-system -w

# Verify all pods are running
kubectl get pods -n longhorn-system

# Expose Longhorn UI (NodePort)
kubectl -n longhorn-system patch svc longhorn-frontend \
  -p '{"spec":{"type":"NodePort"}}'

# Get NodePort
kubectl -n longhorn-system get svc longhorn-frontend

# Access UI: http://192.168.1.10:<NodePort>
```

### Configure Longhorn Settings

```bash
# Set replica count to 2 (for 3-node cluster)
kubectl -n longhorn-system patch settings.longhorn.io default-replica-count \
  --type=json \
  -p='[{"op": "replace", "path": "/value", "value": "2"}]'

# Set data locality to best-effort
kubectl -n longhorn-system patch settings.longhorn.io default-data-locality \
  --type=json \
  -p='[{"op": "replace", "path": "/value", "value": "best-effort"}]'
```

### Set Longhorn as Default StorageClass

```bash
# Mark Longhorn as default
kubectl patch storageclass longhorn \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Verify
kubectl get storageclass
```

---

## Phase 5: Core Services Deployment

### Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

### Deploy Traefik Ingress Controller

```bash
# Create namespace
kubectl create namespace traefik

# Add Helm repo
helm repo add traefik https://traefik.github.io/charts
helm repo update

# Install Traefik
helm install traefik traefik/traefik \
  --namespace traefik \
  --set service.type=LoadBalancer \
  --set ports.web.nodePort=30080 \
  --set ports.websecure.nodePort=30443

# Get service IP
kubectl get svc -n traefik

# Access dashboard: http://192.168.1.10:30080/dashboard/
```

### Deploy Gitea (Lightweight Git Service)

```bash
# Create namespace
kubectl create namespace gitea

# Create Gitea deployment
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitea-data
  namespace: gitea
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitea
  namespace: gitea
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gitea
  template:
    metadata:
      labels:
        app: gitea
    spec:
      containers:
      - name: gitea
        image: gitea/gitea:1.21-rootless
        ports:
        - containerPort: 3000
          name: http
        - containerPort: 2222
          name: ssh
        volumeMounts:
        - name: data
          mountPath: /var/lib/gitea
        - name: config
          mountPath: /etc/gitea
        env:
        - name: USER_UID
          value: "1000"
        - name: USER_GID
          value: "1000"
        - name: GITEA__database__DB_TYPE
          value: "sqlite3"
        - name: GITEA__server__DOMAIN
          value: "git.cluster.local"
        - name: GITEA__server__ROOT_URL
          value: "http://192.168.1.10:30300"
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: gitea-data
      - name: config
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: gitea
  namespace: gitea
spec:
  type: NodePort
  ports:
  - name: http
    port: 3000
    targetPort: 3000
    nodePort: 30300
  - name: ssh
    port: 2222
    targetPort: 2222
    nodePort: 30222
  selector:
    app: gitea
EOF

# Wait for pod to be ready
kubectl get pods -n gitea -w

# Access Gitea: http://192.168.1.10:30300
```

### Deploy Docker Registry (Harbor)

```bash
# Add Harbor Helm repo
helm repo add harbor https://helm.goharbor.io
helm repo update

# Create namespace
kubectl create namespace harbor

# Install Harbor
helm install harbor harbor/harbor \
  --namespace harbor \
  --set expose.type=nodePort \
  --set expose.tls.enabled=false \
  --set persistence.enabled=true \
  --set persistence.persistentVolumeClaim.registry.storageClass=longhorn \
  --set persistence.persistentVolumeClaim.registry.size=50Gi \
  --set externalURL=http://192.168.1.10:30002 \
  --set harborAdminPassword=admin123

# Get NodePort
kubectl get svc -n harbor

# Access Harbor: http://192.168.1.10:30002
# Default credentials: admin / admin123
```

### Deploy Monitoring Stack (Prometheus + Grafana)

```bash
# Add Prometheus Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create namespace
kubectl create namespace monitoring

# Install kube-prometheus-stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=longhorn \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=10Gi \
  --set grafana.adminPassword=admin123 \
  --set grafana.service.type=NodePort \
  --set grafana.service.nodePort=30000

# Wait for all pods
kubectl get pods -n monitoring -w

# Access Grafana: http://192.168.1.10:30000
# Credentials: admin / admin123
```

---

## Phase 6: Security Hardening

### Configure UFW Firewall

```bash
# On ALL nodes

# Install UFW
sudo apt-get install -y ufw

# Default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH
sudo ufw allow from 192.168.1.0/24 to any port 22

# Allow Kubernetes ports
sudo ufw allow from 192.168.1.0/24 to any port 6443  # API
sudo ufw allow from 192.168.1.0/24 to any port 10250 # Kubelet
sudo ufw allow from 192.168.1.0/24 to any port 2379:2380/tcp # etcd

# Allow NodePort range
sudo ufw allow from 192.168.1.0/24 to any port 30000:32767/tcp

# Allow Longhorn
sudo ufw allow from 192.168.1.0/24 to any port 9500:9504/tcp

# Enable firewall
sudo ufw --force enable

# Check status
sudo ufw status numbered
```

### Configure Automatic Updates

```bash
# On ALL nodes

# Install unattended-upgrades
sudo apt-get install -y unattended-upgrades apt-listchanges

# Configure
sudo dpkg-reconfigure -plow unattended-upgrades

# Edit config
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades

# Uncomment and set:
# Unattended-Upgrade::Automatic-Reboot "false";
# Unattended-Upgrade::Automatic-Reboot-Time "03:00";

# Test
sudo unattended-upgrade --dry-run --debug
```

### SSH Hardening

```bash
# On ALL nodes

sudo nano /etc/ssh/sshd_config

# Recommended settings:
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
X11Forwarding no
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2

# Restart SSH
sudo systemctl restart sshd
```

### Setup Fail2ban

```bash
# On ALL nodes

sudo apt-get install -y fail2ban

# Configure
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local

# Under [sshd] section:
enabled = true
port = ssh
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600

# Start service
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

# Check status
sudo fail2ban-client status sshd
```

---

## Maintenance & Operations

### Daily Operations

#### Check Cluster Health

```bash
# Node status
kubectl get nodes

# Pod status across all namespaces
kubectl get pods -A

# Check Longhorn volumes
kubectl get pv

# Check resource usage
kubectl top nodes
kubectl top pods -A
```

#### Backup Configuration

```bash
# Backup K3s config
sudo cp /etc/rancher/k3s/k3s.yaml ~/k3s-backup-$(date +%Y%m%d).yaml

# Backup Longhorn volumes (via UI)
# Navigate to http://192.168.1.10:<longhorn-port>
# Backup -> Create backup for critical volumes
```

### Weekly Operations

#### Update Applications

```bash
# Update Helm repos
helm repo update

# List installed charts
helm list -A

# Upgrade specific chart (example: Gitea)
helm upgrade gitea gitea/gitea -n gitea

# Check rollout status
kubectl rollout status deployment/gitea -n gitea
```

#### Check System Updates

```bash
# On each node
ssh pi@192.168.1.10
sudo apt update
sudo apt list --upgradable

# Apply updates (on control plane first)
sudo apt upgrade -y
sudo reboot

# Wait for node to rejoin cluster
kubectl get nodes -w
```

### Monthly Operations

#### Certificate Rotation

```bash
# K3s auto-rotates certificates
# Check expiry
sudo k3s certificate check

# Manual rotation if needed
sudo systemctl stop k3s
sudo k3s certificate rotate
sudo systemctl start k3s
```

#### Storage Cleanup

```bash
# Clean up unused images
kubectl get pods -A -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}' | sort -u

# Prune on each node
sudo crictl rmi --prune

# Clean up completed pods
kubectl delete pods --field-selector=status.phase==Succeeded -A
kubectl delete pods --field-selector=status.phase==Failed -A
```

### Disaster Recovery

#### Node Failure Scenario

```bash
# If a node fails, K8s automatically reschedules pods

# Check node status
kubectl get nodes

# If node is NotReady, investigate
kubectl describe node pi-node2

# Drain node for maintenance
kubectl drain pi-node2 --ignore-daemonsets --delete-emptydir-data

# After fixing, uncordon
kubectl uncordon pi-node2
```

#### Restore from Backup

```bash
# If cluster is lost, reinstall K3s with same token
# On new control plane:
curl -sfL https://get.k3s.io | K3S_TOKEN=<old-token> sh -s - server

# Restore kubeconfig
cp ~/k3s-backup-YYYYMMDD.yaml ~/.kube/config

# Restore Longhorn volumes from backup
# Via Longhorn UI or CLI
```

---

## Troubleshooting

### Issue: Node Not Booting from NVMe

**Symptoms**: System boots from SD card instead of NVMe

**Solution**:
```bash
# Check boot order
sudo rpi-eeprom-config

# Update if needed
sudo rpi-eeprom-config --edit
# Set BOOT_ORDER=0xf416

sudo rpi-eeprom-update -a
sudo reboot
```

### Issue: Cannot Unlock LUKS Remotely

**Symptoms**: Cannot SSH to port 22 during boot

**Solution**:
```bash
# Connect monitor/keyboard to Pi
# Or boot from SD card to debug

# Check Dropbear config
cat /etc/dropbear-initramfs/authorized_keys

# Rebuild initramfs
sudo update-initramfs -u

# Check network config in initramfs
cat /etc/initramfs-tools/initramfs.conf
```

### Issue: Pods Stuck in Pending

**Symptoms**: Pods don't schedule

**Solution**:
```bash
# Check pod events
kubectl describe pod <pod-name> -n <namespace>

# Common causes:
# 1. Insufficient resources
kubectl top nodes

# 2. PVC not bound
kubectl get pvc -A

# 3. Node taints
kubectl get nodes -o json | jq '.items[].spec.taints'
```

### Issue: Longhorn Volume Not Attaching

**Symptoms**: Volume stuck in "Attaching" state

**Solution**:
```bash
# Check Longhorn manager logs
kubectl logs -n longhorn-system -l app=longhorn-manager

# Check iscsi service
sudo systemctl status iscsid

# Restart if needed
sudo systemctl restart iscsid

# Delete and recreate volume attachment
kubectl delete volumeattachment <attachment-name>
```

### Issue: High CPU/Memory Usage

**Symptoms**: Node becomes unresponsive

**Solution**:
```bash
# Check resource usage
kubectl top nodes
kubectl top pods -A

# Identify culprit
kubectl get pods -A --sort-by=.status.containerStatuses[0].restartCount

# Set resource limits on problematic pods
kubectl set resources deployment <name> \
  --limits=cpu=500m,memory=512Mi \
  --requests=cpu=250m,memory=256Mi
```

### Issue: Network Connectivity Between Pods

**Symptoms**: Pods cannot communicate with each other

**Solution**:
```bash
# Check CNI plugin status
kubectl get pods -n kube-system -l k8s-app=flannel

# Test network from a pod
kubectl run test-pod --image=busybox -it --rm -- sh
# Inside pod:
nslookup kubernetes.default
ping <other-pod-ip>

# Check network policies
kubectl get networkpolicies -A

# Restart flannel if needed
kubectl delete pods -n kube-system -l k8s-app=flannel
```

### Issue: Service Not Accessible via NodePort

**Symptoms**: Cannot access service on NodePort

**Solution**:
```bash
# Check service configuration
kubectl get svc <service-name> -n <namespace>

# Verify NodePort is open
sudo ufw status | grep <port>

# Test from node itself
curl http://localhost:<nodeport>

# Check endpoint
kubectl get endpoints <service-name> -n <namespace>
```

---

## Advanced Configurations

### Option A: Tang Server for Automatic LUKS Unlock

**Why**: Eliminates need for manual unlock during boot while maintaining security

**Setup**:

```bash
# Install Tang on a separate always-on device (or node1)
# On Tang server (e.g., your router or a separate device):
sudo apt install tang
sudo systemctl enable tangd.socket
sudo systemctl start tangd.socket

# Get Tang server thumbprint
tang-show-keys 7500

# On each Pi node:
sudo apt install clevis clevis-luks clevis-initramfs

# Bind LUKS to Tang server
sudo clevis luks bind -d /dev/nvme0n1p2 tang \
  '{"url":"http://192.168.1.9:7500","thp":"<thumbprint>"}'

# Verify binding
sudo clevis luks list -d /dev/nvme0n1p2

# Update initramfs
sudo update-initramfs -u

# Test: Reboot node
# It should auto-unlock when on trusted network
```

**Security Note**: Tang provides "network-bound" encryption. Drives auto-unlock only when connected to your trusted network.

### Option B: MetalLB for LoadBalancer Services

**Why**: Get real LoadBalancer IPs instead of NodePort

**Setup**:

```bash
# Install MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml

# Wait for pods
kubectl get pods -n metallb-system -w

# Create IP address pool
kubectl apply -f - <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.20-192.168.1.29
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
  - default-pool
EOF

# Test by creating a LoadBalancer service
kubectl expose deployment nginx --type=LoadBalancer --port=80
kubectl get svc nginx
# Should show EXTERNAL-IP from pool
```

### Option C: Cert-Manager for Automatic SSL

**Why**: Automatic SSL certificates from Let's Encrypt

**Setup**:

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.2/cert-manager.yaml

# Wait for pods
kubectl get pods -n cert-manager -w

# Create ClusterIssuer for Let's Encrypt
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: traefik
EOF

# Use in Ingress resources:
# annotations:
#   cert-manager.io/cluster-issuer: "letsencrypt-prod"
```

### Option D: Velero for Cluster Backups

**Why**: Complete cluster state and volume backups

**Setup**:

```bash
# Install Velero CLI
wget https://github.com/vmware-tanzu/velero/releases/download/v1.12.1/velero-v1.12.1-linux-arm64.tar.gz
tar -xvf velero-v1.12.1-linux-arm64.tar.gz
sudo mv velero-v1.12.1-linux-arm64/velero /usr/local/bin/

# Install Velero with Minio for storage
kubectl create namespace velero

# Install Minio (object storage)
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minio-pvc
  namespace: velero
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 50Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: velero
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
      - name: minio
        image: minio/minio:latest
        args:
        - server
        - /data
        - --console-address
        - ":9001"
        env:
        - name: MINIO_ROOT_USER
          value: "minio"
        - name: MINIO_ROOT_PASSWORD
          value: "minio123"
        ports:
        - containerPort: 9000
        - containerPort: 9001
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: minio-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: minio
  namespace: velero
spec:
  type: NodePort
  ports:
  - name: api
    port: 9000
    targetPort: 9000
    nodePort: 30900
  - name: console
    port: 9001
    targetPort: 9001
    nodePort: 30901
  selector:
    app: minio
EOF

# Create Minio bucket via UI (http://192.168.1.10:30901)
# Login: minio / minio123
# Create bucket: "velero"

# Install Velero
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket velero \
  --secret-file ./credentials-velero \
  --use-volume-snapshots=false \
  --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.velero.svc:9000

# Create credentials file:
cat > credentials-velero <<EOF
[default]
aws_access_key_id = minio
aws_secret_access_key = minio123
EOF

# Create backup schedule (daily at 2 AM)
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --ttl 720h0m0s

# Manual backup
velero backup create manual-backup-$(date +%Y%m%d)

# List backups
velero backup get

# Restore from backup
velero restore create --from-backup <backup-name>
```

---

## Performance Optimization

### CPU Governor Settings

```bash
# On all nodes - set performance mode
echo "performance" | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# Make persistent
sudo apt install -y cpufrequtils
echo 'GOVERNOR="performance"' | sudo tee /etc/default/cpufrequtils
sudo systemctl restart cpufrequtils
```

### NVMe Power Management

```bash
# Disable APST (Autonomous Power State Transition) for stability
echo "options nvme_core default_ps_max_latency_us=0" | \
  sudo tee /etc/modprobe.d/nvme.conf

# Reboot to apply
sudo reboot
```

### Memory Tuning for Kubernetes

```bash
# Reduce reserved memory for system
# Edit K3s service
sudo nano /etc/systemd/system/k3s.service

# Add to ExecStart:
--kube-reserved=cpu=100m,memory=512Mi \
--system-reserved=cpu=100m,memory=512Mi

# Reload and restart
sudo systemctl daemon-reload
sudo systemctl restart k3s
```

### Longhorn Performance Tuning

```bash
# Increase Longhorn engine replica timeout
kubectl -n longhorn-system patch settings.longhorn.io engine-replica-timeout \
  --type=json \
  -p='[{"op": "replace", "path": "/value", "value": "16"}]'

# Set concurrent replica rebuild limit
kubectl -n longhorn-system patch settings.longhorn.io concurrent-replica-rebuild-per-node-limit \
  --type=json \
  -p='[{"op": "replace", "path": "/value", "value": "2"}]'
```

---

## Monitoring Dashboards

### Import Grafana Dashboards

```bash
# Access Grafana: http://192.168.1.10:30000
# Login: admin / admin123

# Import these community dashboards by ID:
# - 15760: Kubernetes Cluster Monitoring
# - 13825: Longhorn Dashboard
# - 1860: Node Exporter Full
# - 15282: K3s Monitoring

# Go to: Dashboards -> Import -> Enter ID -> Load
```

### Setup Alerting

```bash
# Configure Alertmanager for email alerts
kubectl -n monitoring edit configmap prometheus-prometheus-kube-prometheus-alertmanager

# Add email configuration:
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'your-email@gmail.com'
  smtp_auth_username: 'your-email@gmail.com'
  smtp_auth_password: 'your-app-password'

receivers:
- name: 'email'
  email_configs:
  - to: 'your-email@gmail.com'
    send_resolved: true

route:
  receiver: 'email'
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
```

---

## Cost Summary

### Initial Investment

| Category | Items | Cost |
|----------|-------|------|
| **Compute** | 3x Raspberry Pi 5 (8GB) | $240 |
| **Storage** | 3x NVMe SSD (512GB) | $105 |
| **Power/Network** | 3x PoE+NVMe HAT | $120 |
| **Accessories** | Cases, cables | $45 |
| **Backup Power** | UPS (optional) | $180 |
| **Total (no UPS)** | | **$510** |
| **Total (with UPS)** | | **$690** |

### Operating Costs (Annual)

| Item | Calculation | Annual Cost |
|------|-------------|-------------|
| **Electricity** | 45W × 24h × 365d × $0.15/kWh | ~$59 |
| **Internet** | Already paid | $0 |
| **Replacement Budget** | 10% of hardware | ~$51 |
| **Total Annual** | | **~$110** |

**Per Month**: ~$9 operating cost

---

## Comparison with Alternatives

### vs. Cloud (AWS EKS)

| Metric | Pi Cluster | AWS EKS |
|--------|------------|---------|
| **Initial Cost** | $510 | $0 |
| **Monthly Cost** | $9 | $150+ |
| **Break-even** | 4 months | N/A |
| **Learning** | Full control | Abstracted |
| **Latency** | <1ms local | 20-50ms |

### vs. Single Server

| Metric | Pi Cluster | NUC/Mini PC |
|--------|------------|-------------|
| **Initial Cost** | $510 | $600-1200 |
| **Power Draw** | 45W | 65-150W |
| **Redundancy** | 3 nodes | Single point |
| **Upgradability** | Individual nodes | All or nothing |
| **K8s Learning** | True distributed | Simulated |

---

## Learning Resources

### Documentation

- **K3s**: https://docs.k3s.io/
- **Longhorn**: https://longhorn.io/docs/
- **Kubernetes**: https://kubernetes.io/docs/
- **Gitea**: https://docs.gitea.io/
- **Harbor**: https://goharbor.io/docs/

### Tutorials

- **Kubernetes Basics**: https://kubernetes.io/docs/tutorials/
- **Helm Charts**: https://helm.sh/docs/
- **GitOps with ArgoCD**: https://argo-cd.readthedocs.io/
- **CI/CD Pipelines**: https://tekton.dev/docs/

### Community

- **Reddit**: r/kubernetes, r/homelab, r/selfhosted
- **Discord**: Kubernetes, K3s official servers
- **GitHub**: Browse example configurations

---

## Future Expansion Ideas

### Add More Nodes

```bash
# Scale to 5 or 6 nodes for:
# - More redundancy
# - Higher workload capacity
# - Dedicated monitoring node

# Each additional node: ~$170
```

### Add GPU Node

```bash
# Raspberry Pi AI Kit (Hailo-8L)
# For ML/AI workloads
# ~$70 per accelerator
```

### Upgrade Storage

```bash
# Move to 1TB NVMe drives
# Additional cost: ~$20 per drive

# Or add USB SSD for cold storage
# Additional cost: ~$50 per drive
```

### Add-on Services

- **ArgoCD**: GitOps deployment
- **Tekton**: Cloud-native CI/CD
- **Keycloak**: Identity management
- **NextCloud**: Private cloud storage
- **Wiki.js**: Documentation platform
- **Uptime Kuma**: Monitoring dashboard
- **Bitwarden_rs**: Password manager

---

## Conclusion

This Raspberry Pi cluster provides:

✅ **Production-grade** Kubernetes experience  
✅ **Enterprise security** with full disk encryption  
✅ **High availability** across 3 nodes  
✅ **Cost-effective** at ~$9/month operating cost  
✅ **Educational** for learning DevOps/Cloud technologies  
✅ **Expandable** for future growth  
✅ **Power-efficient** at 45W total draw  

**Perfect for**:
- Learning Kubernetes and cloud-native technologies
- Running personal GitLab/Gitea instance
- Private container registry
- Development and testing environment
- Home automation and self-hosted services
- Building portfolio projects

**Next Steps**:
1. Order hardware components
2. Follow Phase 1-2 for first node
3. Verify encrypted boot and remote unlock
4. Replicate to remaining nodes
5. Deploy K3s and Longhorn
6. Start building your applications!

---

## Appendix A: Quick Command Reference

### Node Management
```bash
# Check node status
kubectl get nodes

# Drain node for maintenance
kubectl drain <node> --ignore-daemonsets

# Uncordon node
kubectl uncordon <node>

# Delete node
kubectl delete node <node>
```

### Pod Management
```bash
# Get pods in all namespaces
kubectl get pods -A

# Describe pod
kubectl describe pod <pod> -n <namespace>

# Get pod logs
kubectl logs <pod> -n <namespace>

# Execute command in pod
kubectl exec -it <pod> -n <namespace> -- /bin/bash
```

### Storage Management
```bash
# List PVCs
kubectl get pvc -A

# List PVs
kubectl get pv

# Describe PV
kubectl describe pv <pv-name>

# Delete PVC
kubectl delete pvc <pvc-name> -n <namespace>
```

### Service Management
```bash
# List services
kubectl get svc -A

# Expose deployment
kubectl expose deployment <name> --port=80 --type=NodePort

# Get service details
kubectl describe svc <service> -n <namespace>
```

### Troubleshooting
```bash
# Get cluster info
kubectl cluster-info

# Get events
kubectl get events -A --sort-by='.lastTimestamp'

# Top nodes/pods
kubectl top nodes
kubectl top pods -A

# K3s logs
sudo journalctl -u k3s -f
```

---

## Appendix B: Configuration File Templates

### Sample Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  namespace: default
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
  - host: example.cluster.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: example-service
            port:
              number: 80
```

### Sample Deployment with Resources

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: example
  template:
    metadata:
      labels:
        app: example
    spec:
      containers:
      - name: app
        image: nginx:alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

---

## Document Version

**Version**: 1.0  
**Last Updated**: October 2025  
**Author**: Pi Cluster Setup Guide  
**License**: MIT

For questions or issues, consult the troubleshooting section or reach out to the Kubernetes/Raspberry Pi communities.