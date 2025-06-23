# Assigning Pods to Nodes
## 1. Node Selector

nodeSelector is a selector which allows you to assign a pod to a specific node. It matches a node key/value pair, also known as a Label, which tells the Kubernetes scheduler which node to schedule the pod to.

If the label specified as nodeSelector doesn't exist on any node, then the pod will fail to be scheduled. If we still want to schedule our pod (even though the label doesn't exist on a node), we need to use Node/Pod affinity and Anti-affinity . We will cover and discuss this later in this scenario.

Note: By default, Kubernetes adds labels to nodes such as kubernetes.io/hostname , beta.kubernetes.io/arch and so on. For further information, read Built-in node labels in the Kubernetes documenation.

In the following section we will schedule the happypanda pod to the node01 node by using nodeSelector(disk=ssd).

Discover node labels
First of all, let's look at the current node labels:
```bash
kubectl get nodes --show-labels
```
### Add a new node label
Now, add the label disk=ssd to the node01 node:
```bash
kubectl label nodes node01 disk=ssd
```
Ensure the label we've just created has been added to node01 :
```bash
kubectl get nodes --show-labels
```
Assign the happypanda pod to node01, matching disk:ssd label
Check the file pod-nodeselector.yaml :

- pod-nodeselector.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: happypanda
  labels: 
    app: redis
    segment: backend
    company: mycompany 
    disk: ssd  
spec:
  containers:
  - name: redis
    image: redis
    ports:
    - name: redisport
      containerPort: 6379
      protocol: TCP
  nodeSelector:
    disk: ssd
```

Notice that nodeSelector is matching the label we just added to the node - disk:ssd . The Kubernetes scheduler will use this label to try and schedule pods onto a node with the label.

Next, we will create the happypanda pod by running the following command:
```bash
kubectl apply -f pod-nodeselector.yaml
```
Let's verify
Check that happypanda has been successfully scheduled on the node01 node:
```bash
kubectl get pods -o wide
```
### Deleting happypanda pod and label
Delete happypanda pod:
```bash
kubectl delete -f /manifests/pod-nodeselector.yaml
# or
kubectl delete pod happypanda
```
Remove label from node01:
```bash
kubectl label node node01 disk-
```

## 2. Node Affinity and Anti-Affinity

In the previous chapter, we introduced nodeSelector . This works well if your nodes have the required node labels, but if the nodeSelector doesn't match a label on a node, then the pod will not be scheduled. Node/Pod Affinity and Anti-Affinity resolves this issue by introducing soft and hard conditions.

### 2.1. Node Affinity
When you look at the Kubernetes API Reference, you'll notice that the two specs for Node Affinity are:

- spec.affinity.nodeAffinity.preferredDuringSchedulingIgnoredDuringExecution - Soft NodeAffinity and Anti-Affinity: If the node label exists, the Pod will be ran there. If not, then the Pod will be rescheduled elsewhere within the cluster.

- spec.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution - Hard NodeAffinity and Anti-Affinity: If the node label doesn't exist, then the pod won't be scheduled at all.

#### 2.1.1. Soft Node Affinity
Let's inspect the node-soft-affinity.yaml manifest:

- node-soft-affinity.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: happypanda
  labels: 
    app: redis
    segment: backend
    company: mycompany 
    disk: ssd  
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: fruit
            operator: In
            values:
            - apple
  containers:
  - name: redis
    image: redis
    ports:
    - name: redisport
      containerPort: 6379
      protocol: TCP
```

The manfifest reads as: "If there are no nodes labelled as apple , then still schedule the pod to a node".

#### 2.1.2. Hard Node Affinity
Now inspect the node-hard-affinity.yaml manifest:

- node-hard-affinity.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: happypanda
  labels: 
    app: redis
    segment: backend
    company: mycompany 
    disk: ssd  
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: fruit
            operator: In
            values:
            - apple
  containers:
  - name: redis
    image: redis
    ports:
    - name: redisport
      containerPort: 6379
      protocol: TCP
