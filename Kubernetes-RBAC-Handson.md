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
* `serviceaccount`: identity that can access Kubernetes API.

---

## 1. Create Namespace

```bash
kubectl create namespace dev
```
---
## 2. Create `dev-user`

## Create a certificate signing request
Since Kubernetes doesn't have a "user" resource, all that's required is a client certificate and key with the common name (CN) to match the user's name.

In our case, when we created the RoleBinding, we assigned it to the user "dev-user", so that user will assume the permissions from the Role for that resource.

As long as the CN in the key is "dev-user", we will be able to use this to access the Kubernetes API.

To create a private key, we can use the openssl command-line tool. We'll use 2048 bit encryption and we'll name it dev-user.key
```bash
openssl genrsa -out dev-user.key 2048
```
Kubernetes itself is a certificate authority, therefore, it can approve and generate certificates. How convenient!

Let's create a Certificate Signing Request (CSR) for the Kubernetes API using our private key and insert the common name and output that to a file named dev-user.csr with the following command
```bash
openssl req -new -key dev-user.key -subj "/CN=dev-user" -out dev-user.csr
```
🛑IMPORTANT🛑: Make sure to insert the Common Name (CN) into your CSR, or else the certificate will become invalid
Listing the contents of your current directory should look like this:

```bash
ls
dev-user.csr  dev-user.key
```

## Submit CSR to Kubernetes API

Now that we have a CSR, we can submit it to the Kubernetes API for approval.

First, let's store the value of the CSR in an environment variable named "REQUEST"

```bash
export REQUEST=$(cat dev-user.csr | base64 -w 0)
```
Then, we can create a YAML manifest and sumbit it to the Kubernetes API. Insert the $REQUEST variable next to "request: " like so
```bash
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: dev-user
spec:
  groups:
  - system:authenticated
  request: $REQUEST
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
EOF
```

The output of the command kubectl get csr should result in the following:

```bash
kubectl get csr
NAME        AGE   SIGNERNAME                                    REQUESTOR                  REQUESTEDDURATION   CONDITION
dev-user     23s   kubernetes.io/kube-apiserver-client           kubernetes-admin           <none>              Pending
csr-z5454   13d   kubernetes.io/kube-apiserver-client-kubelet   system:node:controlplane   <none>              Approved,Issued
```

## Approve CSR

Let's assume the role that we set for our new user and test access to Kubernetes!

In order to get our client certificate that we can use in our kubeconfig, we'll approve the CSR we submitted to the Kubernetes API
```bash
kubectl certificate approve dev-user
```
The output of the command k get csr should now have the condition "Approved,Issued":
```bash
kubectl get csr
NAME        AGE     SIGNERNAME                                    REQUESTOR                  REQUESTEDDURATION   CONDITION
dev-user     2m58s   kubernetes.io/kube-apiserver-client           kubernetes-admin           <none>              Approved,Issued
csr-z5454   13d     kubernetes.io/kube-apiserver-client-kubelet   system:node:controlplane   <none>              Approved,Issued
```
### Configure Kubeconfig for `dev-user`

We can extract the client certificate out from the "kubectl get csr" command, decode it and save it to a file named dev-user.crt

```bash
kubectl get csr dev-user -o jsonpath='{.status.certificate}' | base64 -d > dev-user.crt
```
Now that we have the key and certificate, we can set the credentials in our kubeconfig and embed the certs within

```bash
kubectl config set-credentials dev-user --client-key=dev-user.key --client-certificate=dev-user.crt --embed-certs
```
🔥TIP🔥: You can remove the --embed-certs and they will remain pointers to the key and certificate files. Try it out!

The output of kubectl config view will now show dev-user as one of the users

## Test role with the new user

Next, we'll set and use the context in which kubectl uses to access the Kubernetes API
```bash
kubectl config set-context dev-user --user=dev-user --cluster=kubernetes

kubectl config use-context dev-user
```

