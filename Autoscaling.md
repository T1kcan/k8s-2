# Scaling

- tutum-rs.yaml
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: tutum-rs
  labels:
    app: tutum
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tutum
  template:
    metadata:
      labels:
        app: tutum
    spec:
      containers:
      - name: tutum
        image: tutum/hello-world
        ports:
        - containerPort: 80
```

## Manual Scaling
To manually scale a ReplicaSet up or down, use the scale command. Scale the tutum pods down to 2 with the command:
```bash
kubectl scale rs tutum-rs --replicas=2
```
You can verify that 3 of the 5 tutum instances have been terminated:
```bash
kubectl get pods
```
or watch them until they finish
```bash
kubectl get po --watch
```
Of course, the ideal way to do this is to update our Manifest to reflect these changes.

## AutoScaling
Kubernetes provides native autoscaling of your Pods. However, kube-scheduler might not be able to schedule additional Pods if your cluster is under high load. In addition, if you have a limited set of compute resources, autoscaling Pods can have severe consequences, unless your worker nodes can automatically scale as well (e.g. AWS autoscaling groups).

- hpa-tutum.yaml
```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-scaler
spec:
  scaleTargetRef:
    kind: ReplicaSet
    name: tutum-rs
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```
To see all the autoscale options:
```bash
kubectl autoscale --help
```
It is also possible to automatically generate a config file, which we've seen before. The command to output a YAML config looks like this:
```bash
kubectl autoscale rs tutum-rs --max=10 --min=3 --cpu-percent=50 --dry-run=true -o=yaml
```
Note --dry-run=true , this means that Kubernetes will not apply the desired state changes to our cluster. However, we provided it with -o=yaml , which means output the configuration as YAML. This lets us easily generate a Manifest.

Tip --dry-run with -o=yaml is an excellent way to generate configurations!

We've provided this content in hpa-tutum.yaml .

Now actually apply the configuration: kubectl create -f .hpa-tutum.yaml

At this point, we have a ReplicaSet managing the Tutum Pods, with Horizontal Pod Autoscaling configured. Let's clean up our environment:
```bash
kubectl delete -f hpa-tutum.yaml

kubectl delete -f tutum-rs.yaml
```

## Deployments on the CLI
Kubernetes Deployments can be created on the command line with kubectl run . It enables you to configure both the Pods and ReplicaSets.

kubectl run NAME --image=image
   --port=port]
  [--env="key=value"]
  [--replicas=replicas]
  [--dry-run=bool]
  [--overrides=inline-json]
  [--command]
  -- [COMMAND] [args...]
To create a simple Kubernetes deployment from the command-line:
```bash
kubectl create deployment tutum --image=tutum/hello-world --port 80
```
Congrats, you have just created your first Deployment. The run command created a Deplyment which automatically performed a few things for you:

- it searched for a suitable node to run the pod
- it scheduled the pod to run on that Node
- it configured the cluster to restart / reschedule the pod when needed
Basically, it created all of the objects we defined, which include Pods and ReplicaSets. It scheduled the Pods on a node capable of accepting workloads.

Let's think back, what is the difference between this command, and how we create Pods on the CLI?

--restart=Never

To verify that the command created a Deployment:
```bash
kubectl get deployments
```
To see the Pods created by the Deployment:
```bash
kubectl get pods
```
To see the ReplicaSet created by the Deployment:
```bash
kubectl get replicasets
```
We can also get more information about our Deployment:
```bash
kubectl describe deployment tutum
```
## The magic of Deployments
If a pod that was created by a Deployment should ever crash, the Deployment will automatically restart it. To see this in action, kill the Pod directly:
```bash
kubectl delete pod $(kubectl get pods --no-headers=true | awk '{print $1;}')
```
The pod should be deleted successfully. Now wait a moment or two and check the pod again:
```bash
kubectl get pods
```
Notice the the pod is running again. This is because the Deployment will restart a pod when it fails. What actually restarts those Pods?

Let's quickly clean up and delete our Deployment: kubectl delete deployment tutum

## Deployment Manifests
The most effective and repeatable way to manage our Deployments is with Manifest files. Here is one that defines our simple Tutum application (tutum-simple.yaml ):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tutum-deployment
spec:
  template:
    spec:
      containers:
      - name: tutum
        image: tutum/hello-world
        ports:
        - containerPort: 80
```
If we look at this Deployment, it looks very similar to our PodSpec and RS Manifests. We can add any configuration that we've already covered in the Pod section to this manifest. We should also configure the ReplicaSet to match our replication requirements.

Let's create our Deployment: kubectl create -f tutum-simple.yaml

All Kubernetes deployment YAML files must contain the following specifications:

apiVersion - apps/v1

The apiVersion specifies the version of the API to use. The API objects are defined in groups. The deployment object belongs to the apps API group. Group objects can be declared alpha , beta , or stable :

