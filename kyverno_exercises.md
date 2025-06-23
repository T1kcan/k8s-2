
# Kyverno Hands-On Exercises

This document guides you through installing Kyverno on Kubernetes and practicing three core policy types: validation, mutation, and generate.

---

## Prerequisites

- Kubernetes cluster (v1.19+ recommended)
- `kubectl` configured for your cluster

---

# Part 1: Install Kyverno

Install Kyverno using the official manifest:

```bash
kubectl create -f https://github.com/kyverno/kyverno/releases/download/v1.14.0/install.yaml
```

Verify Kyverno pods are running:

```bash
kubectl get ns
kubectl get pods -n kyverno
```

Expected output:

```
NAME                        READY   STATUS    RESTARTS   AGE
kyverno-xxxxxx-yyyyy        1/1     Running   0          1m
```

---

# Part 2: Validation Policy — Require `app` Label on Pods

Create the validation policy file `require-app-label.yaml`:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-app-label
spec:
  validationFailureAction: enforce
  rules:
  - name: check-app-label
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "The label 'app' is required."
      pattern:
        metadata:
          labels:
            app: "?*"
```
<!-- ```bash
kubectl create -f- << EOF
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  rules:
  - name: check-team
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      failureAction: Enforce
      message: "label 'team' is required"
      pattern:
        metadata:
          labels:
            team: "?*"
EOF
``` -->
Apply the policy:

```bash
kubectl apply -f require-app-label.yaml
```

### Test Validation

Try to create a pod **without** the `app` label (should fail):

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-invalid
spec:
  containers:
  - name: nginx
    image: nginx
EOF
```

Expected error:

```
Error from server: error when creating "STDIN": admission webhook "validate.kyverno.svc" denied the request: 
policy require-app-label: check-app-label validation failed: The label 'app' is required.
```

Create a pod **with** the `app` label (should succeed):

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-valid
  labels:
    app: myapp
spec:
  containers:
  - name: nginx
    image: nginx
EOF
```
```bash
kubectl get pod --show-labels
```
Run the following command to retrieve the Policy Report that Kyverno just created. Run `describe` command to get further information:
```bash
kubectl get policyreport -o wide
kubectl describe policyreport
```
Notice that there is a single Policy Report with just one result listed under the “PASS” column. This result is due to the Pod we just created having passed the policy.

```bash
kubectl delete -f require-app-label.yaml
#or
kubectl delete clusterpolicy require-labels
```
---

# Part 3: Mutation Policy — Auto-add `env: dev` Label on Pods

Create the mutation policy file `add-env-label.yaml`:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-env-label
spec:
  rules:
  - name: add-env-label-to-pods
    match:
      resources:
        kinds:
        - Pod
    mutate:
      patchStrategicMerge:
        metadata:
          labels:
            env: dev
```

Apply the mutation policy:

```bash
kubectl apply -f add-env-label.yaml
```

### Test Mutation

Create a pod **without** the `env` label:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: mutated-pod
spec:
  containers:
  - name: nginx
    image: nginx
EOF

kubectl run redis --image redis
```

Verify that the `env: dev` label was added automatically:

```bash
kubectl get pod mutated-pod -o jsonpath='{.metadata.labels}'
```

Expected output:

```json
{"env":"dev"}
```
```bash
kubectl get pod --show-labels
```
Expected output:
```bash
controlplane ~ ➜  kubectl get pod --show-labels
NAME             READY   STATUS    RESTARTS   AGE   LABELS
mutated-pod      1/1     Running   0          20m   env=dev
redis            1/1     Running   0          57s   env=dev,run=redis
test-pod-valid   1/1     Running   0          22m   app=myapp
```
Let’s now create a new Pod which already have the desired label defined.
```bash
kubectl run newredis --image redis -l env=prod
kubectl get pod newredis --show-labels
```
This time, you should see Kyverno did not add the team label with the value defined in the policy since one was already found on the Pod.

Now that you’ve experienced mutate policies and seen how logic can be written easily, clean up by deleting the policy you created above.
```bash
kubectl get clusterpolicy
kubectl delete clusterpolicy add-env-label
```
---
# Part 4: Generate Policy — Auto-create Secret in New Namespace
We will use a Kyverno generate policy to generate an image pull secret in a new Namespace.

First, create this Kubernetes Secret in your cluster which will simulate a real image pull secret.
```bash
kubectl -n default create secret docker-registry regcred \
  --docker-server=harbor.aistuido.com.tr \
  --docker-username=tamer.bincan \
  --docker-password=Passw0rd123! \
  --docker-email=tamer.bincan@aistudio.com.tr
