# Kubernetes Course-End Project 7 – PHP (WordPress) + MySQL on K8s

Deploy a multi-tier WordPress + MySQL application on Kubernetes. This project demonstrates the integration of persistent storage, configuration management, security (RBAC/Secrets), and resource controls.

## 🚀 Project Overview

**Key Features:**
* **Namespace Isolation:** All resources deployed in `project7-ns`.
* **Storage:** NFS-backed PersistentVolumes and PersistentVolumeClaims.
* **Configuration:** Secrets for credentials and ConfigMaps for application settings.
* **Security:** RBAC with a dedicated ServiceAccount and ClusterRoleBinding for the Kubernetes Dashboard.
* **Resource Management:** ResourceQuotas and LimitRanges to control consumption.
* **Networking:** NodePort services to expose WordPress and the Dashboard.

---

## 📂 Repository Structure

```text
.
├── dashboard-admin-crb.yaml      # ClusterRoleBinding for Dashboard admin
├── dashboard-admin-sa.yaml       # ServiceAccount for Dashboard admin
├── mysql-deployment.yaml         # MySQL Deployment (uses Secret + PVC)
├── mysql-pv.yaml                 # MySQL PersistentVolume (NFS)
├── mysql-pvc.yaml                # MySQL PersistentVolumeClaim
├── mysql-secret.yaml             # Database credentials (Secret)
├── mysql-service.yaml            # MySQL ClusterIP Service
├── ns-limits.yaml                # LimitRange for namespace
├── ns-quota.yaml                 # ResourceQuota for namespace
├── project7-admin-crb.yaml       # (Duplicate of dashboard-admin-crb.yaml)
├── project7-admin-sa.yaml        # (Duplicate of dashboard-admin-sa.yaml)
├── wordpress-config.yaml         # WordPress ConfigMap (DB host & name)
├── wordpress-deployment.yaml     # WordPress Deployment
├── wordpress-pv.yaml             # WordPress PersistentVolume (NFS)
├── wordpress-pvc.yaml            # WordPress PersistentVolumeClaim
└── wordpress-service.yaml        # WordPress NodePort Service (port 30080)
```

> **Note:** The files `dashboard-admin-*.yaml` and `project7-admin-*.yaml` contain the same resources. They are duplicated to match course naming conventions. You can apply either pair.

---

## 🛠 Prerequisites

### 1. Kubernetes Cluster
* 1 Control-plane node + at least 1 Worker node.
* `kubectl` configured.
* CNI Plugin installed (Flannel/Calico).
* CoreDNS running.

### 2. NFS Server (On Manager Node)
This project uses NFS for persistence. Ensure the NFS server is set up on your manager node (e.g., `172.31.23.205`).

**On the Manager Node:**

```bash
sudo apt update
sudo apt install -y nfs-kernel-server

# Create directories
sudo mkdir -p /srv/nfs/mysql /srv/nfs/wordpress
sudo chown -R nobody:nogroup /srv/nfs
sudo chmod -R 0777 /srv/nfs

# Configure exports
echo '/srv/nfs/mysql      172.31.0.0/16(rw,sync,no_subtree_check,no_root_squash)' | sudo tee -a /etc/exports
echo '/srv/nfs/wordpress  172.31.0.0/16(rw,sync,no_subtree_check,no_root_squash)' | sudo tee -a /etc/exports

# Apply changes
sudo exportfs -rav
sudo systemctl restart nfs-kernel-server
```

**On the Worker Node(s):**

```bash
sudo apt update
sudo apt install -y nfs-common

# Verify visibility
showmount -e 172.31.23.205
```

> **Important:** If your manager IP is different from `172.31.23.205`, update the `server:` field in `mysql-pv.yaml` and `wordpress-pv.yaml` before deploying.

### 3. Kubernetes Dashboard
The official Kubernetes Dashboard should already be installed in the `kubernetes-dashboard` namespace. This repo adds the admin access configuration.

---

## 📦 Deployment Guide

Run all commands from the root of this repository.

### Step 1: Create Namespace

```bash
kubectl create namespace project7-ns
```

### Step 2: Apply Resource Quotas & Limits

Enforce CPU, memory, and storage constraints.

