## 🛠️ Kubernetes System Prep (Linux) : Setup Leader Master Node

This section prepares your Linux system for Kubernetes installation by loading required kernel modules and configuring system networking parameters.

---
1. ###  SSH into the Master EC2 server
   Do sudo -i 

   Disable Swap using the below commands

  ```bash
  swapoff -a
  sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
   ```

2. ### 🔧 Load Required Kernel Modules

  #### Forwarding IPv4 and letting iptables see bridged traffic
Create a config file to load necessary modules at boot:

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
#### Notes
overlay: Required for container storage layers

br_netfilter: Enables iptables visibility for bridged traffic

ip_forward: Allows packet routing between interfaces (essential for pod networking)

Why This Matters
Without these steps, Kubernetes networking (like pod-to-pod communication and service routing) might fail silently. This setup ensures your system is ready to handle container traffic and Kubernetes networking rules.

3. ### Install container runtime
Manual Installation of containerd (v1.7.14)
This guide installs containerd manually from the official GitHub release and configures it for Kubernetes compatibility.

```bash
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
#### Notes
This method gives you full control over the containerd version.

Be sure to match containerd version with Kubernetes compatibility matrix.

SystemdCgroup = true is essential for kubelet to work properly.

4. ### Install runc (Container Runtime CLI)
runc is a low-level CLI tool used by containerd to spawn and manage containers. This step installs the latest stable release manually.

```bash
curl -LO https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
```
#### Notes
runc is required by containerd to manage containers at the OS level

Installing manually ensures you get the exact version needed for compatibility

5. ### Install CNI Plugins (Container Network Interface)
Kubernetes uses CNI plugins to manage pod networking. These plugins enable communication between pods across nodes.

```bash
curl -LO https://github.com/containernetworking/plugins/releases/download/v1.5.0/cni-plugins-linux-amd64-v1.5.0.tgz
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.5.0.tgz
```
#### Notes
These plugins are required for Kubernetes to set up pod networking.

Common plugins include bridge, host-local, loopback, and portmap.

You can later install advanced CNI solutions like Calico, Cilium, or Flannel for full network policies and overlays.

6. ### Install Kubernetes Components: kubeadm, kubelet, and kubectl
These tools are essential for setting up and managing your Kubernetes cluster:

kubeadm: Initializes and configures the cluster

kubelet: Runs on every node and manages containers

kubectl: CLI tool to interact with the cluster

```bash
# Step 1: Update system and install required packages
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# Step 2: Add Kubernetes apt repository
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Step 3: Update package list
sudo apt-get update

# Step 4: Install specific Kubernetes versions (adjust version if needed)
sudo apt-get install -y kubelet=1.33.0-1.1 kubeadm=1.33.0-1.1 kubectl=1.33.0-1.1 --allow-downgrades --allow-change-held-packages

# Step 5: Hold the package versions to prevent upgrades
sudo apt-mark hold kubelet kubeadm kubectl

# Step 6: Verify installation
kubeadm version
kubelet --version
kubectl version --client
```
### check kubelet in properly installed and enabled or not

```
systemctl status kubelet
```
7. ### Kubernetes Control Plane Initialization with kubeadm
```
kubeadm init \
  --control-plane-endpoint "<load-balancer-private-ip>:6443" \
  --upload-certs \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=<private-ip-of-this-ec2-instance>
```

8. ### TLS certificates 
Kubernetes generates a set of TLS certificates post kubeadm initialization that are essential for securing communication between cluster components. 

Why Are These Important?
Security: All communication between Kubernetes components (API server, kubelet, etcd, controller manager) is encrypted and authenticated using these certificates.

Trust: The ca.crt acts as the root of trust — all other certs are signed by it.

Access Control: Service accounts and kubelets use certs to prove identity and permissions.

9. ### 🛠️ Configure kubectl Access
Run these commands as your regular user:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Alternatively, if you are the root user, you can run:

```
export KUBECONFIG=/etc/kubernetes/admin.conf
```

10. ### Install calico (network addon ) to start pod to pod communication
```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml

curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml -O

kubectl apply -f custom-resources.yaml
```

### Final step to ensure pod communication is enabled : 
```
kubectl get nodes
NAME           STATUS   ROLES           AGE   VERSION
ip-10-0-1-37   Ready    control-plane   24m   v1.33.0
```


Feature	Provided by Calico
Pod IP assignment	✅ Yes

Pod-to-pod routing	✅ Yes

Cross-node communication	✅ Yes

Network policies	✅ Yes

Matches CIDR 192.168.0.0/16	✅ Yes


### control-plane node is in ready state





