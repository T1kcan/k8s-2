
---
#ArgoCD HandsOn Lab

## Installation

01. Setting-Up Argo CD

Install Argo CD using Non-HA manifests in argocd namespace.
After installing Argo CD, make sure that pods are running and ready.

```bash
kubectl create ns argocd
wget https://raw.githubusercontent.com/T1kcan/argocd-example-apps/refs/heads/master/install_argocd.yaml
kubectl apply -f install_argocd.yaml
```
Wait until the pods are running and ready
Initial admin password is stored as a secret in argocd namespace:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
# OkYifesK7KdIMGAM
```
02. Access Web UI

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

example: argocd login argocdHost --grpc-web .
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
```
Enter admin as user and enter the password that you got previously.

After successful login you can start using other commands such as listing apps or clusters.
```bash
argocd cluster list --grpc-web

```

##  Creating Argo CD Application Declaratively

01. Create Argo CD Application

Create an Argo CD application declaratively using below Yaml format with specs and apply it using kubectl:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: 
  name: 
  namespace: argocd
spec: 
  destination: 
    namespace: 
    server: ""
  project: default
  source: 
    path: 
    repoURL: ""
    targetRevision: 
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```
We are going to deploy a sample guestbook application: 
- Name: guestbook
- Destination cluster url (local cluster): https://kubernetes.default.svc
- Destination namespace: guestbook
- Source repo: https://github.com/T1kcan/argocd-example-apps.git , or you can fork the repo and set your repo url.
- Source path: guestbook , (path of manifests where it include k8s service and deployment files).
- Source branch: master

Create an Argo CD application declaratively using Yaml with below specs and apply it using kubectl:

- application.yaml
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
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
Create the application using kubectl:
```bash
kubectl apply -f application.yaml
```
Verify application is created:
```bash
controlplane:~$ kubectl get application -n argocd -w
NAME        SYNC STATUS   HEALTH STATUS
guestbook   OutOfSync     Missing
```
03. Sync Application
You need admin user password to login in web:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
# ThIrhmlKK2tvNhQf
```
Sync the application to create the resources in destination cluster.


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





## Creating Argo CD Application using CLI
```bash
kubectl apply -f https://raw.githubusercontent.com/argoproj/argocd-example-apps/master/manifests/guestbook.yaml -n argocd
```

Or use `argocd` CLI (after installing):
```bash
argocd login 172.30.1.2:32073 --grpc-web --plaintext
argocd app create app-2 \
  --repo https://github.com/T1kcan/argocd-example-apps.git \
  --revision master \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace app-2 \
  --sync-option CreateNamespace=true \
  --grpc-web
#application 'app-2' created
argocd app list --grpc-web
argocd app sync app-2 --grpc-web
argocd app list --grpc-web
```




## Helm Options
Practice creating application that deploy a helm chart that exist in a git repo , in this practice we will cover:

Declare an Argo CD application declaratively that deploy a helm chart.
Create the application using kubectl.
Sync the application and verify resources are created in cluster.
Update the helm release name in Argo CD application definition.
Re-sync the app and prune the old resources.

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
kubectl apply -f /practice/application.yaml
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
    repoURL: "https://github.com/mabusaa/argocd-example-apps.git"
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



### 3.1 Clone the sample app repo
```bash
git clone https://github.com/argoproj/argocd-example-apps.git
cd argocd-example-apps
```


argocd app create guestbook \
    --repo https://github.com/argoproj/argocd-example-apps.git \
    --path guestbook \
    --dest-server https://kubernetes.default.svc \
    --dest-namespace default
```


### 3.3 Sync the App
```bash
argocd app sync guestbook
```

### 3.4 Verify
```bash
kubectl get pods -n default
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

## Step 5: Argo CD CLI Installation (Optional)

On Linux/macOS:
```bash
brew install argocd
```
Or manually:
```bash
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/
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


```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-argo-application
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/T1kcan/argocd-example-apps
    targetRevision: HEAD
    path: guestbook
  destination: 
    server: https://kubernetes.default.svc
    namespace: test

  syncPolicy:
    syncOptions:
    - CreateNamespace=true

    automated:
      selfHeal: true
      prune: true
```

```bash
git clone -b scenarios https://github.com/T1kcan/argocd-course-apps-definitions.git
cd argocd-course-apps-definitions/argocd-install/overlays/production/
kubectl apply -k . -n argocd


curl -s https://raw.githubusercontent.com/argoproj-labs/training-material/main/argo-workflows/install.sh | sh
```
template example:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: container-
spec:
  entrypoint: main
  templates:
  - name: main
    container:
      image: busybox
      command: [echo]
      args: ["hello world"]
```

```bash
argo submit --serviceaccount argo-workflow --watch container-workflow.yaml
```
Port-forward to the Argo Server pod...
```bash
kubectl -n argo port-forward --address 0.0.0.0 svc/argo-server 2746:2746 > /dev/null &
```