```
By default, Kyverno is configured with minimal permissions and does not have access to security sensitive resources like Secrets. You can provide additional permissions using cluster role aggregation. The following role permits the Kyverno background-controller to create (clone) secrets.
```bash
kubectl apply -f- << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kyverno:secrets:view
  labels:
    rbac.kyverno.io/aggregate-to-admission-controller: "true"
    rbac.kyverno.io/aggregate-to-reports-controller: "true"
    rbac.kyverno.io/aggregate-to-background-controller: "true"
rules:
- apiGroups:
  - ''
  resources:
  - secrets
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kyverno:secrets:manage
  labels:
    rbac.kyverno.io/aggregate-to-background-controller: "true"
rules:
- apiGroups:
  - ''
  resources:
  - secrets
  verbs:
  - create
  - update
  - delete
EOF
```
Next, create the following Kyverno policy. The sync-secrets policy will match on any newly-created Namespace and will clone the Secret we just created earlier into that new Namespace.

```bash
kubectl create -f- << EOF
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: sync-secrets
spec:
  rules:
  - name: sync-image-pull-secret
    match:
      any:
      - resources:
          kinds:
          - Namespace
    generate:
      apiVersion: v1
      kind: Secret
      name: regcred
      namespace: "{{request.object.metadata.name}}"
      synchronize: true
      clone:
        namespace: default
        name: regcred
EOF
```
Create a new Namespace to test the policy.
```bash
kubectl create ns mytestns
```
Get the Secrets in this new Namespace and see if regcred is present.
```bash
kubectl -n mytestns get secret
kubectl describe secret -n mytestns regcred
```
You should see that Kyverno has generated the regcred Secret using the source Secret from the default Namespace as the template. If you wish, you may also modify the source Secret and watch as Kyverno synchronizes those changes down to wherever it has generated it.

With a basic understanding of generate policies, clean up by deleting the policy you created above.
```bash
kubectl delete clusterpolicy sync-secrets
```
# Part 5: Generate Policy — Auto-create ConfigMap in New Namespace

Create the generate policy file `generate-configmap.yaml`:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: generate-configmap-on-namespace
spec:
  rules:
  - name: create-configmap
    match:
      resources:
        kinds:
        - Namespace
    generate:
      apiVersion: v1
      kind: ConfigMap
      name: example-config
      namespace: "{{request.object.metadata.name}}"
      data:
        example.property: "This config was auto-generated"
```

Apply the generate policy:

```bash
kubectl apply -f generate-configmap.yaml
```

### Test Generate

Create a new namespace:

```bash
kubectl create namespace test-namespace
```

Check if the ConfigMap was created:

```bash
kubectl get configmap example-config -n test-namespace -o yaml
```

Expected ConfigMap content:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-config
  namespace: test-namespace
data:
  example.property: This config was auto-generated
```

---

# Cleanup (Optional)

```bash
kubectl delete clusterpolicy require-app-label
kubectl delete clusterpolicy add-env-label
kubectl delete clusterpolicy generate-configmap-on-namespace
kubectl delete clusterpolicy sync-secrets
kubectl delete namespace test-namespace mytestns
kubectl delete -f https://github.com/kyverno/kyverno/releases/download/v1.13.0/install.yaml
```


