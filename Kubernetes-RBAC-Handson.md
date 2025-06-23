# Kubernetes RBAC (Role-Based Access Control) – Detailed Explanation & Hands-on Lab

Kubernetes RBAC is an authorization mechanism that controls which users can perform which actions on which resources. It is critical for security, operational control, and fault tolerance, and it follows the principle of **least privilege**.

---

## Why Use RBAC?

* **Security**: Prevents unauthorized access and reduces accidental or malicious changes.
* **Auditability & Compliance**: Clear traceability of who accessed what and when.
* **Operational Control**: Teams operate within their defined boundaries, reducing complexity.
* **Error Reduction**: Limits the risk of accidental deletion or modification.

---

## RBAC Components

### **Role**

Defines permissions to resources (Pods, Deployments, Services) within a specific namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

### **ClusterRole**

Defines resource permissions cluster-wide, regardless of namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "watch", "list"]
```

### **RoleBinding**

Binds a Role or ClusterRole to a user/group/service account within a namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-in-default
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### **ClusterRoleBinding**

Binds a ClusterRole to a user/group for all namespaces.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes-globally
subjects:
- kind: User
  name: john
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## Types of Users

* **Users**: Typically managed through LDAP, OAuth2/OIDC, or certificate-based authentication.
* **Groups**: Usually imported from external identity providers (LDAP, OIDC).
* **ServiceAccounts**: Identities used by pods to interact with the API server.

---

## RBAC Rules

Each rule includes the following fields:

* **apiGroups**: API group of the resource (e.g., `""`, `apps`, `rbac.authorization.k8s.io`).
* **resources**: Resources to which access is controlled (`pods`, `deployments`, etc.).
* **verbs**: Allowed actions (`get`, `list`, `create`, `delete`, etc.).
* **resourceNames** *(optional)*: Targets specific resource names.

---

## Authorization Flow

1. **Authentication**: The API server verifies the identity.
2. **Authorization**: Access is checked against RoleBindings and ClusterRoleBindings.
3. **Admission Control**: Enforces policies and quotas.

---

## Hands-on Lab Steps

### Prerequisites

* A Kubernetes cluster (Minikube, kind, k3s, etc.)
* `kubectl` installed
* `openssl`

### Scenario

* `dev-user`: read-only access to Pods/Deployments/Services in the `dev` namespace.
* `admin-user`: full access in the `dev` namespace.
* `ci-cd-serviceaccount`: CI/CD pipeline identity that can manage Deployments and Pods in `dev`.

---

### 1. Create Namespace

```bash
kubectl create namespace dev
```

---

### 2. Generate Certificate – `dev-user`

```bash
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -subj "/CN=kube-ca" -days 10000 -out ca.crt

openssl genrsa -out dev-user.key 2048
openssl req -new -key dev-user.key -subj "/CN=dev-user/O=developers" -out dev-user.csr

openssl x509 -req -in dev-user.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out dev-user.crt -days 500
```

---

### 3. Configure Kubeconfig for `dev-user`

```bash
CLUSTER_NAME=$(kubectl config view --minify -o jsonpath='{.clusters[0].name}')
CLUSTER_SERVER=$(kubectl config view -o jsonpath="{.clusters[?(@.name==\"${CLUSTER_NAME}\")].cluster.server}")

kubectl config set-credentials dev-user \
  --client-certificate=dev-user.crt \
  --client-key=dev-user.key \
  --embed-certs=true

kubectl config set-context dev-user-context \
  --cluster="$CLUSTER_NAME" \
  --user=dev-user \
  --namespace=dev
```
---
```bash
kubectl config get-contexts	#List all available contexts
kubectl config use-context <context-name>	#Switch to a different context
kubectl config view --minify	#View details of the current context only (including cluster, user, namespace)
kubectl config view	#Show the full kubeconfig file content
```
---

### 4. Define Roles & Bindings

#### 4.1 `dev-reader` (Read-only)
- dev-reader-role.yaml
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: dev-reader
rules:
- apiGroups: [""]
  resources: ["pods","services","configmaps","persistentvolumeclaims"]
  verbs: ["get","list","watch"]
- apiGroups: ["apps"]
  resources: ["deployments","replicasets", "pods"]
  verbs: ["get","list","watch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: dev-reader-binding
  namespace: dev
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: dev-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f dev-reader-role.yaml
```
# Test Role & Role Bindings
```bash
kubectl config use-context dev-user-context
kubectl create deploy test -n dev --image=nginx --replicas=2
#error: failed to create deployment: Unauthorized
kubectl config use-context kubernetes-admin@kubernetes
kubectl create deploy test -n dev --image=nginx --replicas=2
#deployment.apps/test created
kubectl delete -f dev-reader-role.yaml
```
### 4.1. Admin User
### 4.2. Generate Certificate – `admin-user`

```bash
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -subj "/CN=kube-ca" -days 10000 -out ca.crt
openssl genrsa -out admin-user.key 2048
openssl req -new -key admin-user.key -subj "/CN=admin-user/O=developers" -out admin-user.csr
openssl x509 -req -in admin-user.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out admin-user.crt -days 500
```
---

### 4.3. Configure Kubeconfig for `admin-user`

```bash
CLUSTER_NAME=$(kubectl config view --minify -o jsonpath='{.clusters[0].name}')
CLUSTER_SERVER=$(kubectl config view -o jsonpath="{.clusters[?(@.name==\"${CLUSTER_NAME}\")].cluster.server}")
kubectl config set-credentials admin-user   --client-certificate=admin-user.crt   --client-key=admin-user.key   --embed-certs=true
kubectl config set-context admin-user-context   --cluster="$CLUSTER_NAME"   --user=admin-user   --namespace=dev
```

#### 4.4 `dev-admin` (Full Access)
- dev-admin-role.yaml
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: dev-admin
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: dev-admin-binding
  namespace: dev
subjects:
- kind: User
  name: admin-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: dev-admin
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f dev-admin-role.yaml
kubectl config use-context admin-user-context
kubectl create deploy test1 -n dev --image=nginx --replicas=2
#deployment.apps/test1 created
kubectl config use-context kubernetes-admin@kubernetes
kubectl delete -f dev-admin-role.yaml
```

### 5. Service Account

```bash
# Create a ServiceAccount:
kubectl create serviceaccount my-service-account
#  Verify:
kubectl get serviceaccount my-service-account
# Check Secrets Attached to ServiceAccount
kubectl describe serviceaccount my-service-account
# Notice: No secrets or tokens are created automatically anymore, autocrate secret deprecated by 1.24+.
#Generate a Token for the ServiceAccount (Manual Token)
kubectl create token my-service-account
# Create a Kubernetes Secret Manually with this Token
kubectl create secret generic my-sa-token   --from-literal=token=$(kubectl create token my-service-account)   --type=service-account-token
# Verify:
kubectl get secrets my-sa-token -o yaml
# Use This Token to Access Kubernetes API (Example)
kubectl cluster-info
TOKEN=$(kubectl create token my-service-account)
curl -k -H "Authorization: Bearer $TOKEN" https://172.30.1.2:6443/api
curl -k -H "Authorization: Bearer $TOKEN" https://172.30.1.2:6443/apis
# Try without Token and observe that cannot access:
curl -k https://172.30.1.2:6443/apis
```
---

### 6. Cleanup

```bash
rm ca.*, dev-user.*, admin-user.*
kubectl config delete-context dev-user-context
kubectl config delete-context admin-user-context
kubectl delete -f dev-reader-role.yaml
kubectl delete -f dev-admin-role.yaml
kubectl delete namespace dev
```