```

The manifest reads as: "If there are no nodes labelled as apple , then this pod won't be assigned a node by the scheduler".

### 2.2. Node Anti-Affinity
Node anti-affinity can be achieved by using the NotIn operator. This will help us to ignore nodes while scheduling.

For a node anti-affinity check the node-hard-anti-affinity.yaml and node-soft-anti-affinity.yaml manifest files:

- node-hard-anti-affinity.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: happypanda
  labels: 
    app: redis
    segment: backend
    company: mycompany 
    disk: ssd  
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: fruit
            operator: NotIn
            values:
            - apple
  containers:
  - name: redis
    image: redis
    ports:
    - name: redisport
      containerPort: 6379
      protocol: TCP
```

- node-soft-anti-affinity.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: happypanda
  labels: 
    app: redis
    segment: backend
    company: mycompany 
    disk: ssd  
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: fruit
            perator: NotIn
            values:
            - apple
  containers:
  - name: redis
    image: redis
    ports:
    - name: redisport
      containerPort: 6379
      protocol: TCP
```
```bash
kubectl taint node controlplane node-role.kubernetes.io/control-plane:NoSchedule-
kubectl label node node01 fruit=apple
kubectl apply -f node-hard-affinity.yaml
```
NodeAffinity allows you to schedule pods on specific nodes. But what if you want to run multiple pods on specific nodes? Pod affinity helps with that.

## 3. Pod Affinity
When you look at the Kubernetes API Reference, you'll notice that the two specs for Pod Affinity are:

- spec.affinity.podAffinity.preferredDuringSchedulingIgnoredDuringExecution is for Soft Pod Affinity. If the preferred option is available, the Pod will run there. If not, the Pod can still be scheduled elsewhere.

- spec.affinity.podAffinity.requiredDuringSchedulingIgnoredDuringExecution is for Hard Pod Affinity. If the required option is not available, the Pod cannot run.

### 3.1. Hard Pod Affinity
Let's inspect the pod-hard-affinity.yaml file:

- pod-hard-affinity.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: happypanda
  labels: 
    app: redis
    segment: backend
    company: mycompany 
    disk: ssd  
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
            - key: fruit
              operator: In
              values:
              - apple
          topologyKey: kubernetes.io/hostname
  containers:
  - name: redis
    image: redis
    ports:
    - name: redisport
      containerPort: 6379
      protocol: TCP
```

This is a hard pod affinity . If none of the nodes are labelled with fruit=apple, then the pod won't be scheduled.

The topologyKey is a label of a node, such as kubernetes.io/hostname.
--gethappypanda.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gethappypanda
  labels:
    fruit: apple
spec:
  containers:
  - name: redis
    image: redis
    ports:
    - name: redisport
      containerPort: 6379
      protocol: TCP
  nodeSelector:
    cpu: amd
```
```bash
kubectl apply -f gethappypanda.yaml
kubectl get pod -o wide
kubectl describe node controlplane | grep -i taint
kubectl taint node controlplane node-role.kubernetes.io/control-plane:NoSchedule-
kubectl label nodes node01 cpu=amd
kubectl get pod -o wide
```

### 3.2. Soft Pod Affinity
Soft Pod Affinity will schedule the Pod even though is not finding a pod running with label fruit=apple.

## 4. Pod Anti-Affinity
When you look at the Kubernetes API Reference, you'll notice that the two specs for Pod Anti-Affinity are:

- spec.affinity.podAntiAffinity.preferredDuringSchedulingIgnoredDuringExecution is for Soft Pod Anti-Affinity. If the preferred option is available, the Pod will run there. If not, the Pod can still be sheduled elsewhere.

- spec.affinity.podAntiAffinity.requiredDuringSchedulingIgnoredDuringExecution is for Hard Pod Anti-Affinity. If the required option is not available, the Pod cannot run.

Pod anti-affinity works the opposite way of pod affinity. If one of the nodes has a pod running with label fruit=apple, the pod will be scheduled on different node.