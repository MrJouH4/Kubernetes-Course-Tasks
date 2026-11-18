# 🚀 Kubernetes Task Session 2: FastAPI + MySQL Deployment with ConfigMap, Secret, & Ingress

## 📌 Overview
This repository folder contains the full Kubernetes manifest suite and step-by-step implementation guide for deploying a scalable **FastAPI** web application connected to a persistent **MySQL 8.4** database instance on a local Kubernetes cluster using `minikube`.

### Core Architecture Components
- **Namespace (`webapp-stack`):** Isolated dedicated namespace for all application resources.
- **Secrets (`mysql-secret`):** Securely stores database root password, application database username, and user password.
- **ConfigMap (`webapp-config`):** Holds non-sensitive environment configuration including application title, environment, database host, port, and database name.
- **Storage & Persistence:** 2Gi `PersistentVolume` (`mysql-pv`) bound to a 2Gi `PersistentVolumeClaim` (`mysql-pvc`) using `manual` storage class, pinned to node `minikube-m02` via `nodeAffinity` and host path `/data/webapp-mysql`.
- **MySQL Database Deployment:** 1 replica Deployment using `Recreate` strategy, mounted at `/var/lib/mysql`, with liveness and readiness health probes via `mysqladmin ping`.
- **FastAPI Application Deployment:** 2 replica Deployment consuming configuration via `envFrom` and secrets via `env`, with HTTP health probes on `/health` (port 8000).
- **Networking & Ingress:** Internal `ClusterIP` services for MySQL (port 3306) and FastAPI (port 80), exposed externally via NGINX Ingress Controller (`webapp-ingress`).

---

## 🛠️ Kubernetes Manifests Summary

The deployment consists of 7 modular YAML files located in `Task-2/`:

1. `01-namespace.yaml`: Creates the `webapp-stack` namespace.
2. `02-secret.yaml`: Defines `mysql-secret` containing `root-password`, `app-user`, and `app-password`.
3. `03-configmap.yaml`: Defines `webapp-config` containing `APP_TITLE`, `APP_ENV`, `MYSQL_HOST`, `MYSQL_PORT`, and `MYSQL_DATABASE`.
4. `04-pv-pvc.yaml`: Configures the 2Gi `mysql-pv` with `nodeAffinity` targeting `minikube-m02` and the bound `mysql-pvc`.
5. `05-mysql.yaml`: Deploys MySQL using `APP_DB_USER` and `APP_DB_PASSWORD` environment variable bindings expected by `webapp-mysql:task2` initialization script.
6. `06-webapp.yaml`: Deploys 2 FastAPI replicas exposed via `webapp-api` service on port 80.
7. `07-ingress.yaml`: Configures `webapp-ingress` with `ingressClassName: nginx` to route incoming traffic to `webapp-api`.

---

## 💻 Step-by-Step Deployment Instructions

### 1. Build and Load Container Images
From your project root, build the local Docker images and load them into your Minikube cluster:
```bash
# Build Docker images
docker build -t webapp-api:task2 ./webapp
docker build -t webapp-mysql:task2 ./mysql

# Load images into Minikube image cache
minikube image load webapp-api:task2
minikube image load webapp-mysql:task2
```

### 2. Enable Ingress Controller
```bash
minikube addons enable ingress
```

### 3. Apply Kubernetes Manifests in Sequential Order
```bash
kubectl apply -f Task-2/01-namespace.yaml
kubectl apply -f Task-2/02-secret.yaml
kubectl apply -f Task-2/03-configmap.yaml
kubectl apply -f Task-2/04-pv-pvc.yaml
kubectl apply -f Task-2/05-mysql.yaml
kubectl apply -f Task-2/06-webapp.yaml
kubectl apply -f Task-2/07-ingress.yaml
```

### 4. Port Forward Ingress Controller (For Local Access)
In a separate terminal window, forward port 8080 to the ingress controller:
```bash
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80
```

---

## 🔍 Verification & Testing Checklist

Save your terminal screenshots into `Task-2/images/` and verify each step:

| Step | Command / Action | Expected Result | Screenshot Name |
| :--- | :--- | :--- | :--- |
| **1. Loaded Images** | `minikube image ls \| grep webapp` | Displays `webapp-api:task2` & `webapp-mysql:task2` | `Loaded Images.png` |
| **2. Secret & ConfigMap** | `kubectl get secret,cm -n webapp-stack` | Displays `mysql-secret` and `webapp-config` | `Secrets & ConfigMaps.png` |
| **3. PV & PVC Status** | `kubectl get pv,pvc -n webapp-stack` | Status shows `Bound` for `mysql-pv` & `mysql-pvc` | `PV & PVC Bound.png` |
| **4. Pods & Deployments** | `kubectl get deploy,pods -n webapp-stack -o wide` | `mysql` (1/1) & `webapp-api` (2/2) `Running` | `Pods & Deployments Ready.png` |
| **5. Ingress Status** | `kubectl get ingress -n webapp-stack` | `webapp-ingress` has ingress class `nginx` | `Ingress Active.png` |
| **6. API Info Check** | `curl http://localhost:8080/api/info` | Returns JSON with app title and environment | `API Status Info.png` |
| **7. Persistence Test** | `kubectl delete pod -n webapp-stack -l app=mysql`<br>`curl http://localhost:8080/api/messages` | New MySQL pod attaches PV and retains written messages | `Data Persistence.png` |

---

## ⚠️ Key Troubleshooting & Windows Notes

### 1. MySQL Environment Variable Names (`APP_DB_USER` vs `MYSQL_USER`)
- **Important:** The custom image `webapp-mysql:task2` uses an initialization script (`01-setup.sh`) that specifically checks for `APP_DB_USER` and `APP_DB_PASSWORD` (not standard `MYSQL_USER`). 
- `05-mysql.yaml` must pass `APP_DB_USER` and `APP_DB_PASSWORD` from `mysql-secret`.
- `06-webapp.yaml` must pass `MYSQL_USER` and `MYSQL_PASSWORD` from `mysql-secret` for FastAPI's connection string.

### 2. Clearing Stale MySQL HostPath Data Directory
If MySQL started with incorrect configuration, it creates data files on the host path `/data/webapp-mysql` and skips setup on subsequent restarts. To perform a clean database reset:
```bash
# 1. Delete deployment and PVC
kubectl delete deployment mysql -n webapp-stack
kubectl delete pvc mysql-pvc -n webapp-stack

# 2. Wipe storage directory inside Minikube VM (quotes & -- required)
minikube ssh -n minikube-m02 -- "sudo rm -rf /data/webapp-mysql"

# 3. Re-apply storage and deployment manifests
kubectl apply -f Task-2/04-pv-pvc.yaml
kubectl apply -f Task-2/05-mysql.yaml
```

### 3. Windows Shell Compatibility
- **`chmod` on Windows:** Standard Windows Command Prompt and PowerShell do not recognize `chmod`. You can safely skip `chmod -x mysql/initdb/01-setup.sh` when building with Docker on Windows or use Git Bash / `git update-index --chmod=-x`.
- **PowerShell Quote Escaping:** In PowerShell, use `curl.exe` and escape JSON double quotes with backslashes `\"`:
  ```powershell
  curl.exe -X POST http://localhost:8080/api/messages -H "Content-Type: application/json" -d '{\"author\":\"student\",\"content\":\"hello\"}'
  ```