alpha - may contain bugs and no guarantee that it will work in the future. Example: (object)/v1alpha1
beta - still somewhat unstable, but will most likely go into the Kubernetes main APIs. Example: (object)/v1beta1
stable - Only stable versions are recommended to be used in production systems. Example: apps/v1
NOTE: You can check the latest Kubernetes API version here: https://kubernetes.io/docs/reference/

kind - Deployment
A kind value declares the type of Kubernetes object to be described in the Yaml file. Kubernetes supports the followng 'kind' objects:

componentstatuses
configmaps
daemonsets
Deployment
events
endpoints
horizontalpodautoscalers
ingress
jobs
limitranges
Namespace
nodes
pods
persistentvolumes
persistentvolumeclaims
resourcequotas
replicasets
replicationcontrollers
serviceaccounts
services

metadata

The metadata declares additional data to uniquely identify a Kubernetes object. The key metadata that can be added to an object:

labels - size constrained key-value pairs used internally by k8s for selecting objects based on identifying information
name - the name of the object (in this case the name of the deployment)
namespace - the name of the namespace to create the object (deployment)
annotations - large unstructured key-value pairs used to provide non-identifying information for objects. k8s cannot query on annotations.
spec - the desired state and characteristics of the object. spec has three important subfields:
replicas : the numbers of pods to run in the deployment
selector : the pod labels that must match to manage the deployment
template : defines each pod (containers, ports, etc. )

## Interacting with Deployments
Now that we have created our Tutum Deployment, let's see what we've got:
```bash
kubectl get deployments

kubectl get rs

kubectl get pods
```
Managing all of these objects by creating a Deployment is powerful because it lets us abstract away the Pods and ReplicaSets. We still need to configure them, but that gets easier with time.

Let's see what happened when we created our Deployment:
```bash
kubectl describe deployment tutum-deployment
```
We can see that, in the events for the Deployment, a ReplicaSet was created. Let's see what the ReplicaSet with kubectl describe rs . Your ReplicaSet has a unique name, so you'll need to tab-complete.

When we look at the ReplicaSet events, we can see that it is creating Pods.

When we look at our Pods with kubectl describe pod , we'll see that the host pulled the image, and started the container.

Deployments can be updated on the command line with set . Here's an example:
```bash
kubectl set image deployment/tutum-deployment tutum=nginx:alpine --record
```
Remember, Kubernetes lets you do whatever you want, even if it doesn't make sense. In this case, we updated the image tutum to be nginx , which is allowed, although strange.

Let's see what happened now that we upated the image to something strange:
```bash
kubectl describe deployment tutum-deployment
```
We can see the history, which includes scaling up and down ReplicaSets for Pods from our command. We can also view revision history:
```bash
kubectl rollout history deployment tutum-deployment
```
We didn't specify any reason for our updates, so CHANGE-CLAUSE is empty. We can also update other configuration options, such as environment variables:
```bash
kubectl set env deployment/tutum-deployment username=admin
```
How do we view those updated environment variables?

Let's get the name of the Pod
We need to exec env inside the Pod
You can also update resources , selector , serviceaccount and subject .

For now, let's simply delete our Deployment:
```bash
kubectl delete -f ./resources/tutum-simple.yaml
```

## Configuring Tutum
Now that we've covered a simple Deployment, let's walk through and create a fully featured one. We will deploy Tutum with additional configuration options.

First, let's all start off with the same file: touch ./resources/tutum-full.yaml

Now, let's populate this file with the base content for a Deployment:

apiVersion: 
kind: 
metadata: 
spec: 
What values should we put in for each of these?

apiVersion: apps/v1
kind: Deployment
metadata ? We need to apply a name and labels , let's say app=tutum
spec is a complex component, where we need to configure our RS and Pod
We should have something similar to shis:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tutum-deployment
  labels:
    app: tutum
spec:
```
Next, let's add the scaffolding required to configure the RS and Pods:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tutum-deployment
  labels:
    app: tutum
spec:
  # RS config goes here
  template:
    metadata:

    spec:
      containers:
```
Now that we've got the scaffolding, let's configure the RS

We need to set the number of replicas
We need to configure a selector to matchLabels
Let's stick with 3 replicas. Remember, we need the RS To match labels on the Pod.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tutum-deployment
  labels:
    app: tutum
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tutum
  template:
    metadata:

    spec:
      containers:
```
Now we need to give the Pods the appropriate labels so the RS will match with it. In addition, let's configure the containers.

We need to apply the label app=tutum to the Pod
We need a single container, let's call it tutum
We need to specify the image, which is tutum/hello-world
The Pod needs to expose port 80
This configuration leads us to create this file (or similar):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tutum-deployment
  labels:
    app: tutum
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tutum
  template:
    metadata:
      labels:
        app: tutum
    spec:
      containers:
      - name: tutum
        image: tutum/hello-world
        ports:
        - containerPort: 80
```
Now that we've got the application configured, we can always add additional PodSpec like what we did in the previous Pod lab. For now, let's deploy the application:
```bash
kubectl create -f ./resources/tutum-full.yaml
```
Who can help me validate that the Deployment was successful?

Check Pod status
Make a request against the webserver
At this point, you can make Deployments as simple or advanced as you want.

