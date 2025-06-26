## Introduction
Argo CD (Continuous Delivery) is a declarative, GitOps continuous delivery tool for Kubernetes. It follows GitOps principles where the desired state of Kubernetes resources is stored in Git and Argo CD continuously ensures the actual state matches the desired state.

## Prerequisites
- Kubernetes cluster (Minikube, Kind, or real cluster)
- kubectl configured to access the cluster
- Helm (optional but recommended)
- GitHub account (or any Git server)

## Step 1: Install Argo CD

### 1.1 Create `argocd` namespace and apply official manifest
01. Installing Argo CD 
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd wait deploy --all --for condition=Available --timeout 2m
kubectl get pods -n argocd
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
# -d1TcgILt62Hk6Z5
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
02. Installing Argo CD CLI
Install Argo CD CLI, use the below commands to start with the installation
```bash
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.4.8/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```
Verify that CLI is installed
```bash
argocd
```
03. Through Personal Manifests:
```bash
kubectl create ns argocd
git clone https://github.com/T1kcan/argocd-example-apps.git
cd argocd-example-apps
kubectl apply -f install.yaml -n argocd
kubectl -n argocd wait deploy --all --for condition=Available --timeout 2m
kubectl get pods -n argocd
```
Wait until the pods are running and ready
Initial admin password is stored as a secret in argocd namespace:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
# OkYifesK7KdIMGAM
```

### 1.2 Install Argo CD using Helm
```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo/argo-cd \
  --namespace argocd \
  --set server.service.type=NodePort

  
NAME: argocd
LAST DEPLOYED: Wed Jun 25 19:59:57 2025
NAMESPACE: argocd
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
In order to access the server UI you have the following options:

1. kubectl port-forward service/argocd-server -n argocd 8080:443

    and then open the browser on http://localhost:8080 and accept the certificate

2. enable ingress in the values file `server.ingress.enabled` and either
      - Add the annotation for ssl passthrough: https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-1-ssl-passthrough
      - Set the `configs.params."server.insecure"` in the values file and terminate SSL at your ingress: https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-2-multiple-ingress-objects-and-hosts


After reaching the UI the first time you can login with username: admin and the random password generated during the installation. You can find the password by running:

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

(You should delete the initial secret afterwards as suggested by the Getting Started Guide: https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli)
```

### 1.3 Verify Pods
<!-- The Workflow Controller is responsible for running workflows:
```bash
kubectl -n argocd get deploy workflow-controller
``` -->
And the Argo Server provides a user interface and API:
```bash
kubectl -n argocd get deploy argo-server
```
All pods should be in `Running` state.
```bash
kubectl -n argocd wait deploy --all --for condition=Available --timeout 2m
kubectl get pods -n argocd
```
---

### 1.4 Access Web UI

01. Access Web UI

In this environment we exposed Argo CD server externally using node port.
```bash
controlplane:~$ kubectl get svc -n argocd argocd-server -o wide
NAME            TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE     SELECTOR
argocd-server   NodePort   10.111.162.189   <none>        80:32073/TCP,443:30164/TCP   4m49s   app.kubernetes.io/name=argocd-server
```
You can access Argo CD web ui using admin user and initial admin 'password'. 

---

02. Installing Argo CD CLI
Install Argo CD CLI, use the below commands to start with the installation
```bash
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.4.8/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```
Verify that CLI is installed
```bash
argocd
```

### Login using CLI and start using commands

In order to interact with Argo CD using CLI or web UI, Argo CD server need to be reached. In this environment we exposed Argo CD server externally using node port.
- Get the initial admin user password.
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
#wWsgk6eSY4XDYzai
```
- Login to Argo CD using CLI, use argocd login command.
```bash
example: argocd login argocdHost --grpc-web 
```
```bash
controlplane:~$ kubectl get nodes -o wide
NAME           STATUS   ROLES           AGE    VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
controlplane   Ready    control-plane   7d3h   v1.32.1   172.30.1.2    <none>        Ubuntu 24.04.1 LTS   6.8.0-51-generic   containerd://1.7.27
node01         Ready    <none>          7d3h   v1.32.1   172.30.2.2    <none>        Ubuntu 24.04.1 LTS   6.8.0-51-generic   containerd://1.7.27
controlplane:~$ kubectl get svc -n argocd argocd-server -o wide
NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE     SELECTOR
argocd-server   NodePort   10.109.112.76   <none>        80:32073/TCP,443:30164/TCP   8m15s   app.kubernetes.io/name=argocd-server
```
Since Argo CD server service is exposed using node port 32073, you can use the internal node IP and Argo CD server service port. 172.30.1.2:32073

