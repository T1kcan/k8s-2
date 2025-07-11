# Use Grafana Alloy to send logs to Loki
1. Install Alloy: https://grafana.com/docs/alloy/latest/set-up/install/linux/

2. To view the Alloy UI within the sandbox, Alloy must run on all interfaces. Run the following command before you start the Alloy service.
```bash
sed -i -e 's/CUSTOM_ARGS=""/CUSTOM_ARGS="--server.http.listen-addr=0.0.0.0:12345"/' /etc/default/alloy
```
3. Run Alloy.

# Set up a local Grafana instance

1. Create a directory and save the Docker Compose file as:
```bash
mkdir alloy-tutorial
cd alloy-tutorial
touch docker-compose.yml
```
2. Copy the following Docker Compose file into docker-compose.yml.
- docker-compose.yml
```yaml
 version: '3'
 services:
   loki:
     image: grafana/loki:3.0.0
     ports:
       - "3100:3100"
     command: -config.file=/etc/loki/local-config.yaml
   prometheus:
     image: prom/prometheus:v2.47.0
     command:
       - --web.enable-remote-write-receiver
       - --config.file=/etc/prometheus/prometheus.yml
     ports:
       - "9090:9090"
   grafana:
     environment:
       - GF_PATHS_PROVISIONING=/etc/grafana/provisioning
       - GF_AUTH_ANONYMOUS_ENABLED=true
       - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
     entrypoint:
       - sh
       - -euc
       - |
         mkdir -p /etc/grafana/provisioning/datasources
         cat <<EOF > /etc/grafana/provisioning/datasources/ds.yaml
         apiVersion: 1
         datasources:
         - name: Loki
           type: loki
           access: proxy
           orgId: 1
           url: http://loki:3100
           basicAuth: false
           isDefault: false
           version: 1
           editable: false
         - name: Prometheus
           type: prometheus
           orgId: 1
           url: http://prometheus:9090
           basicAuth: false
           isDefault: true
           version: 1
           editable: false
         EOF
         /run.sh
     image: grafana/grafana:11.0.0
     ports:
       - "3000:3000"
```
3. To start the local Grafana instance, run the following command.
```bash
 docker-compose up -d
```
4. Open http://localhost:3000 in your browser to access the Grafana UI.

# Configure Alloy
After the local Grafana instance is set up, the next step is to configure Alloy. You use components in the config.alloy file to tell Alloy which logs you want to scrape, how you want to process that data, and where you want the data sent.

The examples run on a single host so that you can run them on your laptop or in a Virtual Machine. You can try the examples using a config.alloy file and experiment with the examples.

### Create a config.alloy file
Create a config.alloy file within your current working directory.
```bash
touch config.alloy
```
### First component: Log files
Copy and paste the following component configuration at the top of the file.
```yaml
 local.file_match "local_files" {
     path_targets = [{"__path__" = "/var/log/*.log"}]
     sync_period = "5s"
 }
```
This configuration creates a local.file_match component named local_files which does the following:

- It tells Alloy which files to source.
- It checks for new files every 5 seconds.

### Second component: Scraping

Copy and paste the following component configuration below the previous component in your config.alloy file:
```yaml
  loki.source.file "log_scrape" {
    targets    = local.file_match.local_files.targets
    forward_to = [loki.process.filter_logs.receiver]
    tail_from_end = true
  }
```
This configuration creates a loki.source.file component named log_scrape which does the following:

- It connects to the local_files component as its source or target.
- It forwards the logs it scrapes to the receiver of another component called filter_logs .
- It provides extra attributes and options to tail the log files from the end so you don't ingest the entire log file history.

### Third component: Filter non-essential logs

Filtering non-essential logs before sending them to a data source can help you manage log volumes to reduce costs.

The following example demonstrates how you can filter out or drop logs before sending them to Loki.

