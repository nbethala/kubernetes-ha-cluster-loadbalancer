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
