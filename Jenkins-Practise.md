# Jenkins Kubernetes CI/CD Complete Lab

## Table of Contents

1. Jenkins on Kubernetes with Dynamic Agents  
2. Docker Build & Kubernetes Integration (with Helm)  
3. Verification & Cleanup  

---

# 1. Jenkins Dynamic Agents on Kubernetes - Hands-On Lab

## 1.1 Prerequisites

- Kubernetes Cluster (Minikube/Kind/Cloud)  
- Helm v3+  
- kubectl configured  
- Docker registry (optional)  
- Jenkins with Kubernetes Plugin  

---

## 1.2 Install Jenkins via Helm

```bash
helm repo add jenkins https://charts.jenkins.io
helm repo update

helm upgrade --install jenkins jenkins/jenkins \
  --set controller.adminUser=admin \
  --set controller.adminPassword=admin \
  --set controller.serviceType=NodePort \
  --set controller.resources.requests.cpu=500m \
  --set controller.resources.requests.memory=512Mi

Get Jenkins URL:

bash
Copy
Edit
kubectl get svc jenkins