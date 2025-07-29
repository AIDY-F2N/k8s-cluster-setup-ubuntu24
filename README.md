# Kubernetes Cluster Setup on Ubuntu 24.04 (Master &amp; Worker Nodes)

This guide explains how to set up a Kubernetes cluster on **Ubuntu 24.04 LTS**, using **containerd** as the container runtime and `kubeadm` for cluster bootstrapping.  
It includes the required system setup, network configuration, and role-specific commands for **Master** and **Worker** nodes.


## Commands for *ALL Nodes* (Master and Workers)

These commands must be run on **every node** (master and workers):

```bash
# Allow IP forwarding for Kubernetes networking
sudo iptables -P FORWARD ACCEPT

# Disable swap (Kubernetes will not work with swap enabled)
sudo swapoff -a

# Load required kernel modules
sudo modprobe overlay
sudo modprobe br_netfilter

# Persist modules after reboot
sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF

# Set sysctl parameters required by Kubernetes
sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

# Apply sysctl settings
sudo sysctl --system

# Update system packages
sudo apt update

# Install required packages
sudo apt install -y curl ca-certificates apt-transport-https

# Install containerd
sudo apt install -y containerd

# Generate default config
sudo mkdir -p /etc/containerd
sudo sh -c "containerd config default > /etc/containerd/config.toml"

# Use systemd as the cgroup driver
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Restart and enable containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd

# Add Kubernetes repository
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update apt again and install Kubernetes components
sudo apt update
sudo apt install -y kubelet kubeadm

# (Optional for master or CLI usage)
sudo apt install -y kubectl
```

## Master Node Setup

```bash
# Initialize the cluster
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

#After running kubeadm init, it will print a kubeadm join command — copy this, you'll use it for worker nodes.



# Set up kubeconfig for kubectl usage
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

#Install a CNI (Container Network Interface) plugin (here we use Flannel):
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

#(Optional) If you're using a single-node cluster or want to schedule workloads on the master:
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

## Worker Node Setup
Run this only on worker nodes, after installing containerd and kubeadm:

```bash
# Join the cluster using the command provided by `kubeadm init`
sudo kubeadm join <MASTER_IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```


