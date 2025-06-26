
# Jenkins Helm-Based Kubernetes Deployment Lab

## 1. Prerequisites

- Kubernetes Cluster
- Helm v3+
- Docker Registry (Docker Hub / Harbor / ECR / etc.)
- Jenkins Dynamic Agent setup
- Sample app with Dockerfile and Helm chart

---

## 1.2 Install Jenkins via Helm

----

Configure Helm
Once Helm is installed and set up properly, add the Jenkins repo as follows:
```bash
helm repo add jenkins https://charts.jenkins.io
helm repo update
```
The helm charts in the Jenkins repo can be listed with the command:
```bash
helm search repo jenkins
```
<!-- Create PV:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-pv
  namespace: jenkins
spec:
  storageClassName: jenkins-pv
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 20Gi
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /data/jenkins-volume/

---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: jenkins-pv
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```
service-account.yaml
```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: jenkins
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: jenkins
rules:
- apiGroups:
  - '*'
  resources:
  - statefulsets
  - services
  - replicationcontrollers
  - replicasets
  - podtemplates
  - podsecuritypolicies
  - pods
  - pods/log
  - pods/exec
  - podpreset
  - poddisruptionbudget
  - persistentvolumes
  - persistentvolumeclaims
  - jobs
  - endpoints
  - deployments
  - deployments/scale
  - daemonsets
  - cronjobs
  - configmaps
  - namespaces
  - events
  - secrets
  verbs:
  - create
  - get
  - watch
  - delete
  - list
  - patch
  - update
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - get
  - list
  - watch
  - update
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: jenkins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:serviceaccounts:jenkins
``` -->
<!-- Create `jenkins` ns:
```bash
kubectl create ns jenkins
```
```bash
kubectl apply -f service-account.yaml
``` -->
Optionally download values.yaml file to edit:
```bash
wget https://raw.githubusercontent.com/jenkinsci/helm-charts/main/charts/jenkins/values.yaml -O values.yaml
```
```bash
helm upgrade --install jenkins jenkins/jenkins \
  --set controller.serviceType=NodePort \
  --set controller.nodePort=32000 \
  --namespace jenkins \
  --create-namespace
```

```text
controlplane:~$ helm uninstall jenkins -n jenkins
release "jenkins" uninstalled
controlplane:~$ vi values.yaml 
controlplane:~$ helm install jenkins -n jenkins -f jenkins-values.yaml jenkinsci/jenkins
Error: INSTALLATION FAILED: repo jenkinsci not found
controlplane:~$ helm install jenkins -n jenkins -f jenkins-values.yaml jenkins/jenkins
Error: INSTALLATION FAILED: open jenkins-values.yaml: no such file or directory
controlplane:~$ helm install jenkins -n jenkins -f values.yaml jenkins/jenkins
Release "jenkins" does not exist. Installing it now.
NAME: jenkins
LAST DEPLOYED: Wed Jun 25 21:21:00 2025
NAMESPACE: jenkins
STATUS: deployed
REVISION: 1
NOTES:
1. Get your 'admin' user password by running:
  kubectl exec --namespace jenkins -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password && echo
2. Get the Jenkins URL to visit by running these commands in the same shell:
  export NODE_PORT=$(kubectl get --namespace jenkins -o jsonpath="{.spec.ports[0].nodePort}" services jenkins)
  export NODE_IP=$(kubectl get nodes --namespace jenkins -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT

3. Login with the password from step 1 and the username: admin
4. Configure security realm and authorization strategy
5. Use Jenkins Configuration as Code by specifying configScripts in your values.yaml file, see documentation: http://$NODE_IP:$NODE_PORT/configuration-as-code and examples: https://github.com/jenkinsci/configuration-as-code-plugin/tree/master/demos

For more information on running Jenkins on Kubernetes, visit:
https://cloud.google.com/solutions/jenkins-on-container-engine

For more information about Jenkins Configuration as Code, visit:
https://jenkins.io/projects/jcasc/


NOTE: Consider using a custom image with pre-installed plugins
#NBcbp96kvHkCOV11D6vmVn
```

#helm install jenkins -n jenkins -f values.yaml jenkins/jenkins

Get Jenkins URL:
```bash
kubectl get svc -n jenkins 
kubectl exec --namespace jenkins -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password && echo
#fntT9b6F7WJqNsUE2IYUFu
export NODE_PORT=$(kubectl get --namespace jenkins -o jsonpath="{.spec.ports[0].nodePort}" services jenkins)
export NODE_IP=$(kubectl get nodes --namespace jenkins -o jsonpath="{.items[0].status.addresses[0].address}")
echo http://$NODE_IP:$NODE_PORT
```



### Run Jenkins on Docker:

```bash
docker run -d \
  --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts
```
Get the admin password
```
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```
- docker-compose.yaml
```yaml
version: '3'
services:
  jenkins:
    image: jenkins/jenkins:lts
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home

volumes:
  jenkins_home:
```
```bash
docker compose up -d
```
## 2. Sample App Repository Structure

```
my-app/
│
├── Dockerfile
├── app.py
├── requirements.txt
└── chart/
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
        └── deployment.yaml
```

```groovy
pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-id') // Set in Jenkins > Credentials
        IMAGE_NAME = 'yourdockerhubusername/yourapp'
    }

    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/your-org/your-repo.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${IMAGE_NAME}:latest")
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-id') {
                        docker.image("${IMAGE_NAME}:latest").push()
                    }
                }
            }
        }
    }
}

