# Kubernetes Course-End Project 7 – PHP (WordPress) + MySQL on AWS

This repository contains the Kubernetes manifests used to deploy a multi-tier PHP
(WordPress) and MySQL application, using:

- A dedicated namespace with **ResourceQuota** and **LimitRange**
- **NFS-backed PersistentVolumes/PVCs** for MySQL and WordPress
- A **Kubernetes Dashboard** with an admin ServiceAccount and ClusterRoleBinding
- **Secrets** and **ConfigMaps** for database configuration
- A **NodePort** service to expose WordPress externally

The cluster used in this project runs on AWS EC2 with one control-plane node
(`k8-manager`) and one worker node (`k8-worker`).

---

## Repository structure

```text
manifests/
  namespace/
    ns-quota.yaml        # ResourceQuota for project7-ns
    ns-limits.yaml       # LimitRange (default/limits/requests for CPU & memory)
    # optional: namespace.yaml if you want to create project7-ns via manifest
  storage/
    mysql-pv.yaml        # NFS-backed PV for MySQL (RWO)
    mysql-pvc.yaml       # PVC bound to mysql-pv
    wordpress-pv.yaml    # NFS-backed PV for WordPress (RWX)
    wordpress-pvc.yaml   # PVC bound to wordpress-pv
  config/
    mysql-secret.yaml    # DB credentials (root/user/password, DB name)
    wordpress-config.yaml# ConfigMap with DB host and DB name
  app/
    mysql-deployment.yaml    # MySQL 5.7 Deployment (uses mysql-secret + mysql-pvc)
    mysql-service.yaml       # ClusterIP Service for MySQL
    wordpress-deployment.yaml# WordPress Deployment (uses ConfigMap, Secret + wordpress-pvc)
    wordpress-service.yaml   # NodePort Service exposing WordPress on port 30080
  dashboard/
    dashboard-admin-sa.yaml  # ServiceAccount used to log into the Dashboard
    dashboard-admin-crb.yaml # ClusterRoleBinding granting cluster-admin to the SA
