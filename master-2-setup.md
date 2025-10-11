#  Adding Control Plane Nodes to Kubernetes Cluster

This guide explains how to join additional EC2 instances as control plane nodes to an existing Kubernetes cluster using `kubeadm`.

---

##  Prerequisites

Before joining a new control plane node:

- EC2 instance must be in the **same VPC** and **subnet** as the existing cluster
- Kubernetes components (`kubeadm`, `kubelet`, `kubectl`) must be installed
- Container runtime (e.g., containerd) must be configured
- Security groups must allow:
  - TCP 6443 from HAProxy
  - TCP 2379–2380 for etcd peer communication
  - TCP 10250, 10257, 10259 for control plane components

---
##  Kernel Preparation on Master Node

Before initializing the Kubernetes control plane, prepare the EC2 instance by disabling swap and adjusting kernel settings.


###  Step 1: SSH into the Master EC2 Instance

```bash
ssh -i <your-key.pem> ec2-user@<master-node-public-ip>
```

### Step 2: Switch to Root User

```
sudo -i
```

### Step 3: Disable Swap
Kubernetes requires swap to be disabled for proper scheduling and memory management.

```
swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### Step 4: Enable IPv4 Forwarding and Bridged Traffic Visibility

Kubernetes requires certain kernel parameters to be set so that networking works correctly, especially for container traffic.

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system

# Verify that the br_netfilter, overlay modules are loaded by running the following commands:
lsmod | grep br_netfilter
lsmod | grep overlay

# Verify that the net.bridge.bridge-nf-call-iptables, net.bridge.bridge-nf-call-ip6tables, and net.ipv4.ip_forward system variables are set to 1 in your sysctl config by running the following command:
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

###  Step 5: Install Container Runtime

Kubernetes requires a container runtime to manage and run containers. In this setup, we use **containerd**, a lightweight and widely supported runtime.

```
curl -LO https://github.com/containerd/containerd/releases/download/v1.7.14/containerd-1.7.14-linux-amd64.tar.gz
sudo tar Cxzvf /usr/local containerd-1.7.14-linux-amd64.tar.gz
curl -LO https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
sudo mkdir -p /usr/local/lib/systemd/system/
sudo mv containerd.service /usr/local/lib/systemd/system/
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
sudo systemctl daemon-reload
sudo systemctl enable --now containerd

# Check that containerd service is up and running
systemctl status containerd
```


### Step 6: Install runc

`runc` is a lightweight CLI tool used to spawn and run containers according to the Open Container Initiative (OCI) specification. It is a required low-level runtime for containerd.

```
curl -LO https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
```


### Step 7: Install CNI Plugin (Calico)

Kubernetes requires a Container Network Interface (CNI) plugin to enable pod-to-pod communication across nodes. In this setup, we use **Calico**, a widely adopted CNI that supports network policies and scales well in cloud environments.

```
curl -LO https://github.com/containernetworking/plugins/releases/download/v1.5.0/cni-plugins-linux-amd64-v1.5.0.tgz
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.5.0.tgz
```

### Step 8: Install Kubernetes Utilities

Install the core Kubernetes tools required to initialize and manage the cluster:

- `kubeadm`: Bootstraps the cluster
- `kubelet`: Runs on all nodes and manages containers
- `kubectl`: CLI to interact with the cluster

###  Install kubeadm, kubelet, and kubectl

```
# Update system and install required packages
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# Add Kubernetes apt repository
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update package list
sudo apt-get update

# Install specific Kubernetes versions (adjust version if needed)
sudo apt-get install -y kubelet=1.33.0-1.1 kubeadm=1.33.0-1.1 kubectl=1.33.0-1.1 --allow-downgrades --allow-change-held-packages

# Hold the package versions to prevent upgrades
sudo apt-mark hold kubelet kubeadm kubectl

# Verify installation
kubeadm version
kubelet --version
kubectl version --client
```

### Step 9: Join Additional Control Plane Node

To create a highly available Kubernetes cluster, join this EC2 instance to the existing leader control plane using `kubeadm`.

```
kubeadm join <load-balancer-ip>:6443 \
  --token <your-token> \
  --discovery-token-ca-cert-hash sha256:<your-ca-hash> \
  --control-plane \
  --certificate-key <your-certificate-key>
```
Replace load-balancer-ip, your-token, your-ca-hash, and your-certificate-key with actual values from your setup from Master node 1/leader master.

#### ⚠️ Expiration Warning

The `kubeadm-certs` secret **expires after 2 hours** by default. If you attempt to join a control plane node after this window, you'll encounter an error like:

error downloading the secret: Secret "kubeadm-certs" was not found in the "kube-system" Namespace.

####  How to Re-Initiate the Certificate Key

To regenerate the certificate key and re-upload the secret:

1. SSH into an existing control plane node (e.g., Master-1)
2. Run the following command:

```bash
kubeadm init phase upload-certs --upload-certs
```
Use this new key in your kubeadm join command on the new control plane node.

#### Join Confirmation Output

This node has joined the cluster and a new control plane instance was created:

Certificate signing request was sent to apiserver and approval was received.

The Kubelet was informed of the new secure connection details.

Control plane label and taint were applied to the new node.

The Kubernetes control plane instances scaled up.

A new etcd member was added to the local/stacked etcd cluster.

#### What This Means

- 🔐 **Secure Communication Established**: The node received signed certificates from the API server.
- ⚙️ **Kubelet Configured**: The node is now managing containers and communicating with the cluster.
- 🧭 **Control Plane Role Assigned**: It’s labeled and tainted to run control plane components only.
- 📈 **Cluster Scaled**: Your control plane is now multi-node and highly available.
- 🗃️ **etcd Expanded**: The node is now part of the distributed etcd database.

To start administering your cluster from this node, you need to run the following as a regular user:

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### Verify the cluster status:

```bash
kubectl get nodes
kubectl get pods -n kube-system -o wide

NAME            STATUS   ROLES           AGE   VERSION
ip-10-0-1-37    Ready    control-plane   19h   v1.33.0
ip-10-0-11-41   Ready    control-plane   13m   v1.33.0

```