```

```groovy
pipeline {
    agent any
    tools {
        terraform 'terraform'
}
    environment {
        PATH=sh(script:"echo $PATH:/usr/local/bin", returnStdout:true).trim()
        AWS_REGION = "us-east-1"
        AWS_ACCOUNT_ID=sh(script:'export PATH="$PATH:/usr/local/bin" && aws sts get-caller-identity --query Account --output text', returnStdout:true).trim()
        ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        APP_REPO_NAME = "clarusway-repo/cw-todo-app"
        APP_NAME = "todo"
        HOME_FOLDER = "/home/ec2-user"
        GIT_FOLDER = sh(script:'echo ${GIT_URL} | sed "s/.*\\///;s/.git$//"', returnStdout:true).trim()
    }
        stages {

        stage('Create Infrastructure for the App') {
            steps {
                echo 'Creating Infrastructure for the App on AWS Cloud'
                sh 'terraform init'
                sh 'terraform apply --auto-approve'
            }
        }
        stage('Create ECR Repo') {
            steps {
                echo 'Creating ECR Repo for App'
                sh """
                aws ecr create-repository \
                  --repository-name ${APP_REPO_NAME} \
                  --image-scanning-configuration scanOnPush=false \
                  --image-tag-mutability MUTABLE \
                  --region ${AWS_REGION}
                """
            }
        }
        stage('Build App Docker Image') {
            steps {
                echo 'Building App Image'
                script {
                    env.NODE_IP = sh(script: 'terraform output -raw node_public_ip', returnStdout:true).trim()
                    env.DB_HOST = sh(script: 'terraform output -raw postgre_private_ip', returnStdout:true).trim()
                }
                sh 'echo ${DB_HOST}'
                sh 'echo ${NODE_IP}'
                sh 'envsubst < node-env-template > ./nodejs/server/.env'
                sh 'cat ./nodejs/server/.env'
                sh 'envsubst < react-env-template > ./react/client/.env'
                sh 'cat ./react/client/.env'
                sh 'docker build --force-rm -t "$ECR_REGISTRY/$APP_REPO_NAME:postgre" -f ./postgresql/dockerfile-postgresql .'
                sh 'docker build --force-rm -t "$ECR_REGISTRY/$APP_REPO_NAME:nodejs" -f ./nodejs/dockerfile-nodejs .'
                sh 'docker build --force-rm -t "$ECR_REGISTRY/$APP_REPO_NAME:react" -f ./react/dockerfile-react .'
                sh 'docker image ls'
            }
        }
        stage('Push Image to ECR Repo') {
            steps {
                echo 'Pushing App Image to ECR Repo'
                sh 'aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin "$ECR_REGISTRY"'
                sh 'docker push "$ECR_REGISTRY/$APP_REPO_NAME:postgre"'
                sh 'docker push "$ECR_REGISTRY/$APP_REPO_NAME:nodejs"'
                sh 'docker push "$ECR_REGISTRY/$APP_REPO_NAME:react"'
            }
        }
        stage('wait the instance') {
            steps {
                script {
                    echo 'Waiting for the instance'
                    id = sh(script: 'aws ec2 describe-instances --filters Name=tag-value,Values=ansible_postgresql Name=instance-state-name,Values=running --query Reservations[*].Instances[*].[InstanceId] --output text',  returnStdout:true).trim()
                    sh 'aws ec2 wait instance-status-ok --instance-ids $id'
                }
            }
        }
        stage('Deploy the App') {
            steps {
                echo 'Deploy the App'
                sh 'ls -l'
                sh 'ansible --version'
                sh 'ansible-inventory --graph'
                ansiblePlaybook credentialsId: 'firstkey', disableHostKeyChecking: true, installation: 'ansible', inventory: 'inventory_aws_ec2.yml', playbook: 'docker_project.yml'
             }
        }
        stage('Destroy the infrastructure'){
            steps{
                timeout(time:5, unit:'DAYS'){
                    input message:'Approve terminate'
                }
                sh """
                docker image prune -af
                terraform destroy --auto-approve
                aws ecr delete-repository \
                  --repository-name ${APP_REPO_NAME} \
                  --region ${AWS_REGION} \
                  --force
                """
            }
        }
    }
    post {
        always {
            echo 'Deleting all local images'
            sh 'docker image prune -af'
        }
        failure {

            echo 'Delete the Image Repository on ECR due to the Failure'
            sh """
                aws ecr delete-repository \
                  --repository-name ${APP_REPO_NAME} \
                  --region ${AWS_REGION}\
                  --force
                """
            echo 'Deleting Terraform Stack due to the Failure'
                sh 'terraform destroy --auto-approve'
        }
    }
}
```


Verify Kubernetes Plugin

Locale
Manage Jenkins > Plugins > Installed
and confirm the Kubernetes plugin is installed.
1.4 Configure Kubernetes Cloud in Jenkins
Go to:
Manage Jenkins > Configure Clouds > Kubernetes

Kubernetes URL: https://kubernetes.default

Namespace: default

Credentials: Use "Kubernetes service account"

Jenkins URL: http://jenkins.default.svc.cluster.local:8080


Define Pod Template (YAML)

```yaml
containers:
- name: jnlp
  image: jenkins/inbound-agent:latest