```bash
kubectl apply -f ns-quota.yaml    -n project7-ns
kubectl apply -f ns-limits.yaml   -n project7-ns

# Verify
kubectl get resourcequota,limitrange -n project7-ns
```

### Step 3: Create Persistent Volumes & Claims

```bash
# PVs are cluster-scoped
kubectl apply -f mysql-pv.yaml
kubectl apply -f wordpress-pv.yaml

# PVCs are namespaced
kubectl apply -f mysql-pvc.yaml      -n project7-ns
kubectl apply -f wordpress-pvc.yaml  -n project7-ns

# Verify Binding
kubectl get pv
kubectl get pvc -n project7-ns
```
*Status should be **Bound**.*

### Step 4: Secrets & ConfigMaps

```bash
kubectl apply -f mysql-secret.yaml     -n project7-ns
kubectl apply -f wordpress-config.yaml -n project7-ns
```

### Step 5: Deploy MySQL

```bash
kubectl apply -f mysql-deployment.yaml -n project7-ns
kubectl apply -f mysql-service.yaml    -n project7-ns
```

**Test Database Connectivity (Optional):**
```bash
kubectl run mysql-client --rm -it \
  --image=mysql:5.7 \
  --restart=Never \
  -n project7-ns -- bash

# Inside the pod:
mysql -h mysql-service -u wpuser -pwppass123 -e "SHOW DATABASES;"
```

### Step 6: Deploy WordPress

```bash
kubectl apply -f wordpress-deployment.yaml -n project7-ns
kubectl apply -f wordpress-service.yaml    -n project7-ns

# Verify everything is running
kubectl get deploy,svc,pods -n project7-ns
```

---

## 🌐 Accessing the Application

### WordPress
The application is exposed via NodePort `30080`.

**URL:** `http://<worker-public-ip>:30080/`

**Verification:**
1.  Complete the installation wizard.
2.  Log in to `/wp-admin/`.
3.  Create a test post (e.g., "NFS Persistence Test").
4.  **Test Persistence:** Delete the WordPress pod to ensure data survives.
    ```bash
    kubectl delete pod -l app=wordpress -n project7-ns
    ```
5.  Wait for the new pod to start and refresh the browser. The post should still be there.

---

## 🔐 Kubernetes Dashboard Access (RBAC)

### 1. Create Admin ServiceAccount
```bash
kubectl apply -f dashboard-admin-sa.yaml
kubectl apply -f dashboard-admin-crb.yaml
```

### 2. Retrieve Login Token
```bash
kubectl create token project7-admin-sa -n kubernetes-dashboard
```
*Copy the generated token.*

### 3. Login
**URL:** `https://<worker-public-ip>:30443/`

* Select **Token** authentication.
* Paste the token copied above.
* You now have full `cluster-admin` access via the GUI.

---

## ✅ Verification Checklist

- [ ] Dedicated namespace `project7-ns` created.
- [ ] ResourceQuota & LimitRange active.
- [ ] NFS PVs and PVCs bound.
- [ ] Secrets and ConfigMaps created.
- [ ] MySQL running (ClusterIP).
- [ ] WordPress running (NodePort 30080).
- [ ] **Persistence confirmed** (Data survives pod deletion).
- [ ] Dashboard accessible via NodePort 30443 with Admin Token.

---

## 🧹 Cleanup

To remove all resources created by this project:

```bash
# Delete Project Resources
kubectl delete -n project7-ns \
  -f wordpress-service.yaml \
  -f wordpress-deployment.yaml \
  -f mysql-service.yaml \
  -f mysql-deployment.yaml \
  -f wordpress-pvc.yaml \
  -f mysql-pvc.yaml \
  -f wordpress-config.yaml \
  -f mysql-secret.yaml \
  -f ns-limits.yaml \
  -f ns-quota.yaml

# Delete PVs and Namespace
kubectl delete pv wordpress-pv mysql-pv
kubectl delete namespace project7-ns

# Delete Dashboard Admin Access
kubectl delete -f dashboard-admin-crb.yaml
kubectl delete -f dashboard-admin-sa.yaml
```

> **Note:** To delete the actual data, manually remove `/srv/nfs/mysql` and `/srv/nfs/wordpress` on the NFS server.