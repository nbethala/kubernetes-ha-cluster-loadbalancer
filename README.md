# 🚀 Kubernetes HA Cluster with Load Balancer on AWS

This project demonstrates the deployment of a **Highly Available (HA) Kubernetes cluster** in the **AWS Cloud**, manually provisioned to help understand the architecture and operational flow of a production-grade setup.

---

## 📌 Project Overview

The goal of this project is to build a resilient Kubernetes control plane with multiple master nodes and a load balancer, ensuring high availability and fault tolerance. The cluster is manually configured to provide hands-on insight into each component and its role in the system.


## 🧱 Architecture Summary

The cluster consists of:

- **2 Control Plane Nodes**: Running Kubernetes master components and etcd (stacked topology)
- **2 Worker Nodes**: Hosting application workloads and pods
- **1 Load Balancer Node**: Running HAProxy to distribute traffic to control plane nodes
- **1 VPC**: Custom VPC with a single subnet for all nodes (simplified for learning)
- **3 Security Groups**:
  - `master-sg`: For control plane traffic
  - `worker-sg`: For pod and node communication
  - `load-balancer-sg`: For HAProxy access

All nodes run **Ubuntu 16.04+** and are provisioned with appropriate CPU, RAM, and storage based on their roles.

## 🗺️ Architecture for the multi master setup :  Stacked ETCD

<img width="2177" height="1161" alt="K8s-architecture-lb" src="https://github.com/user-attachments/assets/85597433-0135-43a2-89ae-6906e0e2f06d" />


## 🔧 Manual Implementation Highlights

- EC2 instances launched and configured manually
- SSH access via `.pem` key pair
- Kubernetes components installed using `kubeadm`
- HAProxy manually configured to route traffic to master nodes
- Network and firewall rules set via AWS Security Groups
- Troubleshooting performed using `journalctl`, `systemctl`, and `kubectl`

This manual setup provides deep visibility into each step of the cluster bootstrapping process.

## ⚙️ Next Phase: Terraform Automation

To improve scalability and repeatability, this infrastructure will be re-implemented using **Terraform**, enabling:

- Infrastructure-as-Code provisioning
- Modular and reusable components
- Easier teardown and re-deployment
- Version-controlled infrastructure changes

Terraform modules will cover:
- VPC and subnet creation
- Security group definitions
- EC2 instance provisioning
- Key pair management
- Output variables for IPs and access



## 📚 Learning Objectives

By working through this project, you will:

- Understand the components of a Kubernetes HA cluster
- Learn how to manually configure and troubleshoot each layer
- Gain experience with AWS networking and EC2 provisioning
- Prepare for infrastructure automation using Terraform

## Repository Structure: 

```
kubernetes-ha-cluster-aws/
├── README.md                    # Project overview and architecture
├── troubleshooting-guide.md     # Common errors and fixes
├── setup/
│   ├── setup-guide.md           # Full cluster setup walkthrough 
|   ├──infra-setup.md            # AWS infrastructure details
│   ├── master-setup.md          # Manual setup for control plane nodes
│   ├── worker-setup.md          # Manual setup for worker nodes
│   └── loadbalancer-setup.md    # Manual setup for HAProxy load balancer
```

## 📘 Full Setup Guide: [setup/setup-guide.md](setup/setup-guide.md)

> 💡 Tip: This project is ideal for DevOps engineers, cloud architects, and learners preparing for Kubernetes certifications.

