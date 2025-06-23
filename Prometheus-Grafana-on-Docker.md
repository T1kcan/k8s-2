# Prometheus & Grafana on Docker
#This lab is for a Basic Prometheus/grafana setup.

https://prometheus.io/docs/introduction/comparison/

architecture:
https://prometheus.io/docs/introduction/overview/#architecture


## Metrics
are pulled from http://

/metrics

Two types: https://prometheus.io/docs/concepts/metric_types/#metric-types

- HELP; a description of the metric
- TYPE; 3 sub types
    - counter
    - gauge
    - histogram
notation:
    - <metric_name>{k=v, ...}

## Exporters
if you service/server doesn't provide an endpoint, you can use an 'exporter', which fetches the metrics and coverts to the correct format, and then provides an endpoint for Prometheus to pull from (/metrics)

prometheus.io/docs/instrumenting/exporters/

These exporters also come as a docker container, so you can always use them as a sidecar to your main service container

## Client libraries

You can also embed code into your application for prometheus

https://prometheus.io/docs/instrumenting/clientlibs/

## Pushgateway

Designed to recieve 'push metrics' with short running jobs, so that Prometheus can pull them

## prometheus.yml
the main config file for prometheus, with 3 main sections:

- global;
- rule_files:
- scrape_configs; configuration for each target

note that prometheus has it's own /metrics endpoint on port 9090

## Learn more
promlabs offers a free intro class:

https://training.promlabs.com/training/introduction-to-prometheus

# Step 1

## Initial Setup
- Setup config file
```bash
mkdir tmp

nano tmp/prometheus.yml
```
```ini
global:
  scrape_interval: 30s
  scrape_timeout: 10s

rule_files:
#  - alert.yml

scrape_configs:
  - job_name: services
    metrics_path: /metrics
    static_configs:
      - targets:
          - 'localhost:9090'
```           
- start the docker container
```bash
docker run --name my-prometheus --net host -v $(pwd)/tmp/prometheus.yml:/etc/prometheus/prometheus.yml -p 9090:9090 prom/prometheus
```
Link for traffic into host 1 on port 9090
```bash
localhost:9090
```
Lets check that Prometheus is picking up the metrics endpoint.

In the GUI, goto the status>targets page and you should see the localhost:9090 (Prometheus metrics) endpoint up. You may have to wait/refresh a minute.

In the labels section, note the endpoint label, and the job label, which is the job_name in the prometheus config yml

and stop the container

ctrl-c
```bash
docker stop my-prometheus && docker rm my-prometheus
```

# Step 2

## Monitor Docker
- config docker
Lets configure docker to provider metrics:
```bash
nano /etc/docker/daemon.json
```
add the following to the daemon.json file, besure the yaml syntax is correct.
```ini
  "metrics-addr" : "127.0.0.1:9323",
  "experimental" : true
```
system status docker # restart
```bash
systemctl daemon-reload

systemctl restart docker

systemctl status docker
```
confirm we get metrics:
```bash
curl http://localhost:9323/metrics
```
### add metrics to Prometheus
add the follow to the config:
```bash
cd ~ && nano ./tmp/prometheus.yml

  - job_name: docker  
    metrics_path: /metrics
    static_configs:
      - targets: ['localhost:9323']
```
https://docs.docker.com/config/daemon/prometheus/

start the docker container, connected directed with the host (--net host)
```bash
docker run --name my-prometheus -d --net host -v $(pwd)/tmp/prometheus.yml:/etc/prometheus/prometheus.yml -p 9090:9090 prom/prometheus
```
and connect to the web gui

https://localhost:9090

check the status>targets page do check that it is getting data from the docker endpoint (port 9323)

Open the 'graph tab' and use the 'meterics explorer' next to the Execute button to see available metrics/logs

## Add Grafana
Open a new tab,

https://grafana.com/docs/grafana/v9.0/getting-started/build-first-dashboard/

https://grafana.com/docs/grafana/v9.0/setup-grafana/installation/docker/
```bash
docker run --name grafana --net host -p 3000:3000 grafana/grafana-oss
```
connect to port 3000

https://localhost:3000

can use: un & pw: admin

You can skip the password reset.

go into datasources and add prometheous, with HTTP:URL:

http://localhost:9090

save and test

Lets add a dashboard to monitor the prometheus database itself:

https://grafana.com/grafana/dashboards/

https://grafana.com/grafana/dashboards/3662-prometheus-2-0-overview/

in the Grafana web portal:

click on the 4 squares on the left hand side
click on '+ import'
enter 3662 ' Import via grafana.com'
under 'prometheus' select prometheus (default)
click on import
It should now take you to the dashboard page and show you the statistics.

for reference
https://grafana.com/docs/grafana/v9.0/getting-started/build-first-dashboard/

https://grafana.com/docs/grafana/v9.0/setup-grafana/installation/docker/

# Step 3
## add cAdvisor
"cAdvisor (Container Advisor) provides container users an understanding of the resource usage and performance characteristics of their running containers."

https://prometheus.io/docs/guides/cadvisor/

https://github.com/google/cadvisor

to start cAdvisor, in a new tab:

VERSION=v0.53.0 # use the latest release version from https://github.com/google/cadvisor/releases
```bash
sudo docker run \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --volume=/dev/disk/:/dev/disk:ro \
  --publish=8080:8080 \
  --detach=true \
  --name=cadvisor \
  --privileged \
  --device=/dev/kmsg \
  gcr.io/cadvisor/cadvisor:$VERSION
```
you can access the cAdvisor at http://localhost:8080

To add to Prometheus:
```bash
cd ~ && nano ./tmp/prometheus.yml
```
```ini
  - job_name: cadvisor
    scrape_interval: 5s
    metrics_path: /metrics
    static_configs:
    - targets:
      - localhost:8080
```
restart Prometheus
```bash
docker restart my-prometheus
```
and check the status>target page to confirm the endpoint is up

and add the Grafana dashboard id 14282

# Step 4

## Monitor the Host with an Exporter

https://prometheus.io/docs/guides/node-exporter/#monitoring-linux-host-metrics-with-the-node-exporter
```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.4.0/node_exporter-1.4.0.linux-amd64.tar.gz

tar xvfz node_exporter-1.4.0.linux-amd64.tar.gz

cd node_exporter-1.4.0.linux-amd64

./node_exporter &

curl http://localhost:9100/metrics
```

add the follow to the prometheus config:
```bash
cd ~ && nano ./tmp/prometheus.yml
```
```ini
  - job_name: node
    static_configs:
    - targets: ['localhost:9100']
```

restart Prometheus
```bash
docker restart my-prometheus
```
and check the status>target page to confirm the endpoint is up

In Prometheus:

Metrics specific to the Node Exporter are prefixed with 'node_'

Can you now add the 'Node Exporter Full' Grafana dashboard, with id 1860

## pushgateway - Work-in-progress
https://prometheus.io/docs/practices/pushing/

https://github.com/prometheus/pushgateway/blob/master/README.md


