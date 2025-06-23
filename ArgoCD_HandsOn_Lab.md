# Hands-On Lab: Getting Started with Argo CD

## Introduction
Argo CD (Continuous Delivery) is a declarative, GitOps continuous delivery tool for Kubernetes. It follows GitOps principles where the desired state of Kubernetes resources is stored in Git and Argo CD continuously ensures the actual state matches the desired state.

## Prerequisites
- Kubernetes cluster (Minikube, Kind, or real cluster)
- kubectl configured to access the cluster
- Helm (optional but recommended)
- GitHub account (or any Git server)

## Step 1: Install Argo CD

### 1.1 Create `argocd` namespace
```bash
kubectl create namespace argo
```

### 1.2 Install Argo CD using manifests
```bash
kubectl apply -n argo -f https://github.com/argoproj/argo-workflows/releases/download/v3.6.10/install.yaml
#kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
It will take about 1m for all deployments to become available. Let's look at what is installed while we wait.


### 1.3 Install Argo CD using script
```bash
curl -s https://raw.githubusercontent.com/argoproj-labs/training-material/main/argo-workflows/install.sh | sh
```

### 1.3 Install Argo CD using Helm
```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argo argo/argo-workflows \
  --namespace argo \
  --create-namespace \
  --set workflow.serviceAccount.create=true

controlplane:~$ helm install argo argo/argo-workflows   --namespace argo   --create-namespace   --set workflow.serviceAccount.create=true
NAME: argo
LAST DEPLOYED: Tue Jun 17 11:18:47 2025
NAMESPACE: argo
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
1. Get Argo Server external IP/domain by running:

kubectl --namespace argo get services -o wide | grep argo-argo-workflows-server

2. Submit the hello-world workflow by running:

argo submit https://raw.githubusercontent.com/argoproj/argo-workflows/master/examples/hello-world.yaml --watch

kubectl -n argo rollout status --watch --timeout=600s deployment/argo-argo-workflows-server
controlplane:~$ kubectl -n argo port-forward --address 0.0.0.0 svc/argo-argo-workflows-server 8080:2746 > /dev/null &
```


### 1.4 Verify Pods
The Workflow Controller is responsible for running workflows:
```bash
kubectl -n argo get deploy workflow-controller
```
And the Argo Server provides a user interface and API:
```bash
kubectl -n argo get deploy argo-server
```
All pods should be in `Running` state.
```bash
kubectl -n argo wait deploy --all --for condition=Available --timeout 2m
kubectl get pods -n argo
```
All pods in a workflow run with the service account specified in workflow.spec.serviceAccountName , or if omitted, the default service account of the workflow's namespace. The amount of access which a workflow needs is dependent on what the workflow needs to do. For example, if your workflow needs to deploy a resource, then the workflow's service account will require 'create' privileges on that resource.

We do not recommend using the default service account in production. It is a shared account so may have permissions added to it you do not want. Instead, create a service account only for your workflow. So let's create a service account named argo-workflow in the argo namespace.
```bash
kubectl create serviceaccount argo-workflow -n argo
```
Then we create a clusterRole that allows the Argo workflow executor to create and patch workflow task results (the minimum require permissions for a successful workflow execution). Workflow Task Results are used to manage the progress of a workflow throughout its lifecycle.

```bash
cat <<EOF | kubectl apply -f - >/dev/null
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: workflow-executor-rbac
rules:
  - apiGroups:
      - argoproj.io
    resources:
      - workflowtaskresults
    verbs:
      - create
      - patch
EOF
```
Then we'll bind this clusterRole to the Argo service account. If you want namespace-level isolation, you can create a namespace-level role and role binding instead.
```bash
cat <<EOF | kubectl apply -f - >/dev/null
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: argo-executor-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: workflow-executor-rbac
subjects:
- kind: ServiceAccount
  name: argo-workflow
  namespace: argo
EOF
```

### What is a workflow?
A workflow is defined as a Kubernetes resource. Each workflow consists of one or more templates, one of which is defined as the entrypoint. Each template can be one of several types, in this example we have one template that is a container.
- hello-workflow.yaml
```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  name: hello
spec:
  serviceAccountName: argo-workflow # this is the service account that the workflow will run with
  entrypoint: main # the first template to run in the workflows
  templates:
  - name: main
    container: # this is a container template
      image: busybox # this is the image to run
      command: ["echo"]
      args: ["hello world"]
