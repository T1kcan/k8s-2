# Jobs, Init Containers and Cron Jobs
This scenario provides an overview of Jobs, Init Containers and CronJobs in Kubernetes.

Jobs resources create one or more pods and ensures that all of them successfully terminate.

There are two types of jobs:

- Non-parallel Job: A Job which creates only one Pod (which is re-created if the Pod terminates unsuccessfully), and which is completed when the Pod terminates successfully.
- Parallel jobs with a completion count: A Job that is completed when a certain number of Pods terminate successfully. You specify the desired number of completions using the completions field.
Cron Jobs create a job object, they are useful for creating periodic and recurring tasks, e.g running backups or sending emails.

Init Containers are regular containers within a pod that run before the app container and they also satisfy the following statements:

- They can run setup scripts not present in an app container - e.g prepopulate some data, waiting until a specific service is up and running and etc.
- A pod can have one or more init containers apart from app containers
- Init containers always run to completation
- Each one must complete successfully before the next one is started
- The application container won't run if any of the configured init containers will not finish the execution successfully

# Jobs
Job resources are used to facilitate the execution of a batch job. Through Job resources, Kubernetes also supports parallel jobs which will finish executing when a specific number of successful completions is reached.

Therefore with Job resources, we can run work items such as frames to be rendered, files to be transcoded, ranges of keys in a NoSQL database to scan, and so on.

Have a look at Jobs Api reference to see how to build a job resource in Kubernetes.

Pods created by jobs are not automatically deleted. Keeping the pods around allows you to view the logs of completed jobs in order to check for potential errors. If you want to remove them, you need to do that manually.

## Create Countdown Job
Take a look at the file job.yaml .

This example creates a job which runs a bash command to count down from 10 to 1.

Notice that the field spec.restartPolicy allow only two values: "OnFailure" or "Never". For further information read here

Note: There are situations where you want to fail a job after a number of retries. To do so, use spec.backoffLimit which, by defauly, is set 6. You can use spec.activeDeadlineSeconds to limit the execution time in case you want to manage the duration of a specific job. If the execution reaches this deadline, the Job and all of its Pods are terminated.
Create the countdown job:
- job.yaml
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: countdown
spec:
  template:
    spec:
      containers:
      - name: countdown
        image: bash
        command: ["/bin/sh",  "-c"]
        args: 
          - for i in 9 8 7 6 5 4 3 2 1 ; do echo $i ; done &&
            echo Perfect! 
      restartPolicy: OnFailure
```

```bash
kubectl apply -f /manifests/job.yaml
```
Job status
Check the status of the job:
```bash
kubectl get jobs
```
## Job Logs
In order to see the job's logs we need to get the name of the Job in question:
```bash
kubectl get pods -o 'jsonpath={.items[0].metadata.name}'; echo
```
And then execute the following command to get the logs:
```bash
kubectl logs `kubectl get pods -o 'jsonpath={.items[0].metadata.name}'` 
```
You will get a result like:
```bash
9 8 7 6 5 4 3 2 1 Perfect!
```
Delete Job
```bash
kubectl delete -f /manifests/job.yaml

or

kubectl delete job countdown
```

## Parallel Jobs
To create a parallel job we can use spec.parallelism to set how many pods we want to run in parallel and spec.completions to set how many job completitions we would like to achieve.

Create Countdown Parallel Job
Inspect the file jobs-parallels.yaml .

This is the same countdown job we used in the previous scenario but we have added spec.parallelism and spec.completions parameters.

The job will run 2 pods in parallel until it reaches 8 completions successfully.

Create countdown parallel job:
- jobs-parallels.yaml 
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: countdown
spec:
  completions: 8
  parallelism: 2
  template:
    spec:
      containers:
      - name: countdown
        image: bash
        command: ["/bin/sh",  "-c"]
        args: 
          - for i in 9 8 7 6 5 4 3 2 1 ; do echo $i ; done &&
            echo Perfect! 
      restartPolicy: OnFailure`
