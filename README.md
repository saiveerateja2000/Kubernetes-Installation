# Kubernetes-Installation Guide

## Prerequisites
Ensure you have the following before proceeding:
- **Ubuntu 22.04 LTS**
- **Familiarity with Linux, CLI, and Kubernetes concepts**
- **sudo privileges on your machines**

---
## i used Containerd as the container runtime

## Setting up the Master Node
The master node is responsible for managing the cluster.

### Step 1: Update and Upgrade Ubuntu
Update the package lists and upgrade existing packages.
```sh
sudo apt-get update
sudo apt-get upgrade -y
```

### Step 2: Disable Swap
Kubernetes requires swap to be disabled.
```sh
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### Step 3: Add Kernel Parameters
Load required kernel modules and configure system settings.
```sh
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```

Configure sysctl parameters.
```sh
sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```
Reload the changes:
```sh
sudo sysctl --system
```

### Step 4: Install Containerd Runtime
Install necessary dependencies and set up the container runtime.
```sh
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/docker.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

Install Containerd:
```sh
sudo apt update
sudo apt install -y containerd.io
```

Configure Containerd:
```sh
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
```

Restart and enable Containerd:
```sh
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### Step 5: Install Kubernetes Components
Add the Kubernetes repository and install necessary components.
```sh
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Add the Kubernetes repository:
```sh
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Update and install Kubernetes components:
```sh
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Step 6: Initialize the Kubernetes Master Node
Initialize the Kubernetes cluster using `kubeadm`:
```sh
sudo kubeadm init
```

Set up `kubectl` for the current user:
```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Step 7: Deploy a Pod Network
A pod network is required for communication between nodes. Install Calico:
```sh
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
```

---

## Setting up Worker Nodes
Worker nodes run workloads and applications.

To join worker nodes to the cluster, execute the join command displayed after initializing the master node.

On each worker node, run:
```sh
sudo kubeadm join <master-ip>:6443 --token <token> \
 --discovery-token-ca-cert-hash sha256:<hash>
```

To retrieve the join command again, run on the master node:
```sh
kubeadm token create --print-join-command
```

---

## Verifying the Installation
Ensure the nodes are correctly added to the cluster:
```sh
kubectl get nodes
```

If all nodes show `Ready`, the Kubernetes cluster is successfully set up.

---

### Next Steps
- Deploy applications on the Kubernetes cluster.
- Configure persistent storage and ingress controllers.
- Set up monitoring with Prometheus and Grafana.

## (Optional)
**Running containerd Without Root Privileges**

### **Steps to Configure containerd for Non-Root Users**

1. **Check if containerd is running**  
   ```bash
   ps aux | grep -E 'containerd|dockerd|crio'
   ```
   This checks if `containerd`, `dockerd`, or `crio` is running.

2. **Set up `crictl` to use containerd**  
   ```bash
   sudo crictl config runtime-endpoint unix:///run/containerd/containerd.sock
   ```
   This sets the container runtime endpoint for `crictl`.

3. **Persist the `crictl` configuration**  
   ```bash
   echo "runtime-endpoint: unix:///run/containerd/containerd.sock" | sudo tee /etc/crictl.yaml
   ```
   Saves the runtime endpoint configuration permanently.

4. **Check permissions on the containerd socket**  
   ```bash
   ls -l /run/containerd/containerd.sock
   stat -c "%G" /run/containerd/containerd.sock
   ```
   These commands check the ownership and permissions of the containerd socket.

5. **Add the user to the `containerd` group**  
   ```bash
   sudo usermod -aG containerd $USER
   getent group containerd
   ```
   This ensures that your user is part of the `containerd` group. If the user is not there follow go to the next stage and create a user and add.

6. **Create the `containerd` group (if it doesn’t exist)**  
   ```bash
   sudo groupadd containerd
   sudo usermod -aG containerd $USER
   ```
   Creates the group and adds your user to it.

7. **Update socket ownership and permissions**  
   ```bash
   sudo chown root:containerd /run/containerd/containerd.sock
   sudo chmod 660 /run/containerd/containerd.sock
   ```
   Changes ownership to the `containerd` group and allows read/write access.

8. **Apply group changes without logging out**  
   ```bash
   newgrp containerd
   ```
   Applies the group changes immediately.

9. **Check the status of `containerd`**  
   ```bash
   systemctl status containerd
   ```
   Ensures that `containerd` is running properly.

### **Outcome**
- Normally, `containerd` requires root privileges to interact with its Unix socket.
- These steps allow non-root users to run container workloads by granting group-based access.
- After completing these steps, you should be able to use `crictl` or other container runtimes (`nerdctl`, `ctr`) without `sudo`.

This setup improves security by reducing the need for root access while managing containerized workloads.



This guide provides a minimal setup; for production environments, additional configurations may be necessary.
