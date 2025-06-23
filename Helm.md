# Kubernetes Helm
In this lab we will be setting up HELM, installing a helm chart, and creating a basic helm chart.
https://helm.sh/

In this lab we will install Helm and explore setting up a chart to install a complex application (frontend and backend)

Docs and sources:

https://artifacthub.io/

https://helm.sh/docs

Run Ubuntu updates:
```bash
apt-get update -y

apt install -y tree jq
```
## INSTALL HELM TWO WAYS:
1. install helm Maually (v3.8.2)
install helm3 (from https://github.com/helm/helm/releases)
```bash
wget https://get.helm.sh/helm-v3.8.2-linux-amd64.tar.gz
tar -zxvf helm-v3.8.2-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
```

2. by script (latest)
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```
and check the top command (will take a couple of minutes to set getting metrics)
```bash
helm version
```
Check k8s is running
```bash
kubectl cluster-info
kubectl top nodes
kubectl top pods
```
it might take a couple of minutes, but your should get Kubernetes master is running at

## INSTALL A SAMPLE CHART
### Install metrics-server
There are two way to search repos from the command line:

* helm search hub # searchs the artifact hub at: https://artifacthub.io/ or https://bitnami.com/stacks/helm, although the artifact hub is better layed out
* helm search repo # search the local repo you've added repo's to
#### Using the Repo
* search the repo (all repos that have been added), note each has a chart version and an app version
```bash
helm search repo # None found, so lets add one
helm repo add bitnami https://charts.bitnami.com/bitnami
helm search repo
```
If you ever need to update: 
```bash
helm repo update
```
Here's an example of a chart install, which we've called my-metrics-server
Use this helm chart
```bash
helm search repo metrics-server -l
helm install my-metrics-server bitnami/metrics-server \
  --version=7.3.5 \
  --namespace kube-system \
  --set apiService.create=true \
  --set extraArgs[0]=--kubelet-insecure-tls \
  --set extraArgs[1]=--kubelet-preferred-address-types=InternalIP
```
You can view the charts for bitnami at: https://bitnami.com/stacks/helm

to add addictional parameters: helm install --set param=vale , or supply a 'values' yaml file with the vales --values file.yaml
eg service.port=80

pods/volumes not coming up?
some charts require storage, run:
```bash
kubectl get pod -n kube-system -w
kubectl get pvc -A
kubectl top nodes
kubectl top pods -A
```
to see what the deployment is waiting for

Lets check the helm chart is installed (-A shows all namespaces)
```bash
helm list -A
```
Also note that information is stored in ~/.cache/helm/:
```bash
ls ./.cache/helm/repository/
```
Name this is the release name
App version: this is the version of the actual app Chart Version: this is the version of the chart, every time there is a change to the chart, the chart version is incremented, and you'll see it in the end of the chart name
```bash
helm status my-metrics-server -n kube-system
```
Lets check the endpoint is up (it will take a few minutes)
```bash
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes | jq
```
tip: you can add the --debug argument to troubleshoot

connect to the uri
```bash
kubectl get svc -A
```
We'll port forward to this machine:

Update with the correct values:
```bash
export POD_NAME=$(kubectl get pods --namespace kube-system -l "app.kubernetes.io/name=metrics-server,app.kubernetes.io/instance=my-metrics-server" -o jsonpath="{.items[0].metadata.name}")

export CONTAINER_PORT=$(kubectl get pod --namespace kube-system $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")

kubectl --namespace kube-system port-forward $POD_NAME 8080:$CONTAINER_PORT
```
and connect http://localhost:8080

Check metrics-server
let check it's installed, since it's installed in the kube-system namespace, we have to add the --namespace argument
```bash
helm list -A

helm get notes my-metrics-server
```
and lets check what values have been used:
```bash
helm get values my-metrics-server
```
To get a pervious release, you can use --revision <release number>

Pull down and examine the chart
lets pull down and look at the metric server on the bitnami repo
```bash
helm pull bitnami/metrics-server

tar -zxvf metrics-server-*.tgz

tree metrics-server

cd metrics-server/
```
all the files in the template folder will be processed with Go Templating to produce a yaml file for a k8s apply file

lets see the output, as text, when we process this chart
```bash
helm template .
```
You can override the values in the values.yaml folder when processing, by using '--set'

## CREATE YOUR OWN CHART
We'll install the sample chart provided by helm, this example automatically uses an nginx image:
```bash
cd ~

helm create examplechart

cd examplechart

tree
```
take a read of the chart file
```bash
cat Chart.yaml
```
Note the version: 0.1.0 that defines the chart version, The ApiVersion=v2 , which is actuall Helm 3

The charts folder is for dependant charts for this chart, but we won't using these in this demo. Note the version: 0.1.0 that defines the chart version
```bash
cat values.yaml
```
The values yaml file is for values that you'll want to change every now and then. EG a service port number, again you can override using --set (eg service.port=80)

lets look at the generated yaml, but change a value
```bash
helm template . --set replicaCount=2
```
Lets run a helm lint on this chart to make sure its ok
```bash
helm lint
```
Now install the chart
```bash
cd ~
helm install new-chart examplechart/ --values examplechart/values.yaml
helm list -A
helm status new-chart
kubectl get svc -A
```
We'll port forward to this machine:
```bash
export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=examplechart,app.kubernetes.io/instance=new-chart" -o jsonpath="{.items[0].metadata.name}")

export CONTAINER_PORT=$(kubectl get pod --namespace default $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")

kubectl --namespace default port-forward $POD_NAME 8080:$CONTAINER_PORT
```
and test it:
```bash
curl localhost:8080
```
to remove

helm uninstall new-chart

## GENERATE FROM SCRATCH
Bare Template
```bash
cd ~
helm create my-app
cd my-app
```
lets remove files we will use from scratch
```bash
rm -rf ./templates/*
echo "" > values.yaml
```

### Sample Template

lets get a template from https://k8syaml.com/, just copy the default template will be fine. (you can remove the pod affinity if you wish. Use the copy function in the top right)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: octopus-deployment
  labels:
    app: web
spec:
  selector:
    matchLabels:
      octopusexport: OctopusExport
  replicas: 1
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: web
        octopusexport: OctopusExport
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
```
can paste it into the templates folder
```bash
vi ./templates/app.yaml
```
Now since there is no templating, the yaml file generated will only have comments inserted
```bash
helm template . 
```
### Add a value for the template
```bash
echo  "replicas: 2 " >> values.yaml
controlplane:~/my-app$ cat values.yaml 
replicas: 2  
```
and in the template/app.yaml file, change the replicas line to   replicas: {{ .Values.replicas }}

```bash
vi ./templates/app.yaml
```
and lets see the generated output:
```bash
helm template .
```
Now install the chart
```bash
cd ~
helm install bare-chart my-app/ --values my-app/values.yaml
helm list -A
helm status new-chart
kubectl get pod
```

For more on templating, see https://helm.sh/docs/chart_template_guide/getting_started/


### debugging

When you run the `helm template .`  command, you may recieve an error. You can add the `--debug`  argument and the output will show what yaml files in the templated folder have been processed, and the the error that it was run into with the related yaml file - You can ignore the Go stack trace.

It is important to note that the processor will not process files that start with an underscore(_).

WIP:
`helm template elasticsearch bitnami/elasticsearch -f elasticsearch-values.yaml --debug > errors.yaml` 

[--dry-run]


### pipelines and functions

One of the things that Go/Helm templating can do is provide piping (as in Bash) and functions.

for example, if you used the values.yaml file to define a web-app name, you could make sure that the name was converted into lower case:

`{{ .Values.webappname | lower }}`

for a full function list for Helm see https://helm.sh/docs/chart_template_guide/function_list/

Their is also 'if' functionality (prefix -notation):

`{{if eq .Values.environment "dev"}}-dev{{ end }}`


### Named/partial templated

Files that are stored in the template folder that start with underscore, and end with .tpl. (EG _my-template-1.tpl). 

In this file, you'll define a Go template block, 'define' with the piece of template that you want to reuse in the regular yaml templated files.

{{ define "my-temp1" }}

name: nginx imagine
{{ end }}


In the regular template file, use {{ include . }}  (or the less functional {{ template . }} )
The dot is the hierachal level in the values file to use. (see Helm Docs). You'll also need a indent function, since Go templating does not care with in the line you put the 'include'. 


{{ include "my-temp1" . | indent 6 }}


***pro tip*** use the dash to remove any white space {{-  <cmd>}}


### Update a Chart, and install

Lets make a change to the chart, and then push it to K8s


### Static Analysis

`curl https://get.datree.io | /bin/bash` 
