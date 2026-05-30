# 🏁 EKS & GitOps Master Bootstrapping Guide

Welcome to your **Ultimate Quick-Start & Troubleshooting Cheat-Sheet**! This guide contains every command you need to launch, configure, access, and verify the entire microservices cluster from absolute scratch.

---

## 🏗️ Phase 1: Infrastructure Deployment (Terraform)

Because EKS has chicken-and-egg provider dependencies (e.g., Helm cannot configure ArgoCD before EKS is fully running), we deploy in **3 targeted phases**.

Run these commands inside the `terraform/` directory:

```bash
cd terraform
```

### 1. Initialize Terraform
Downloads the required AWS, Helm, and Kubectl providers:
```bash
terraform init
```

### 2. Phase 1: Deploy VPC
Creates the network base (subnets, NAT gateways, route tables):
```bash
terraform apply -target=module.vpc -auto-approve
```

### 3. Phase 2: Deploy EKS Cluster
Spins up the EKS Control Plane and the EKS Auto Mode general node pool:
```bash
terraform apply -target=module.retail_app_eks -auto-approve
```

### 4. Phase 3: Deploy ArgoCD & Addons
Deploys Nginx Ingress, Cert-Manager, ArgoCD controllers, and mounts the Kubernetes resources:
```bash
terraform apply -auto-approve
```

---

## 🔌 Phase 2: Connect to EKS Cluster (`kubectl` Setup)

Once EKS is created, you must configure your local machine to connect and control the cluster.

### 1. Update Kubeconfig
Run this command to fetch cluster credentials and store them locally (replace `us-west-2` if your AWS region is different):
```bash
aws eks update-kubeconfig --name retail-store-dprw --region us-west-2
```

### 2. Verify Connection
Check if your local machine can see the running nodes inside EKS:
```bash
kubectl get nodes
```

---

## 🐙 Phase 3: Access & Configure ArgoCD Dashboard

ArgoCD manages the deployments of your microservices. Let's retrieve its credentials and access the web UI.

### 1. Retrieve ArgoCD Initial Admin Password
Kubernetes encrypts the default admin password inside secrets. Decrypt and read it:
```bash
# Windows (PowerShell)
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }

# Linux / Mac / Git Bash (Bash)
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```
*(Copy the output string. This is your admin password!)*

### 2. Start Port-Forwarding to ArgoCD Server
ArgoCD is running inside a private network in the cluster. Port-forward it to your local machine (localhost:8080):
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
*(Keep this terminal tab open!)*

### 3. Access ArgoCD UI
* Open your browser and navigate to: **`https://localhost:8080`** (ignore the SSL self-signed certificate warning).
* **Username**: `admin`
* **Password**: *The decrypted password you copied in Step 1*

---

## 🌐 Phase 4: Access Your Retail Store Application

Your microservices are exposed through the Nginx Ingress Controller using an AWS Classic Load Balancer.

### 1. Get the LoadBalancer DNS URL
Run this command to get the public DNS endpoint of your entry-point Load Balancer:
```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```
*(Wait 2-3 minutes for AWS to provision the Load Balancer before accessing it).*

### 2. Access the Application
* Copy the DNS output (looks like `a8b9c...us-west-2.elb.amazonaws.com`).
* Paste it into your browser to view your live, running retail store UI!

---

## 🔍 Useful Troubleshooting & Operations Commands

| Objective | Command |
| :--- | :--- |
| **List Running Pods** | `kubectl get pods -n retail-store` |
| **Check Ingress Rules** | `kubectl get ingress -n retail-store` |
| **Check Pod Logs (e.g., UI)**| `kubectl logs -n retail-store -l app.kubernetes.io/name=ui` |
| **Watch ArgoCD Application Status** | `kubectl get applications -n argocd` |
| **Delete Entire Infrastructure (Cleanup)**| `terraform destroy --auto-approve` |