- name: docker
  image: docker:24
  command:
  - cat
  tty: true
- name: kubectl
  image: bitnami/kubectl:latest
  command:
  - cat
  tty: true
```


## Jenkinsfile Example
```groovy
pipeline {
  agent {
    kubernetes {
      yaml ''' 
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: docker
    image: docker:24
    command:
    - cat
    tty: true
  - name: kubectl
    image: bitnami/kubectl:latest
    command:
    - cat
    tty: true
'''
    }
  }
  stages {
    stage('Build') {
      steps {
        container('docker') {
          sh 'docker --version'
        }
      }
    }
    stage('Deploy') {
      steps {
        container('kubectl') {
          sh 'kubectl version --client'
        }
      }
    }
  }
}
```


Verify Agent Pods
```bash
kubectl get pods -n default
```

---

## Jenkins Helm-Based Kubernetes Deployment Lab

### Sample App Structure

my-app/
│
├── Dockerfile
├── app.py
├── requirements.txt
└── chart/
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
        └── deployment.yaml

### Example app.py

```python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello():
    return "Hello from Jenkins + Helm!"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### Dockerfile
```Dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["python", "app.py"]
```
### chart/values.yaml

```yaml
image:
  repository: your-docker-username/my-app
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 5000
```
---

## 3. Jenkinsfile Example

```groovy
pipeline {
    agent {
        kubernetes {
            yaml ''' 
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: docker
    image: docker:24
    command: ["cat"]
    tty: true
  - name: helm
    image: alpine/helm:3.14.0
    command: ["cat"]
    tty: true
'''
        }
    }
    environment {
        REGISTRY = 'docker.io'
        IMAGE_NAME = 'your-docker-username/my-app'
    }
    stages {
        stage('Build Image') {
            steps {
                container('docker') {
                    sh 'docker build -t $REGISTRY/$IMAGE_NAME:latest .'
                }
            }
        }
        stage('Push Image') {
            steps {
                container('docker') {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin $REGISTRY'
                        sh 'docker push $REGISTRY/$IMAGE_NAME:latest'
                    }
                }
            }
        }
        stage('Deploy with Helm') {
            steps {
                container('helm') {
                    sh 'helm upgrade --install my-app ./chart --set image.repository=$REGISTRY/$IMAGE_NAME --set image.tag=latest'
                }
            }
        }
    }
}
```

---

## 4. Jenkins Credentials

- ID: `dockerhub-creds`
- Type: Username/Password
- DockerHub credentials
---

## 5. Verify Deployment

```bash
kubectl get all
kubectl port-forward svc/my-app 5000:5000
curl http://localhost:5000
```

Expected Output:

```
Hello from Jenkins + Helm!
```

---

## 6. Cleanup

```bash
helm uninstall my-app
```