Copy and paste the following component configuration below the previous component in your config.alloy file:
```yaml
  loki.process "filter_logs" {
    stage.drop {
        source = ""
        expression  = ".*Connection closed by authenticating user root"
        drop_counter_reason = "noisy"
      }
    forward_to = [loki.write.grafana_loki.receiver]
    }
```

The loki.process component allows you to transform, filter, parse, and enrich log data. Within this component, you can define one or more processing stages to specify how you would like to process log entries before they're stored or forwarded.

This configuration creates a loki.process component named filter_logs which does the following:

- It receives scraped log entries from the default log_scrape component.
- It uses the stage.drop block to define what to drop from the scraped logs.
- It uses the expression parameter to identify the specific log entries to drop.
- It uses an optional string label drop_counter_reason to show the reason for dropping the log entries.
- It forwards the processed logs to the receiver of another component called grafana_loki .

The loki.process documentation provides more comprehensive information on processing logs.

### Fourth component: Write logs to Loki
Copy and paste this component configuration below the previous component in your config.alloy file.
```yaml
  loki.write "grafana_loki" {
    endpoint {
      url = "http://localhost:3100/loki/api/v1/push"

      // basic_auth {
      //  username = "admin"
      //  password = "admin"
      // }
    }
  }
```

Final status of alloy file:
```yaml
local.file_match "local_files" {
  path_targets = [{"__path__" = "/var/log/*.log"}]
  sync_period = "5s"
}

loki.source.file "log_scrape" {
  targets    = local.file_match.local_files.targets
  forward_to = [loki.process.filter_logs.receiver]
  tail_from_end = true
}

loki.process "filter_logs" {
  stage.drop {
    source = ""
    expression  = ".*Connection closed by authenticating user root"
    drop_counter_reason = "noisy"
    }
  forward_to = [loki.write.grafana_loki.receiver]
}

loki.write "grafana_loki" {
  endpoint {
    url = "http://localhost:3100/loki/api/v1/push"

    // basic_auth {
    //  username = "admin"
    //  password = "admin"
    // }
  }
}
```

This final component creates a loki.write component named grafana_loki that points to http://localhost:3100/loki/api/v1/push .

This completes the simple configuration pipeline.

The basic_auth block is commented out because the local docker-compose stack doesn't require it. It's included in this example to show how you can configure authorization for other environments. For further authorization options, refer to the loki.write component reference.

With this configuration, Alloy connects directly to the Loki instance running in the Docker container.

## Reload the configuration

Copy your local config.alloy file into the default Alloy configuration file location.
```bash
sudo cp config.alloy /etc/alloy/config.alloy
```
Call the /-/reload endpoint to tell Alloy to reload the configuration file without a system service restart.
```bash
curl -X POST http://localhost:12345/-/reload
```
This step uses the Alloy UI on localhost port 12345 . If you chose to run Alloy in a Docker container, make sure you use the --server.http.listen-addr= argument. If you don't use this argument, the debugging UI won't be available outside of the Docker container.

Optional: You can do a system service restart Alloy and load the configuration file.
```bash
sudo systemctl reload alloy
```
Inspect your configuration in the Alloy UI

Open http://localhost:12345 and click the Graph tab at the top. The graph should look similar to the following:

The Alloy UI shows you a visual representation of the pipeline you built with your Alloy component configuration.

You can see that the components are healthy, and you are ready to explore the logs in Grafana.

## Log in to Grafana and explore Loki logs

Open http://localhost:3000/explore to access Explore feature in Grafana.

Select Loki as the data source and click the Label Browser button to select a file that Alloy has sent to Loki.

Here you can see that logs are flowing through to Loki as expected, and the end-to-end configuration was successful.

---

# Use Grafana Alloy to send metrics to Prometheus

## Configure Alloy
In this tutorial, you configure Alloy to collect metrics and send them to Prometheus.

You add components to your config.alloy file to tell Alloy which metrics you want to scrape, how you want to process that data, and where you want the data sent.

The following steps build on the config.alloy file you created in the previous tutorial.