Execute this command to login to Argo CD:
```bash
argocd login 172.30.1.2:32073 --grpc-web --plaintext
#
Username: admin           
Password: 
'admin:login' logged in successfully
Context '172.30.1.2:32073' updated

```
Enter admin as user and enter the password that you got previously.

After successful login you can start using other commands such as listing apps or clusters.
```bash
argocd cluster list --grpc-web
```
----------

## Create applications
### Create ArgoCD Application Declaratively
- Check the application definition and create an Argo CD application declaratively using below Yaml format with specs and apply it using kubectl:

- application.yaml
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: 
  name: guestbook-app
  namespace: argocd
spec: 
  destination: 
    namespace: guestbook
    server: "https://kubernetes.default.svc"
  project: default
  source: 
    path: guestbook
    repoURL: "https://github.com/T1kcan/argocd-example-apps.git"
    targetRevision: master
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```
Create the application:
```bash
kubectl apply -f application.yaml
```
Verify application is created
```bash
kubectl get application -n argocd
NAME        SYNC STATUS   HEALTH STATUS
guestbook   OutOfSync     Missing
```
Sync Applications
<!-- If needed again admin user password to login in web.
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
``` -->
Open web UI and Access ArgoCD

Sync the applications to create the resources in destination cluster.

## Creating Argo CD Application using Web UI

Create an Argo CD application using Web UI by clicking (New App), use below specs:

- Name: app-1
- project: default
- Sync options: Select auto create namespace.
- Source repo: https://github.com/T1kcan/argocd-example-apps.git , or you can fork the repo and set your repo url.
- Source path: guestbook , (path of manifests where it include k8s service and deployment files).
- Source branch: master
- Destination cluster url (local cluster): https://kubernetes.default.svc
- Destination namespace: app-1

```bash
kubectl get all -n app-1
```

- Sync the application to create the resources in destination cluster.

### Using CLI
<!-- Run the argocd command to login:
```bash
HOST="localhost:443"
PASSWD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo)
argocd login ${HOST} --username admin --password ${PASSWD} --grpc-web
``` -->
Create the application
```bash
argocd app create gbook-app \
  --repo https://github.com/T1kcan/argocd-example-apps.git \
  --revision master \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace app-2 \
  --sync-option CreateNamespace=true \
  --grpc-web
```
Verify app is created
```bash
argocd app list --grpc-web
```
Sync the application:
```bash
argocd app sync gbook-app --grpc-web
```
Verify app is synced
```bash
argocd app list --grpc-web

```
## Check reconciliation
### Now do GitOps

Replica Count Update
- Update one of the applications and click on the APP DETAILS,SYNC POLICY, ENABLE AUTO-SYNC , PRUNE RESOURCES and SELF HEAL buttons.

- Update the replica count on the guestbook-ui-deployment , e.g.

# github.com/T1kcan/argocd-example-apps/blob/master/guestbook/guestbook-ui-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: guestbook-ui
spec:
  replicas: 1  # change this to 3
  revisionHistoryLimit: 3
```
...
Git commit and push

Watch the changes on the UI

Check the other application: It should be "Out of Sync"

Try it manually via kubectl

Update 1 of the applications and click on the ENABLE AUTO-SYNC , PRUNE RESOURCES and SELF HEAL buttons.

Patch the deployment manually on the app-2 namespace:
```bash
kubectl patch deployment guestbook-ui \
  -n app-2 -p '{"spec":{"replicas":5}}'
```

## Kustomize & Helm

### Kustomize and Helm

#### Kustomize
Investigate another application that uses kustomize:

- kapplication.yaml
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: 
  name: kdemo
  namespace: argocd
spec: 
  destination: 
    namespace: kustomize
    server: "https://kubernetes.default.svc"
  project: default
  source: 
    path: kustomize-guestbook
    repoURL: "https://github.com/T1kcan/argocd-example-apps.git"
    targetRevision: master
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Create the application:
```bash
kubectl apply -f kapplication.yaml
```
Change the image tag on kustomization.yaml , commit and push:

# github.com/T1kcan/argocd-example-apps/blob/master/kustomize-guestbook/kustomization.yaml
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namePrefix: kustomize-

images:
- name: gcr.io/heptio-images/ks-guestbook-demo
  newTag: "0.1"   # change this to "0.2"

resources:
- guestbook-ui-deployment.yaml
- guestbook-ui-svc.yaml
```

#### Helm
ArgoCD supports deploying Helm charts, however it does not deploy it as a Helm release. It runs helm template first and then deploys the manifest If you want to have helm releases deployed, then you should you flux.

## Helm Options
Practice creating application that deploy a helm chart that exist in a git repo, in this practice we will cover:

Declare an Argo CD application declaratively that deploy a helm chart.
Create the application using kubectl.
Sync the application and verify resources are created in cluster.
Update the helm release name in Argo CD application definition.
Re-sync the app and prune the old resources.

- helm-application.yaml
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: 
  name: helm-app
  namespace: argocd
spec: 
  destination: 
    namespace: helm-app
    server: "https://kubernetes.default.svc"
  project: default
  source: 
    path: helm-guestbook
    repoURL: "https://github.com/T1kcan/argocd-example-apps.git"
    targetRevision: master
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

Create the application using kubectl
```bash
kubectl apply -f helm-application.yaml
```
Verify application is created
```bash
kubectl get application -n argocd
```
Sync the application to create the resources in destination cluster.
Note: by default in Argo CD, helm release name is equal to app name unless we specify it explicitly.

Update release name:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: 
  name: helm-app
  namespace: argocd
spec: 
  destination: 
    namespace: helm-app
    server: "https://kubernetes.default.svc"
  project: default
  source: 
    path: helm-guestbook
    repoURL: "https://github.com/T1kcan/argocd-example-apps.git"
    targetRevision: master
    helm:
     releaseName: my-release
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```
```bash
kubectl apply -f application.yaml
kubectl get application -n argocd
```
Sync the application to create the resources in destination cluster.

## Directory of Files Options

Practice creating application that deploy a directory of Yaml files that exist in a git repo , in this practice we will cover:

Declare an Argo CD application declaratively that deploy a directory of Yaml files.
Create the application using kubectl.
Sync the application and verify resources are created in cluster.
Update Argo CD application definition to include all files in all sub-directories using recursive option.
Re-sync the app.

### Create Argo CD Application
By default running manifest underneath subdirectories are disabled. 
Test the differences with applying following application:
- application3.yaml
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: 
  name: directory-app
  namespace: argocd
spec: 
  destination:
    namespace: directory-app
    server: "https://kubernetes.default.svc"
  project: default
  source: 
    path: guestbook-with-sub-directories
    repoURL: "https://github.com/T1kcan/argocd-example-apps.git"
    targetRevision: master
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```
Sync the app.
Observe the result on argocd ui, then update yaml file as follows:
application3.yaml
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: 
  name: directory-app
  namespace: argocd
spec: 
  destination:
    namespace: directory-app
    server: "https://kubernetes.default.svc"
  project: default
  source: 
    path: guestbook-with-sub-directories
    repoURL: "https://github.com/T1kcan/argocd-example-apps.git"
    targetRevision: master
    directory:
      recurse: true
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```
Re-sync the app.

##  Sync Policies, Automated Sync
- application4.yaml
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: 
  name: auto-sync-app
  namespace: argocd
spec:
  destination:
    namespace: auto-sync-app
    server: "https://kubernetes.default.svc"
  project: default
  source: 
    path: guestbook-with-sub-directories
    repoURL: "https://github.com/T1kcan/argocd-example-apps.git"
    targetRevision: master
    directory:
      recurse: true
  syncPolicy:
    automated: {}
    syncOptions:
      - CreateNamespace=true
```
Observe that the new application is being automatically deployed.

## Remote Clusters

We need to get the below info related to the cluster:

- Cluster API Url.
- Bearer token with admin role.
- Certificate authority data.

Now, connect the remote cluster in Argo CD declaratively:

You can use the secret file cluster.yaml
```yaml
apiVersion: v1
kind: Secret
metadata:
  namespace: argocd
  name: remote-cluster
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: remote-cluster
  server:
  config: |
    {
      "bearerToken": "",
      "tlsClientConfig": {
        "insecure": false,
        "caData": ""
      }
    }
```
```bash
Name: remote-cluster
Server: set the remote cluster public API Url
bearerToken: set the bearerToken
caData: set certificate authority data
```
## Step 4: Application Lifecycle Operations

### 4.1 Refresh Application
```bash
argocd app refresh guestbook
```

### 4.2 Rollback to Previous Version
```bash
argocd app history guestbook
argocd app rollback guestbook <revision>
```

### 4.3 Delete Application
```bash
argocd app delete guestbook --cascade
```

## Cleanup
```bash
kubectl delete namespace argocd
kubectl delete namespace default  # if you created demo apps there
```

## Summary
You have successfully:
- Installed Argo CD
- Accessed the Argo CD UI
- Deployed and managed a sample app
- Practiced syncing, rolling back, and deleting applications

## References
- [Argo CD Official Docs](https://argo-cd.readthedocs.io/en/stable/)
- [Argo CD GitHub Repo](https://github.com/argoproj/argo-cd)
