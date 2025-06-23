
# 🧪 Prometheus, Node Exporter, and Grafana Hands-On

This document includes two versions of a full monitoring stack deployment:
- [Version 1](#version-1-running-prometheus-node-exporter-and-grafana-using-docker): Docker-based setup
- [Version 2](#version-2-running-prometheus-node-exporter-and-grafana-in-kubernetes): Kubernetes-based setup

---

## ✅ Version 1: Running Prometheus, Node Exporter, and Grafana using Docker

### 🧱 1. Prerequisites

Ensure you have the following installed:

- Docker
- Docker Compose
- A web browser

### 📁 2. Create the Working Directory

```bash
mkdir prometheus-docker
cd prometheus-docker
```

### 📝 3. Create the Prometheus Configuration File

```bash
mkdir prometheus
cat <<EOF > prometheus/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['prometheus:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['node-exporter:9100']
EOF
```

### ⚙️ 4. Create docker-compose.yml

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:v2.53.0
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  node-exporter:
    image: prom/node-exporter:v1.8.0
    ports:
      - "9100:9100"

  grafana:
    image: grafana/grafana:10.4.3
    ports:
      - "3000:3000"
```

### 🚀 5. Start All Services

```bash
docker-compose up -d
```

### 🌐 6. Access the UIs

- Prometheus: http://localhost:9090
- Node Exporter: http://localhost:9100/metrics
- Grafana: http://localhost:3000  
  Login: admin / admin

### 📊 7. Configure Grafana

1. Add a Prometheus data source: `http://prometheus:9090`
2. Import dashboard ID `1860` ("Node Exporter Full")

### 🧹 8. Cleanup

```bash
docker-compose down
rm -rf prometheus-docker
```

---

## 🧩 Version 2: Running Prometheus, Node Exporter, and Grafana in Kubernetes

### 🔧 1. Prerequisites

- Kubernetes cluster
- kubectl
- (Optional) Helm

### 📁 2. Create Namespace

```bash
kubectl create namespace monitoring
```

### 📄 3. Deploy Prometheus & Node Exporter

Create `prometheus.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
      - job_name: prometheus
        static_configs:
          - targets: ['localhost:9090']
      - job_name: node-exporter
        static_configs:
          - targets: ['node-exporter.monitoring.svc.cluster.local:9100']
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:v2.53.0
        args:
          - --config.file=/etc/prometheus/prometheus.yml
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: config-volume
          mountPath: /etc/prometheus/
      volumes:
      - name: config-volume
        configMap:
          name: prometheus-config
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitoring
spec:
  type: NodePort
  selector:
    app: prometheus
  ports:
    - protocol: TCP
      port: 9090
      targetPort: 9090
      nodePort: 30090
```

Create `node-exporter.yaml`:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.8.0
        ports:
        - containerPort: 9100
          hostPort: 9100
          protocol: TCP
        securityContext:
          privileged: true
---
apiVersion: v1
kind: Service
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  type: ClusterIP
  selector:
    app: node-exporter
  ports:
  - port: 9100
    targetPort: 9100
```

Apply both files:

```bash
kubectl apply -f prometheus.yaml
kubectl apply -f node-exporter.yaml
kubectl get all -n monitoring 
```

### 📊 4. Deploy Grafana with Helm

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install grafana grafana/grafana -n monitoring --set service.type=NodePort
```

Get admin password:

```bash
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

Get Grafana NodePort:

```bash
kubectl get svc -n monitoring grafana
```

zznWkUYEd0HsPoloEEORVd59CpHqYOuUoaoeJ9eK
### 🖼️ 5. Configure Grafana

- Open `http://<node-ip>:<grafana-nodeport>`
- Add Prometheus as data source: `http://prometheus:9090`
- Import dashboard ID `1860`

---
