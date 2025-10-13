# ☁️ AWS Infrastructure Setup

This section outlines the foundational AWS infrastructure required to deploy a Kubernetes cluster with high availability potential.

---

## 🌐 VPC Configuration

A single Virtual Private Cloud (VPC) is provisioned to host all cluster components.

### VPC Details

- **VPC CIDR Block**: `10.0.0.0/16`
- **Subnets**:
  - One subnet used for all nodes (control plane, worker, and load balancer)
  - Subnet is configured with internet access via an Internet Gateway
  - Note: For production-grade high availability, multiple subnets across availability zones are recommended



## 🔐 Security Groups

Three security groups are created to manage traffic between cluster components:

### 1. Control Plane Security Group (`master-sg`)

Used by EC2 instances running Kubernetes control plane components.

**Inbound Rules**:
- TCP 6443: Kubernetes API server
- TCP 2379–2380: etcd server client API
- TCP 10250: Kubelet API
- TCP 22: SSH access (optional)

**Outbound Rules**:
- Allow all traffic



### 2. Worker Node Security Group (`worker-sg`)

Used by EC2 instances running Kubernetes worker nodes.

**Inbound Rules**:
- TCP 10250: Kubelet API
- TCP 30000–32767: NodePort services
- TCP 22: SSH access (optional)
- Internal traffic from control plane SG

**Outbound Rules**:
- Allow all traffic



### 3. Load Balancer Security Group (`load-balancer-sg`)

Used by the HAProxy EC2 instance acting as the control plane load balancer.

**Inbound Rules**:
- TCP 80/443: Public access (optional)
- TCP 6443: Forward to control plane nodes

**Outbound Rules**:
- TCP 6443: To control plane SG
- Allow all traffic



## 🖥️ EC2 Instance Configuration

The Kubernetes cluster is deployed across five EC2 instances, each running Ubuntu 16.04 or later.

### 🧠 Control Plane Nodes (2 Machines)

- **Instance Type**: `t2.medium`
- **Operating System**: Ubuntu 16.04+
- **vCPU**: 2
- **Memory**: 2 GB RAM
- **Storage**: 10 GB
- **Security Group**: `master-sg`



### 🧱 Worker Nodes (2 Machines)

- **Instance Type**: `t2.small`
- **Operating System**: Ubuntu 16.04+
- **vCPU**: 1
- **Memory**: 2 GB RAM
- **Storage**: 10 GB
- **Security Group**: `worker-sg`



### 🌐 Load Balancer Node (1 Machine)

- **Instance Type**: `t2.small`
- **Operating System**: Ubuntu 16.04+
- **vCPU**: 1
- **Memory**: 2 GB RAM
- **Storage**: 10 GB
- **Security Group**: `load-balancer-sg`

---

## 🔑 Key Pair Setup

To securely access EC2 instances via SSH, a key pair must be created and associated with each instance during launch.

---

### 1. Create a Key Pair in AWS

- Go to the **EC2 Dashboard** in the AWS Console.
- In the left sidebar, click **Key Pairs** under **Network & Security**.
- Click **Create Key Pair**.
- Enter a name (e.g., `k8s-demo-key`) and choose **RSA** as the key type.
- Select **.pem** as the file format and click **Create**.
- The `.pem` file will be downloaded automatically — store it securely.



### 2. Use the Key Pair When Launching Instances

- When launching EC2 instances (control plane, worker, or load balancer), select the key pair you created under **Key pair (login)**.
- This allows SSH access using the corresponding private key.



### 3. Set Proper Permissions Locally

Before using the key to connect from your local machine:

```bash
chmod 400 k8s-demo-key.pem
```

> 💡 Tip: Ensure all instances are launched in the same VPC and subnet to maintain connectivity across the cluster.