## 3. Define Roles & Bindings

### `dev-reader` (Read-only)
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
  resources: ["deployments","replicasets"]
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
## Test Role & Role Bindings

Basic context commands:

```bash
kubectl config get-contexts	#List all available contexts
kubectl config use-context <context-name>	#Switch to a different context
kubectl config view --minify	#View details of the current context only (including cluster, user, namespace)
kubectl config view	#Show the full kubeconfig file content
```
---

Finally, we can test if our dev-user user can get pods in the dev namespace

```bash
kubectl config use-context dev-user
kubectl create deploy test -n dev --image=nginx --replicas=2
#error: failed to create deployment: deployments.apps is forbidden: User "dev-user" cannot create resource "deployments" in API group "apps" in the namespace "dev"
```
However we can see the content of the dev namespace:
```bash
kubectl get pod -n dev
#NAME                    READY   STATUS    RESTARTS   AGE
#test-6bc6b589d7-snlg4   1/1     Running   0          2m50s

kubectl config use-context kubernetes-admin@kubernetes

kubectl create deploy test -n dev --image=nginx --replicas=2
#deployment.apps/test created
kubectl delete -f dev-reader-role.yaml
```
## 4.Create `admin-user`

### Create a certificate signing request

- create a private key, we can use the openssl command-line tool. We'll use 2048 bit encryption and we'll name it admin-user.key
```bash
openssl genrsa -out admin-user.key 2048
```

- create a Certificate Signing Request (CSR) 
```bash
openssl req -new -key admin-user.key -subj "/CN=admin-user" -out admin-user.csr
```

## Submit CSR to Kubernetes API

- submit it to the Kubernetes API for approval.

```bash
export REQUEST=$(cat admin-user.csr | base64 -w 0)

cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: admin-user
spec:
  groups:
  - system:authenticated
  request: $REQUEST
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
EOF
```

## Approve CSR

- approve the CSR we submitted to the Kubernetes API
```bash
kubectl certificate approve admin-user
```
### Configure Kubeconfig for `admin-user`

- extract the client certificate out from the "kubectl get csr" command, decode it and save it to a file named admin-user.crt

```bash
kubectl get csr admin-user -o jsonpath='{.status.certificate}' | base64 -d > admin-user.crt
```
- set the credentials in our kubeconfig and embed the certs within

```bash
kubectl config set-credentials admin-user --client-key=admin-user.key --client-certificate=admin-user.crt --embed-certs
```

### Test role with the new user

- set and use the context in which kubectl uses to access the Kubernetes API

```bash
kubectl config set-context admin-user --user=admin-user --cluster=kubernetes
kubectl config get-contexts
kubectl config use-context admin-user
```

## 5. `dev-admin` (Full Access)

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
kubectl config use-context kubernetes-admin@kubernetes
kubectl apply -f dev-admin-role.yaml
kubectl config use-context admin-user
kubectl create deploy test1 -n dev --image=nginx --replicas=2
#deployment.apps/test1 created
kubectl config use-context kubernetes-admin@kubernetes
kubectl delete -f dev-admin-role.yaml
```

## 6. Service Account

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

## 7. Cleanup

```bash
rm ca.*, dev-user.*, admin-user.*
kubectl config delete-context dev-user
kubectl config delete-context admin-user
kubectl delete -f dev-reader-role.yaml
kubectl delete -f dev-admin-role.yaml
kubectl delete namespace dev
```
## 8. Notes
Some examples of Imperative ways to create role&role-bindings:

```bash
kubectl create ns web
kubectl -n web create role pod-reader --verb=get,list --resource=pods
kubectl -n web create rolebinding pod-reader-binding --role=pod-reader --user=carlton
kubectl -n web get role,rolebinding

kubectl create role example-role --verb=get,list,watch --resource=pods
kubectl create rolebinding example-rolebinding --role=example-role --serviceaccount=default:example-sa
```