```

```bash
kubectl apply -f /manifests/jobs-parallels.yaml
```
## Job status
Wait for a few seconds to get the 8 completions and then check the status of the job:
```bash
kubectl get jobs
```
You should see a similar result to the following, but if not, wait for a few more seconds and check again:

NAME DESIRED SUCCESSFUL AGE countdown 8 8 16s

This job was executed successfully 8 time by running 2 jobs in parallel.

## Job Logs
In order to see the job's logs, we need to get the job name:
```bash
kubectl get pods -o 'jsonpath={.items[0].metadata.name}'; echo
```
And then execute the following command to get the logs:
```bash
kubectl logs `kubectl get pods -o 'jsonpath={.items[0].metadata.name}'`
```
You will get a result like:

9 8 7 6 5 4 3 2 1 Perfect!

Delete Job
```bash
kubectl delete -f /manifests/jobs-parallels.yaml

or

kubectl delete job countdown
```

## Cron Jobs
Written in a Cron format, a Cron Job resource runs a job periodically on a given schedule. These are useful for creating periodic and recurring tasks, e.g running backups or sending emails.

Create Hello Cron Job
Take a look at the file cronjob.yaml . This example create a job every minute which prints the current time and a hello message.

- cronjob.yaml 
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```
```bash
kubectl apply -f /manifests/cronjob.yaml
```
Cron Job status
Check the status of the cronjob:
```bash
kubectl get cronjob hello
```
Immediatly after creating a cron job, the LAST-SCHEDULE column will have no value (<none> ). This indicates that the CronJob hasn't run yet.

master $ kubectl get cronjob hello NAME SCHEDULE SUSPEND ACTIVE LAST SCHEDULE AGE hello */1 * * * * False 0 <none> 8s


Once the LAST-SCHEDULE column gets a value, it indicates that the CronJobs is now scheduled to run:
```bash
kubectl get cronjob --watch
```
Check the cron job again, you should see that the cronjob has been scheduled at the time specified in LAST-SCHEDULE :
```bash
kubectl get cronjob hello
```
Cron Job Logs
In order to see the job's logs, we need to know the pod's name:
```bash
kubectl get pod -o 'jsonpath={.items[0].metadata.name}'; echo
```
And then:
```bash
kubectl logs `kubectl get pod -o 'jsonpath={.items[0].metadata.name}'`
```
Delete Cron Job
```bash
kubectl delete cronjob hello
```

## Init Container
An Init Container is a container which is executed before the application container is started. Init-containers are usually used for deploying utilities or execute scripts which are not loaded and executed in the application container image.

Create a Pod with an init container
Take a look at the file init-container.yaml .

This example runs an init-container which creates a helloworld file in a volume. The application pod will be scheduled if the helloworld file exist at a specific path and the pod can access it.

Create the init container:
- init-container.yaml 
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: happypanda
spec:
  containers:
  - name: busybox
    image: busybox:latest
    command: ["/bin/sh", "-c"]
    args: ["cat /opt/workdir/helloworld && sleep 3600"]
    volumeMounts:
    - name: workdir
      mountPath: /opt/workdir
  initContainers:
  - name: init-container
    image: busybox
    command:
            - sh
            - -c
            - 'echo "The app is running" > /opt/workdir/helloworld'
    volumeMounts:
    - mountPath: /opt/workdir
      name: workdir
  volumes:
  - name: workdir
    emptyDir: {}
```
```bash
kubectl apply -f /manifests/init-container.yaml
```
It could take some time until the Init container finishes the execution successfully and the application container is scheduled to run.

Pod status
The Init container will take some time until it creates the file so you might have to check the status of the pod a couple of times:
```bash
kubectl get pods
```
If the pod is running, it means that the file was created succesfully and the pod can read it. We are going to manually check that the file is at the specified path and it has the correct content:
```bash
kubectl exec -ti happypanda -- cat /opt/workdir/helloworld
```
You should see a result like:

The app is running

Delete Pod
```bash
kubectl delete -f /manifests/init-container.yaml

or

kubectl delete pod happypanda
```