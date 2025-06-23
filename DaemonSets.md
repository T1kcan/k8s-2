# Manage DaemonSets
Overview
A DaemonSet ensures that all (or some) Nodes run a copy of a Pod. As nodes are added to the cluster, Pods are added to them. As nodes are removed from the cluster, those Pods are evicted. Deleting a DaemonSet will clean up the Pods it created.

Some typical uses of a DaemonSet are:

- running a cluster storage daemon, such as glusterd and ceph , on each node.
- running a logs collection daemon on every node, such as fluentd or logstash .
- running a node monitoring daemon on every node, such as Prometheus Node Exporter (node_exporter ), collectd , Datadog agent, New Relic agent, or Ganglia gmond .

In a simple case, one DaemonSet, covering all nodes, would be used for each type of daemon. A more complex setup might use multiple DaemonSets for a single type of daemon, but with different flags and/or different memory and cpu constraints for different hardware types.

## Create a DaemonSet
In this scenario, we're going to create an nginx DaemonSet. Initially, we'll run this on our worker nodes (node01), but then we will manipulate the DaemonSet to get it to run on the master node too.

### nginx DaemonSet

First, let's create all the prerequisites needed for this DaemonSet to run:

- nginx-ds-prereqs.yaml 
```yaml
---
kind: Namespace
apiVersion: v1
metadata:
  name: development
  labels:
    name: development

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-svc-acct
  namespace: development
  labels:
    name: nginx-svc-acct
```

```bash
kubectl create -f nginx-ds-prereqs.yaml
```
Now we've created the namespace (and other prerequisites), let's inspect the manifest for the nginx DaemonSet:

- nginx-daemonset.yaml 
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx
  namespace: development
  labels:
    app: nginx
    name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
      name: nginx
  template:
    metadata:
      labels:
        app: nginx
        name: nginx
    spec:
      serviceAccountName: nginx-svc-acct
      containers:
      - image: katacoda/docker-http-server:latest
        name: nginx
        ports:
        - name: http
          containerPort: 80
```

As you can see, we're running a basic DaemonSet - in the development namespace - which exposes port 80 inside the container.

Create it:
```bash
kubectl create -f nginx-daemonset.yaml
```
Now check the status of the DaemonSet:
```bash
kubectl get daemonsets -n development
```

## Accessing nginx
Now that we've created our nginx DaemonSet, let's see what host it's running on:
```bash
kubectl get po -n development -l app=nginx -o 'jsonpath={.items[0].spec.nodeName}'; echo
```
Notice that it's running on node01 and not master . By default, Kubernetes won't schedule any workload to master nodes unless they're tainted. Essentially, what this means is that workload has to be specifically set to be ran on master nodes.

It's not best practice to run 'normal' workload on master nodes, as it's where etcd and other crucial Kubernetes components reside. However, it's acceptable to run workload such as log collection and node monitoring daemons on master nodes as you want to understand what's happening on those nodes.

### Testing the Webserver
We want to get the IP address for the pod so we can test that it's working, now that we know which node it's running on:
```bash
kubectl get po -n development -l app=nginx -o 'jsonpath={.items[0].status.podIP}'; echo
```
Curl it:
```bash
curl `kubectl get po -n development -l app=nginx -o 'jsonpath={.items[0].status.podIP}'`
```
You should see a result like:
```text
This request was processed by host: nginx-8n2qj
```

### Updating a DaemonSet (Rolling Update)
As mentioned in the previous chapter, workload isn't scheduled to master nodes unless specificaly tainted. In this scenario, we want to run the nginx DaemonSet across both master and node01 .

We need to update the DaemonSet, so we're going to use the nginx-daemonset-tolerations.yaml file to replace the manifest.

First, let's see what we added to the -tolerations.yaml file:
```bash
cat nginx-daemonset-tolerations.yaml; echo
```
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx
  namespace: development
  labels:
    app: nginx
    name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
        name: nginx
    spec:
      serviceAccountName: nginx-svc-acct
      containers:
      - image: katacoda/docker-http-server:latest
        name: nginx
        ports:
        - name: http
          containerPort: 80
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
```
As you can see, we've added the following to the spec section:
```yaml
tolerations:
- key: node-role.kubernetes.io/master
  effect: NoSchedule
```
This is what manifests need to be tainted with in order to be ran on master nodes. Proceed to update the DaemonSet:
```bash
kubectl replace -f nginx-daemonset-tolerations.yaml
```
Now check to see if an additional pod has been created. Remember - a DaemonSet schedules a pod to every node, so there should be two pods created:

kubectl get po -n development -l app=nginx -o wide

If there's two pods - great. That means that the tolerations have worked and we are now running across two nodes.

Accessing the pod on the master node
Find the pod IP address for the newly created pod on the master node:
```bash
kubectl get po -n development -l app=nginx -o 'jsonpath={.items[1].status.podIP}'; echo
```
Notice that it's different to the IP address that we curl'ed before.

Now curl the new pod's IP address:
```bash
curl `kubectl get po -n development -l app=nginx -o 'jsonpath={.items[1].status.podIP}'`
```
You should see a similar result to the one in the previous chapter:

This request was processed by host: nginx-njq9h

## Deleting a DaemonSet
Clean up our workspace:
```bash
kubectl delete daemonset nginx -n development
```
Alternatively, we could use the short hand, which achieves the same result:
```bash
kubectl delete ds nginx -n development
```
Success - you've deleted the DaemonSet. Check for pods:
```bash
kubectl get pods -n development
```
Great! You're all done.