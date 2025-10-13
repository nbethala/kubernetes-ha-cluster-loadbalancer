## Set Up Worker Nodes

Worker nodes run your application workloads and connect to the control plane for orchestration. Here's how to add them to your cluster.

---

###  Prerequisites

Ensure each worker node has:

- Kubernetes components installed (`kubeadm`, `kubelet`, `kubectl`)
- Container runtime configured (e.g., containerd)
- Swap disabled
- IPv4 forwarding and bridged traffic enabled
- CNI plugin installed (automatically applied after joining)

---

###  Worker Node Setup: Kernel Configuration

Before joining a worker node to the Kubernetes cluster, ensure the Linux kernel is properly configured to support container networking and routing.

---

###  Step 1: Disable Swap

Kubernetes requires swap to be disabled:

```bash
sudo -i
swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### Step 2: Enable IPv4 Forwarding and Bridged Traffic

```
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

### step 3: Install Container Runtime

Kubernetes requires a container runtime to manage and run containers. We'll use **containerd**, a lightweight and widely supported runtime.

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

### step 4: Install runc

`runc` is a lightweight CLI tool used by container runtimes like containerd to spawn and run containers according to the OCI (Open Container Initiative) specification.

```
curl -LO https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
```

### step 5: Install CNI Plugin


The Container Network Interface (CNI) plugin enables pod-to-pod communication across nodes. While the control plane usually applies the CNI configuration cluster-wide, it's important to ensure your worker node is ready to support it.

```
curl -LO https://github.com/containernetworking/plugins/releases/download/v1.5.0/cni-plugins-linux-amd64-v1.5.0.tgz
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.5.0.tgz
```

### step 6: Install Kubernetes Utilities

Install the core Kubernetes tools required for joining and managing the cluster:

- `kubeadm`: Used to join the cluster
- `kubelet`: Runs on all nodes and manages containers
- `kubectl`: CLI to interact with the cluster (optional on worker nodes)

```
# Step 6.1: Update system and install required packages
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# Step 6.2: Add Kubernetes apt repository
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Step 6.3: Update package list
sudo apt-get update

# Step 6.4: Install specific Kubernetes versions (adjust version if needed)
sudo apt-get install -y kubelet=1.33.0-1.1 kubeadm=1.33.0-1.1 kubectl=1.33.0-1.1 --allow-downgrades --allow-change-held-packages

# Step 6.5: Hold the package versions to prevent upgrades
sudo apt-mark hold kubelet kubeadm kubectl

# Step 6.6: Verify installation
kubeadm version
kubelet --version
kubectl version --client

```

### Step 7: Join Worker Nodes to the Control Plane

Worker nodes must be joined to the control plane to participate in the Kubernetes cluster and run workloads.

#### Recreate Worker Node Join Token 

If your original worker node token has expired (typically after 24 hours), you can easily generate a new one from any control plane node.

Run the following command on any control plane node:

```bash
kubeadm token create --print-join-command
```

This command will:

✅ Create a new bootstrap token (valid for 24 hours by default)

📋 Print the full kubeadm join command for worker nodes

Run Kubectl get nodes 

```
kubectl get nodes' on the control-plane to see this node join the cluster.
```














