# Kubernetes Cluster Setup on Ubuntu 24.04 (Master &amp; Worker Nodes)

<div align="center">
    <img src="aidyf2n.png" alt="AIDY-F2N">
</div>

This guide explains how to set up a Kubernetes cluster on **Ubuntu 24.04 LTS**, using **containerd** as the container runtime and `kubeadm` for cluster bootstrapping.  
It includes the required system setup, network configuration, and role-specific commands for **Master** and **Worker** nodes.


## Commands for *ALL Nodes* (Master and Workers)

These commands must be run on **every node** (master and workers):

### 1. Kernel and System Configurations

To enable required networking and resource management:

```bash
sudo iptables -P FORWARD ACCEPT

sudo swapoff -a

sudo ufw disable #NOT RECOMMENDED in PROD

sudo modprobe overlay
sudo modprobe br_netfilter

sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF

sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

### 2. Install containerd
We use containerd as the container runtime, recommended by Kubernetes.


```bash
sudo apt update
sudo apt install -y curl ca-certificates apt-transport-https containerd

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```

### 3. Install Kubernetes Components
Add the official Kubernetes package repository:


```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" | \
sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm
sudo apt install -y kubectl  # ONLY FOR MASTER
```

## Master Node Setup
Run the following steps only on the master node.

### 1. Initialize the Cluster
We’ll use Flannel as the network plugin, so specify its CIDR:
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
Save the kubeadm join command shown at the end — you’ll need it for the workers.

### 2. Configure kubectl for Your User
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
### 3. Install Flannel CNI
This allows pods to communicate within the cluster:
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

### 4. (Optional) Allow Scheduling on Master
Useful for single-node clusters:
```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

## Worker Node Setup
Run these commands on each worker node to join the cluster, after installing containerd and kubeadm:

### 1. Join the Cluster
Use the kubeadm join command that was displayed on the master after initialization. It looks like:
```bash
# Join the cluster using the command provided by `kubeadm init`
sudo kubeadm join <MASTER_IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```
Replace <MASTER_IP>, <TOKEN>, and <HASH> with actual values.

<div align="center">
    <img src="join.png" alt="Join">
</div>

Test the cluster on the Matser node : 
```bash
kubectl get nodes

<div align="center">
    <img src="1_IconsAll_Hori.png" alt="AIDY-F2N">
</div>
```

```bash
kubectl get pods -A
```

<div align="center">
    <img src="1_IconsAll_Hori.png" alt="AIDY-F2N">
</div>

## (Optional) Visualize Your Cluster with Lens
Once your cluster is ready, use Lens — a powerful Kubernetes IDE — to manage it visually instaed of using the CLI by kubectl.

Steps:
1- Download Lens: https://k8slens.dev/download
2- Launch the app.
3- It will automatically detect your kubeconfig file (~/.kube/config).
4- Click “Add Cluster” and select your cluster.
5- You're ready to view workloads, logs, nodes, and much more.

<div align="center">
    <img src="lens.png" alt="Lens">
</div>



