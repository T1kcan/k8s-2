
# Hands-On: Getting Started with Prometheus

This document will walk you through setting up Prometheus to monitor your local machine. We'll cover installing Prometheus, a Node Exporter (to get machine metrics), and Grafana for visualization.

## 1. Prerequisites

Before we begin, ensure you have the following installed on your system:

- `wget` or `curl` (for downloading files)
- `tar` (for extracting compressed archives)
- A web browser (for accessing Prometheus and Grafana UIs)

We'll primarily use the command line for setup.

## 2. Setting Up Your Environment

We'll create a dedicated directory for our Prometheus setup.

```bash
mkdir prometheus-hands-on
cd prometheus-hands-on
```

## 3. Installing Prometheus Server

### 3.1 Download Prometheus

Visit the Prometheus Downloads page to find the latest stable release. We'll use v2.53.0 for this example.

```bash
PROMETHEUS_VERSION="2.53.0"
PROMETHEUS_TAR="prometheus-${PROMETHEUS_VERSION}.linux-amd64.tar.gz"
wget https://github.com/prometheus/prometheus/releases/download/v${PROMETHEUS_VERSION}/${PROMETHEUS_TAR}
```

### 3.2 Extract and Organize

```bash
tar xvf ${PROMETHEUS_TAR}
mv prometheus-${PROMETHEUS_VERSION}.linux-amd64 prometheus-server
```

### 3.3 Basic Prometheus Configuration

Create a minimal `prometheus.yml`:

```bash
cat <<EOF > prometheus-server/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
EOF
```

### 3.4 Start Prometheus Server

```bash
cd prometheus-server
./prometheus --config.file=prometheus.yml &
cd ..
```

### 3.5 Verify Prometheus UI

Open your browser: [http://localhost:9090](http://localhost:9090)  
Go to **Status → Targets** and verify `localhost:9090` is UP.

## 4. Installing Node Exporter (Machine Metrics)

### 4.1 Download Node Exporter

```bash
NODE_EXPORTER_VERSION="1.8.0"
NODE_EXPORTER_TAR="node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz"
wget https://github.com/prometheus/node_exporter/releases/download/v${NODE_EXPORTER_VERSION}/${NODE_EXPORTER_TAR}
```

### 4.2 Extract and Organize

```bash
tar xvf ${NODE_EXPORTER_TAR}
mv node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64 node-exporter
```

### 4.3 Start Node Exporter

```bash
cd node-exporter
./node_exporter &
cd ..
```

### 4.4 Verify Node Exporter Metrics

Visit: [http://localhost:9100/metrics](http://localhost:9100/metrics)

## 5. Configure Prometheus to Scrape Node Exporter

### 5.1 Edit `prometheus.yml`

Append:

```yaml
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
```

Full config:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
```

### 5.2 Reload Prometheus Configuration

```bash
cd prometheus-server
curl -X POST http://localhost:9090/-/reload
cd ..
```

### 5.3 Verify in UI

Check: **Status → Targets**. Both jobs should be UP.

## 6. Installing Grafana (Visualization)

### 6.1 Download Grafana

```bash
GRAFANA_VERSION="10.4.3"
GRAFANA_TAR="grafana-${GRAFANA_VERSION}.linux-amd64.tar.gz"
wget https://dl.grafana.com/oss/release/${GRAFANA_TAR}
```

### 6.2 Extract and Organize

```bash
tar xvf ${GRAFANA_TAR}
mv grafana-${GRAFANA_VERSION} grafana
```

### 6.3 Start Grafana Server

```bash
cd grafana/bin
./grafana-server web &
cd ../..
```

### 6.4 Access Grafana UI

Visit: [http://localhost:3000](http://localhost:3000)  
Default credentials:
- **Username**: admin
- **Password**: admin

## 7. Connecting Grafana to Prometheus

### 7.1 Add Prometheus Data Source

1. In Grafana, click **Configuration (gear)** → **Data sources**
2. Click **Add data source**
3. Select **Prometheus**

### 7.2 Configure Source

- Name: `Prometheus-Local`
- URL: `http://localhost:9090`
- Click **Save & Test**

## 8. Importing a Node Exporter Dashboard

### 8.1 Find Dashboard

Dashboard ID: **1860** on [grafana.com](https://grafana.com/grafana/dashboards/1860-node-exporter-full/)

### 8.2 Import It

1. Click **+ → Import**
2. Enter ID: `1860` → Click **Load**
3. Select Prometheus data source → Click **Import**

### 8.3 Explore Dashboard

View system metrics like CPU, memory, disk I/O, and network traffic.

## 9. Cleaning Up (Optional)

### Stop Running Processes

```bash
ps aux | grep prometheus | grep -v grep
ps aux | grep node_exporter | grep -v grep
ps aux | grep grafana-server | grep -v grep
kill <PID>
```

Or Ctrl+C in their terminal windows if run in the foreground.

### Remove the directory

```bash
cd ..
rm -rf prometheus-hands-on
```

## Conclusion

You’ve successfully set up Prometheus, Node Exporter, and Grafana to monitor your local system. From here, explore custom metrics, alerts, and dashboards!