### First component: Scraping
Paste the following component configuration at the top of your config.alloy file:
```yaml
prometheus.exporter.unix "local_system" { }

prometheus.scrape "scrape_metrics" {
  targets         = prometheus.exporter.unix.local_system.targets
  forward_to      = [prometheus.relabel.filter_metrics.receiver]
  scrape_interval = "10s"
}
```
This configuration creates a prometheus.scrape component named scrape_metrics which does the following:

- It connects to the local_system component as its source or target.
- It forwards the metrics it scrapes to the receiver of another component called filter_metrics .
- It tells Alloy to scrape metrics every 10 seconds.

### Second component: Filter metrics

Filtering non-essential metrics before sending them to a data source can help you reduce costs and allow you to focus on the data that matters most.

The following example demonstrates how you can filter out or drop metrics before sending them to Prometheus.

Paste the following component configuration below the previous component in your config.alloy file:
```yaml
prometheus.relabel "filter_metrics" {
  rule {
    action        = "drop"
    source_labels = ["env"]
    regex         = "dev"
  }

  forward_to = [prometheus.remote_write.metrics_service.receiver]
}
```
The prometheus.relabel component is commonly used to filter Prometheus metrics or standardize the label set passed to one or more downstream receivers. You can use this component to rewrite the label set of each metric sent to the receiver. Within this component, you can define rule blocks to specify how you would like to process metrics before they're stored or forwarded.

This configuration creates a prometheus.relabel component named filter_metrics which does the following:

- It receives scraped metrics from the scrape_metrics component.
- It tells Alloy to drop metrics that have an "env" label equal to "dev" .
- It forwards the processed metrics to the receiver of another component called metrics_service .

### Third component: Write metrics to Prometheus
Paste the following component configuration below the previous component in your config.alloy file:
```yaml
prometheus.remote_write "metrics_service" {
    endpoint {
        url = "http://localhost:9090/api/v1/write"

        // basic_auth {
        //   username = "admin"
        //   password = "admin"
        // }
    }
}
```

This final component creates a prometheus.remote_write component named metrics_service that points to http://localhost:9090/api/v1/write .

This completes the simple configuration pipeline.

The basic_auth is commented out because the local docker-compose stack doesn't require it. It's included in this example to show how you can configure authorization for other environments. For further authorization options, refer to the prometheus.remote_write component documentation.
This connects directly to the Prometheus instance running in the Docker container.

## Reload the configuration
Copy your local config.alloy file into the default Alloy configuration file location.
```bash
sudo cp config.alloy /etc/alloy/config.alloy
```
Call the /-/reload endpoint to tell Alloy to reload the configuration file without a system service restart.
```bash
curl -X POST http://localhost:12345/-/reload
```
This step uses the Alloy UI on localhost port 12345 . If you chose to run Alloy in a Docker container, make sure you use the --server.http.listen-addr= argument. If you don't use this argument, the debugging UI won't be available outside of the Docker container.

Optional: You can do a system service restart Alloy and load the configuration file:
```bash
sudo systemctl reload alloy
```

## Inspect your configuration in the Alloy UI
Open http://localhost:12345 and click the Graph tab at the top. The graph should look similar to the following:

The Alloy UI shows you a visual representation of the pipeline you built with your Alloy component configuration.

You can see that the components are healthy, and you are ready to explore the metrics in Grafana.

## Log into Grafana and explore metrics in Prometheus

Open http://localhost:3000/explore/metrics/ to access the Metrics Drilldown feature in Grafana.

From here you can visually explore the metrics sent to Prometheus by Alloy.

You can also build PromQL queries manually to explore the data further.

Open http://localhost:3000/explore to access the Explore feature in Grafana.

Select Prometheus as the data source and click the Metrics Browser button to select the metric, labels, and values for your labels.

Here you can see that metrics are flowing through to Prometheus as expected, and the end-to-end configuration was successful.


