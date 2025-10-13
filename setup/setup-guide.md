# Kubernetes Multi Master Cluster Setup With Loadbalancer - HA 

# 🚀 Kubernetes Cluster Setup Guide

This repository provisions a highly available Kubernetes cluster with:

- 🧠 Two control plane nodes using **stacked etcd**
- 🧱 Two worker nodes for running workloads
- 🔐 Secure communication and certificate management
- 🌐 Load balancer for control plane endpoint
- ☁️ AWS infrastructure with VPC and security groups

---

## 🗺️ Architecture for the multi master setup :  Stacked ETCD

<img width="2177" height="1161" alt="K8s-architecture-lb" src="https://github.com/user-attachments/assets/85597433-0135-43a2-89ae-6906e0e2f06d" />

---

## 🧩 Setup Workflow

### 1. ☁️ AWS VPC Provisioning
Create a dedicated VPC for the Kubernetes cluster

Define public and private subnets across availability zones

Set up route tables and internet gateway for public access

### 2. 🔐 Security Group Configuration
Provision three security groups:

Control Plane SG: Allows traffic on ports 6443 (Kubernetes API), 2379–2380 (etcd), and 10250 (kubelet)

Worker Node SG: Allows traffic on ports 10250 (kubelet), 30000–32767 (NodePort), and internal pod communication

Load Balancer SG: Allows inbound traffic on port 443/80 and forwards to control plane nodes on port 6443

### 3. 🔧 Node Preparation
Configure hostnames and networking

Disable swap and apply kernel settings

Load required modules for container networking

### 4. 🐳 Install Core Components
Container runtime (containerd)

runc binary

Kubernetes tools: kubeadm, kubelet, kubectl

CNI plugin (e.g., Calico)

### 5. 🧠 Initialize First Control Plane Node
Use kubeadm to initialize the cluster

Upload control plane certificates

Configure kubectl access

### 6. 🧠 Join Second Control Plane Node
Use the certificate key to securely join

If expired, regenerate using the control plane upload phase

### 7. 🧱 Join Worker Nodes
Generate a new join token from the control plane

Use the token and CA hash to join each worker node

### 8. 🔍 Verify Cluster Health
Confirm all nodes are in Ready state

Ensure control plane components and CNI pods are running

### 9. 🚀 Deploy a Test Pod
Provision a sample NGINX pod to verify workload scheduling

✅ Outcome
Your cluster is now:

Highly available with multiple control plane nodes

Backed by stacked etcd for embedded storage

Running on secure, scalable AWS infrastructure

Ready to deploy and scale workloads across worker nodes

Setup scripts and configuration files are located in the /setup directory.