## Configuring NGiNX
Now that we've covered Manifest files, its time to tackle some real applications.

Form groups of 2-3 people, and you will put together a Manifest file to help us configure an application. Here's what you need to do:

Create a Manifest file: touch ./resources/nginx-self.yaml
Fill in the required configuration information to run your Deployment
Run the Deployment and make sure it works properly
Here is what you'll need to put in your Pod (in addition to other requirements, like apiVersion):

Name the Deployment
Configure labels for the Deployment
Have the RS maintain 5 Pods
Use the nginx-alpine image
Listen on port 80
Configure environment variables user=admin, password=root, host=katacoda
Configure resource limits: 1 CPU core, 256 MB of RAM
Once you've created the Manifest file, save it, and create the Deployment: kubectl create -f ./resources/nginx-self.yaml

Next, to prove it is working correctly, open up a shell and view the environment variables, and CURL the welcome page from your host. After that, get the logs to make sure everything is working properly. Finally, open up a shell on the Pod, and find out what processes are running inside.

HINT: you might need to install additional software.

Bonus Task: update the image used by the Pod to alpine:latest , apply the configuration change, what happens?

Configuring Minecraft

Next, we will deploy software many of probably has heard of, the Minecraft server. The Minecraft server is free to run, and many people have made businesses out of hosting other people's servers. We'll do this with Kubernetes.

The configuration is relatively simple. Create your base file: touch ./resources/minecraft.yaml

Now, you need to configure the following values, in addition to everything else necessary to create a Deployment:

replicase = 1
image = itzg/minecraft-server
environment variables: EULA="true"
container port = 25565
volume: Pod local = /data , use an emptyDir for the actual storage
There are many more scaffolding requirements for this Deployment, such as apiVersion . Refer back to your notes, and you may need to check out what you've previously done in the Pod lab. You can find old files that you've previously worked on in the /old/ directory on this host.

Once you've deployed it, it should be pretty easy to verify that everything is working correctly.

Deploying applications is really easy with Kubernetes. If any of you have softare running on a server in your home, I can almost guarantee someone is currently maintaining a Deployment Manifest for it on GitHub.

## Rolling Updates and Rollbacks
Now that we've gotten a good taste of creating our own Deployments, its time to use the rolling update and rollback features.

First, let's all start off with a fully configured NGiNX Deployment, located at ./resources/nginx.yaml

Here are the file contents:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 5
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```
For our ReplicaSet, we can configure a strategy that defines how to safely perform a rolling update.
```yaml
strategy:
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 1
  type: RollingUpdate
```
This strategy utilizes Rolling Updates. With rolling updates, Kubernetes will spin up a new Pod, and when it is ready, tear down an old Pod. The maxSurge refers to the total number of Pods that can be active at any given time. If maxSurge = 6 and replicas = 5, that means 1 new Pod (6 - 5) can be created at a time for the rolling update. maxUnavailable is the total number (or percentage) of Pods that can be unavailable at a time.

Here is what our Manifest looks like after integrating this:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 5
  selector:
    matchLabels:
      app: nginx
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```
Now, let's apply this configuration change: kubectl apply -f ./resources/nginx.yaml

Now that the application is deployed, lets update the Manifest to use a different image: nginx:alpine . Now apply the changes.
```bash
kubectl get pods --watch
```
We can see that the Pods are being updated one at a time. If we look at the Deployment events, we can see this as well:
```bash
kubectl describe deployment nginx-deployment

Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  1m    deployment-controller  Scaled up replica set nginx-deployment-555958bc44 to 1
  Normal  ScalingReplicaSet  1m    deployment-controller  Scaled down replica set nginx-deployment-67594d6bf6 to 3
  Normal  ScalingReplicaSet  1m    deployment-controller  Scaled up replica set nginx-deployment-555958bc44 to 2
  Normal  ScalingReplicaSet  1m    deployment-controller  Scaled down replica set nginx-deployment-67594d6bf6 to 2
  Normal  ScalingReplicaSet  1m    deployment-controller  Scaled up replica set nginx-deployment-555958bc44 to 3
  Normal  ScalingReplicaSet  1m    deployment-controller  Scaled down replica set nginx-deployment-67594d6bf6 to 1
  Normal  ScalingReplicaSet  1m    deployment-controller  Scaled up replica set nginx-deployment-555958bc44 to 4
  Normal  ScalingReplicaSet  1m    deployment-controller  Scaled down replica set nginx-deployment-67594d6bf6 to 0
We can see that the Deployment scaled up RS for the new Pods, and then scaled down the old RS. These actions were done one at a time, as specified by our RollingUpdate configuration.
```
We can now get our Deployment rollout history:
```bash
kubectl rollout history deployment/nginx-deployment
```
We can jump back a version:
```bash
kubectl rollout undo deployment.v1.apps/nginx-deployment
```
Or we can jump back to a specific version:
```bash
kubectl rollout undo deployment.v1.apps/nginx-deployment --to-revision=X , where X is the version you want to rollback to
```

