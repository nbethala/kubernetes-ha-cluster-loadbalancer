## 🛠️ Troubleshooting Guide

This section documents common issues encountered during the Kubernetes cluster setup and how to resolve them.

---

### ❌ Error 1: Private Key File — Permission Denied

**Cause**:  
This error occurs when your SSH private key file (e.g., `k8s-demo.pem`) has overly permissive file permissions. SSH refuses to use it for security reasons.

**Why This Key Is Needed**:  
The `.pem` private key file is required to securely connect from your **local machine** to an **EC2 instance** running inside your AWS VPC. It matches the public key embedded in the EC2 instance during launch, allowing encrypted access via SSH.

This is essential for:
- 🔐 Secure remote access to AWS nodes
- 🛠️ Running setup scripts from your local terminal
- 📦 Transferring files or debugging cluster components

**Solution**:  
Restrict access to the key file using:

```bash
chmod 400 k8s-demo.pem
```

This makes the file readable only by you, which is required by SSH.

### ❌ Error 2: HAProxy Restart — No Backend Available
Message:

haproxy[4836]: backend kube-master-nodes has no server available!
Cause: This occurs after restarting HAProxy before the worker nodes (EC2 instances) are configured.

Solution: Ignore this warning during initial setup. It resolves once worker nodes are provisioned and reachable.

### ❌ Error 3: Kubelet Status — Pod Inactive
Command:

```
systemctl status kubelet
```

Message:

Pod Inactive

Cause: The control plane has not been initialized yet.

Solution: Run kubeadm init on the master node, then check logs with:

```
journalctl -u kubelet -f
```

### ❌ Error 4: kubeadm init Fails — Port/File/Directory Conflicts
Message:

[ERROR Port-6443]: Port 6443 is in use
[ERROR FileAvailable--etc-kubernetes-manifests-kube-apiserver.yaml]: already exists
[ERROR DirAvailable--var-lib-etcd]: /var/lib/etcd is not empty
Cause: A previous or partial Kubernetes setup left behind configuration files and processes.

Solution: Perform a fast cleanup and re-init:

✅ Fast Cleanup Steps
Stop Kubernetes-related services

Run kubeadm reset

Remove leftover files from /etc/kubernetes/ and /var/lib/etcd

Flush iptables

Kill any stale API server processes

Re-run kubeadm init

### Error 5 : ⚠️ Kubernetes HA Control Plane Setup with HAProxy
❌ Issue: API Server Unreachable via HAProxy
Symptoms:

```
curl https://<haproxy-ip>:6443/livez → timeout
kubectl get nodes → i/o timeout
```

Root Cause: HAProxy EC2 instance was launched in the default VPC, while master nodes were in a custom VPC. VPCs are isolated by default, so there was no network connectivity.

🔍 Troubleshooting Steps
 - Verified HAProxy health checks were failing

 - Tested direct connectivity using curl and nc — all timed out

 - Reviewed NACLs and security groups — confirmed open

 - Discovered VPC mismatch

 - Recreated HAProxy EC2 instance in the correct VPC

 - Retested connectivity — success!

✅ Resolution
Once HAProxy was placed in the same VPC as the master nodes, health checks passed and kubeadm init succeeded using the load-balanced endpoint:

bash
kubeadm init \
  --control-plane-endpoint "<haproxy-private-ip>:6443" \
  --upload-certs \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=<master-node-ip>