```
There are several other types of templates, and we'll come to more of them soon.

Because a workflow is just a Kubernetes resource, you can use kubectl with them.

Create a workflow:
```bash
kubectl -n argo apply -f hello-workflow.yaml
```
Then you can wait for it to complete (around 1m):
```bash
kubectl -n argo wait workflows/hello --for condition=Completed --timeout 2m
```
## Step 2: Access Argo CD UI
The argo-server (and thus the UI) defaults to client authentication, which requires clients to provide their Kubernetes bearer token in order to authenticate. For more information, refer to the Argo Server Auth Mode documentation.

We will switch the authentication mode to server so that we can bypass the UI login for now.

Additionally, Argo Server runs over https by default. We will disable https temporary. This is not something we recommend for production installs.

```bash
kubectl patch deployment \
  argo-server \
  --namespace argo \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/args", "value": [
  "server",
  "--auth-mode=server",
  "--secure=false"
]},
{"op": "replace", "path": "/spec/template/spec/containers/0/readinessProbe/httpGet/scheme", "value": "HTTP"}
]'
```

We need to wait for the Argo Server to redeploy:
```bash
kubectl -n argo rollout status --watch --timeout=600s deployment/argo-server
```
You can then view the user interface by running a port forward:
```bash
kubectl -n argo port-forward --address 0.0.0.0 svc/argo-server 8080:2746
#kubectl -n argo port-forward --address 0.0.0.0 svc/argo-server 2746:2746 > /dev/null &
```
You can then click here to access the UI. As it's your first time using the Workflows UI, you will see a number of modals explaining the new features. Dismiss them.


<!-- ### 2.1 Port-forward Argo CD API Server
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Access: [https://localhost:8080](https://localhost:8080)

### 2.2 Get initial admin password
```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d; echo
```
Username: `admin` -->

## Step 3: Deploy Sample Application

Lets start a workflow from the user interface:

Click "Submit new workflow":
Click "Edit using full workflow options"
Paste this YAML into the editor:

```yaml
metadata:
  generateName: hello-world-
  namespace: argo
spec:
  serviceAccountName: argo-workflow
  entrypoint: main
  templates:
    - name: main
      container:
        image: busybox
        command: ["echo"]
```
Click "Create". You will see a diagram of the workflow. The yellow icon shows that it is pending, after a few seconds it'll turn blue to indicate it is running, and finally green to show that it has completed successfully:
After about 30s, the icon will change to green:

Exercise
Take a few minutes to play around with the user interface. Find out how to:

List workflows.
View a workflow.
Resubmit a completed workflow.


### Using the CLI

To run workflows, the easiest way is to use the Argo CLI, you can install it as follows:

```bash
curl -sLO https://github.com/argoproj/argo-workflows/releases/download/v3.6.10/argo-linux-amd64.gz
gunzip argo-linux-amd64.gz
chmod +x argo-linux-amd64
mv ./argo-linux-amd64 /usr/local/bin/argo
```

To check it is installed correctly:
```bash
argo version
```
Let's run a workflow!
```bash
argo submit -n argo --serviceaccount argo-workflow --watch https://raw.githubusercontent.com/argoproj/argo-workflows/master/examples/hello-world.yaml
```
You can list workflows easily:
```bash
argo list -n argo
```
Get details about a specific workflow. @latest is an alias for the latest workflow:
```bash
argo get -n argo @latest
```
And you can view that workflows logs:
```bash
argo logs -n argo @latest
```
Finally, you can get help:
```bash
argo --help
```
Let's recap:

Argo Workflows is a workflow engine for Kubernetes.
A workflow is a Kubernetes resource, so you can use kubectl to manage them.
The user interface allows you to create and view workflows in a web browser.
The CLI also allows to create and view workflows, but in the console.


## Using the API
### Access Token
If you want to automate tasks with the Argo Server API or CLI, you will need an access token. An access token is just a Kubernetes service account token. So, to set up a service account for our automation, we need to create:

* A role with the permission we want to use.
* A service account for our automation user.
* A service account token for our service account.
* A role binding to bind the role to the service account.

In our example, we want to create a role for Jenkins so it can create, get and list workflows:

### Create the role:
```bash
kubectl create role jenkins --verb=create,get,list --resource=workflows.argoproj.io --resource=workfloweventbindings --resource=workflowtemplates
```
Create the service account:
```bash
kubectl create sa jenkins
```
Bind the service account to the role:
```bash
kubectl create rolebinding jenkins --role=jenkins --serviceaccount=argo:jenkins
```
Now we can create a token:
```bash
ARGO_TOKEN="Bearer $(kubectl create token jenkins)"
```
Print out the token:
```bash
echo $ARGO_TOKEN
```
You should see something like the following:
```text
Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6...
```
To use the token, you add it as an Authorization header to your HTTP request:
```bash
curl http://localhost:8080/api/v1/info -H "Authorization: $ARGO_TOKEN"
```
You should see something like the following:
```bash
{"modals":{"feedback":false,"firstTimeUser":false,"newVersion":false}}...
```
Now you are ready to create an Argo Workflow using the API. 
If you like to use GUI then copy Token and paste into box where token is needed to enter on WEB UI.
`
### Create a Workflow
```bash
curl \
   http://localhost:8080/api/v1/workflows/argo \
  -H "Authorization: $ARGO_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
  "workflow": {
    "metadata": {
      "generateName": "hello-world-"
    },
    "spec": {
      "templates": [
        {
          "name": "main",
          "container": {
            "image": "busybox",
            "command": [
              "echo"
            ],
            "args": [
              "hello world"
            ]
          }
        }
      ],
      "entrypoint": "main"
    }
  }
}'
```
