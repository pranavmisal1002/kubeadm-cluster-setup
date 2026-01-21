# kubeadm-cluster-setup
## ✅ Architecture (What We Will Create)

| Node        | EC2 Instance | Purpose |
|------------|--------------|---------|
| Master Node | EC2-1         | Control Plane (API Server, etcd, scheduler) |
| Worker Node | EC2-2         | Runs pods/containers |

## 1) ✅ AWS Prerequisites (IMPORTANT)

### ✅ 1.1 Create 2 EC2 Instances

Create **two Ubuntu 22.04 EC2 instances** (recommended).

- **AMI:** Ubuntu 22.04
- **Instances Required:** 2
- **Instance Type:** `t2.medium` or `t3.medium` *(minimum recommended)*
- ✅ Kubernetes requires at least **2 CPU + 2GB RAM**, so `t2.medium` is safe.
- **Storage:** 20 GB

### ✅ 1.2 Security Group Rules

Attach the **same Security Group** to both EC2 instances *(recommended for beginners)*, or use separate SGs but ensure the following inbound rules are allowed.

#### Inbound Rules

| Type                  | Port           | Source |
|-----------------------|----------------|--------|
| SSH                   | 22             | Your IP |
| Kubernetes API        | 6443           | Worker SG / VPC CIDR |
| NodePort Services     | 30000-32767     | Anywhere *(optional)* |
| Flannel/Calico overlay| All traffic     | Same SG / VPC CIDR |

✅ **Easiest for learning:**  
- Allow **All TCP traffic** within the **same Security Group** (SG-to-SG).

### ✅ 1.3 Set Hostname (Recommended)

Set hostnames to easily identify nodes:

- **Master Node:** `k8s-master`
- **Worker Node:** `k8s-worker`
## 2) ✅ Connect to Both Instances

Login to **both** EC2 instances (Master and Worker) using SSH.
## 3) ✅ Steps to Run on BOTH Master + Worker

Perform the following steps on **both nodes** (Master and Worker).
### Step 1: Update System

Update and upgrade packages on both nodes:

```bash
sudo apt update -y
sudo apt upgrade -y
```
### Step 2: Disable Swap (Kubernetes requires swap OFF)

Disable swap on both nodes:

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
Verify swap is disabled:
```bash
free -h
```
### Step 3: Load Kernel Modules

Load required kernel modules for Kubernetes networking:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```
### Step 4: Enable `sysctl` Settings

Configure required networking settings for Kubernetes:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```
### Step 5: Install Container Runtime (containerd)

Install `containerd` on both nodes:

```bash
sudo apt install -y containerd
```
Create containerd configuration:

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```
Restart and enable containerd:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```
Verify containerd status:

```bash
sudo systemctl status containerd
```
### Step 6: Install kubeadm, kubelet, kubectl

Install required dependencies:

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```
Add Kubernetes repository:

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
Install Kubernetes packages:

```bash
sudo apt update -y
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
Enable kubelet service:

```bash
sudo systemctl enable kubelet
```

## 4) ✅ Setup MASTER Node (Control Plane)

Run the following steps **only on the Master node**.

### Step 1: Initialize Cluster

Initialize the Kubernetes cluster on the Master node:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
After initialization, kubeadm will output a JOIN command.
### Step 2: Setup kubectl for Ubuntu User

Configure `kubectl` access for the current user:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Test the configuration:

```bash
kubectl get nodes
```
### Step 3: Install Network Plugin (Flannel)

Install Flannel CNI on the Master node:

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```
Check Flannel and system pods:

```bash
kubectl get pods -n kube-flannel
kubectl get pods -n kube-system
```
## 5) ✅ Setup WORKER Node

Go to the **Worker node** and run the **JOIN command** copied from the Master node.

Example:

```bash
sudo kubeadm join <MASTER_PRIVATE_IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```
## 6) ✅ Verify Cluster

Run the following command on the **Master node**:

```bash
kubectl get nodes
```
