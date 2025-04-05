# 🚀 Kubernetes Installation with Docker Runtime (`cri-dockerd`)

This guide provides a single-script solution to set up Kubernetes v1.29 with Docker using `cri-dockerd` on Ubuntu.

---

## 📜 Script Overview

The script:

- Disables swap
- Configures kernel modules & sysctl
- Installs Docker and `cri-dockerd`
- Sets up Kubernetes packages (`kubelet`, `kubeadm`, `kubectl`)

---

## 🛠️ Instructions

### 1. Save the Script

Create a file named `setup-k8s-docker.sh`:

```bash
#!/bin/bash

set -e
set -o pipefail

echo "[Step 1] Updating system..."
sudo apt-get update
sudo apt-get upgrade -y

echo "[Step 2] Disabling swap..."
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

echo "[Step 3] Setting up kernel modules..."
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF
sudo modprobe br_netfilter

echo "[Step 4] Setting sysctl params..."
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system

echo "[Step 5] Installing Docker..."
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    apt-transport-https \
    software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/docker.gpg

sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io

echo "[Step 6] Enabling Docker..."
sudo systemctl enable docker
sudo systemctl start docker

echo "[Step 7] Installing cri-dockerd for Kubernetes compatibility..."
sudo apt-get install -y git golang-go

# Clone cri-dockerd
git clone https://github.com/Mirantis/cri-dockerd.git
cd cri-dockerd
mkdir bin
go build -o bin/cri-dockerd

# Install cri-dockerd binary
sudo install -o root -g root -m 0755 bin/cri-dockerd /usr/bin/cri-dockerd

# Set up systemd service
sudo cp -a packaging/systemd/* /etc/systemd/system/
sudo sed -i 's:/usr/bin/cri-dockerd:/usr/bin/cri-dockerd:' /etc/systemd/system/cri-docker.service
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable cri-docker.service
sudo systemctl enable --now cri-docker.socket

cd ..
rm -rf cri-dockerd

echo "[Step 8] Adding Kubernetes APT repo..."
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

echo "✅ Docker and Kubernetes installation complete. To init the cluster, run:"
echo "   kubeadm init --cri-socket unix:///var/run/cri-dockerd.sock"
