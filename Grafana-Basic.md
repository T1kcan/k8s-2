Throughout this course, you will learn the following:
* How to install Grafana on your local machine or server.
* The basics of configuring Grafana and connecting it to a data source such as Prometheus.
* Creating a basic dashboard to visualize your data.
* By the end of this course, you will have a solid understanding of Grafana and be able to create your own monitoring dashboard to gain valuable insights from your data.

Let's get started!

### Step 1: Collecting Metrics
Before we can start visualizing our data, we need to collect some metrics. The easiest todo this is by collecting our virtiual enviroments system metrics. To do this we will use Node Exporter and Prometheus. Lets break down each of these components and we can install them within our virtual enviroment.

### Node Exporter
Node Exporter is a Prometheus exporter for hardware and OS metrics exposed by *nix kernels, written in Go with pluggable metric collectors. It allows for the collection of hardware and OS metrics and is a great way to collect system metrics for your virtual enviroment. Lets install Node Exporter on our virtual enviroment:

Download the latest version of Node Exporter from the Prometheus download page.
```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
```
Extract the tarball and navigate to the extracted directory. Move to bin directory.
```bash
tar -xvf node_exporter-1.7.0.linux-amd64.tar.gz && cd node_exporter-1.7.0.linux-amd64 && sudo cp node_exporter /usr/local/bin
```
Lets turn Node Exporter into a service (we will do the hard work for you).
```bash
sudo useradd -rs /bin/false node_exporter && echo -e "[Unit]\nDescription=Node Exporter\nAfter=network.target\n\n[Service]\nUser=node_exporter\nGroup=node_exporter\nType=simple\nExecStart=/usr/local/bin/node_exporter\n\n[Install]\nWantedBy=multi-user.target" | sudo tee /etc/systemd/system/node_exporter.service > /dev/null && sudo systemctl daemon-reload && sudo systemctl start node_exporter
```
Lets finaly check if Node Exporter is running by visiting the following URL in your browser: http://localhost:9100. If you see a page with a bunch of metrics, then Node Exporter is running correctly.

## Prometheus
Prometheus is a monitoring and alerting toolkit that is designed for reliability, scalability, and maintainability. It is a powerful tool for collecting and querying metrics and is a great way to store the metrics collected by Node Exporter. Lets install Prometheus on our virtual enviroment:

Download the latest version of Prometheus from the Prometheus download page.
```bash
wget https://github.com/prometheus/prometheus/releases/download/v2.50.1/prometheus-2.50.1.linux-amd64.tar.gz
```
Extract the tarball and navigate to the extracted directory. Move to bin directory.
```bash
tar -xvf prometheus-2.50.1.linux-amd64.tar.gz && cd prometheus-2.50.1.linux-amd64 && sudo cp prometheus /usr/local/bin
```
Lets turn Prometheus into a service.
```bash
sudo useradd -rs /bin/false prometheus && sudo mkdir /etc/prometheus /var/lib/prometheus && sudo cp prometheus.yml /etc/prometheus/prometheus.yml && sudo cp -r consoles/ console_libraries/ /etc/prometheus/ && sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus && echo -e "[Unit]\nDescription=Prometheus\nAfter=network.target\n\n[Service]\nUser=prometheus\nGroup=prometheus\nType=simple\nExecStart=/usr/local/bin/prometheus --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/var/lib/prometheus --web.console.templates=/etc/prometheus/consoles --web.console.libraries=/etc/prometheus/console_libraries\n\n[Install]\nWantedBy=multi-user.target" | sudo tee /etc/systemd/system/prometheus.service > /dev/null && sudo systemctl daemon-reload && sudo systemctl start prometheus
```
Lets finally add the Node Exporter as a target to Prometheus. Here is what the prometheus.yml file should look like:
- prometheus.yml
```yaml
# my global config
global:
  scrape_interval: 5s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 5s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "Node Exporter"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["0.0.0.0:9100"]
```
We will copy this file to the /etc/prometheus directory and restart the prometheus service.

sudo cp prometheus.yml /etc/prometheus/prometheus.yml && sudo systemctl restart prometheus
Lets finaly check if Prometheus is running by visiting the following URL in your browser: http://localhost:9090. If you see a page with a graph, then Prometheus is running correctly.

## Step 2: Installing Grafana

In this step, we will install Grafana on within our virtual environment. Grafana is an open-source platform for monitoring and observability that allows you to create, explore, and share dashboards and data visualizations.

### Installing Grafana
Lets install Grafana via apt install.
```bash
sudo apt-get install -y apt-transport-https software-properties-common wget &&
sudo mkdir -p /etc/apt/keyrings/ &&
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null &&
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list &&
sudo apt-get update && sudo apt-get install -y grafana
```
Lets make sure our Grafana service is running.
```bash
sudo systemctl start grafana-server && sudo systemctl status grafana-server
```
We should now be able to access Grafana by visiting the following URL in your browser: http://localhost:3000. If you see a page with a login prompt, then Grafana is running correctly.

## Step 3: Provisioning our Prometheus datasource

Now that we have our Grafana and Prometheus services running, we need to configure Grafana to use Prometheus as a data source. This will allow us to visualize the metrics collected by Prometheus in Grafana. We will show you two methods to do this: using the Grafana UI and using a provisioning file.

### Method 1: Using the Grafana UI
Open your browser and go to the Grafana UI by clicking on the following link: http://localhost:3000

Sign in with the following credentials:

Username: admin
Password: admin
Once logged in, open the left hand side menu, then click on Connections -> Data Sources. 

Click on the Add data source button.

Select Prometheus from the list of data sources. 

In the Connection section, set the following fields:

Prometheus server URL: http://localhost:9090
Click on the Save & Test button to save the data source and test the connection to Prometheus.

You have now successfully configured Grafana to use Prometheus as a data source. You can now start creating dashboards to visualize the metrics collected by Prometheus.

### Method 2: Using a Provisioning File

We have already created a provisioning file for you. We will move this file to the correct location and restart the Grafana service. Run the following command to do this:

- prometheus_datasource.yml
```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    orgId: 1
    url: http://localhost:9090
    basicAuth: false
    isDefault: true
    version: 1
    editable: true
```
```bash
sudo cp prometheus_datasource.yml /etc/grafana/provisioning/datasources/ && sudo systemctl restart grafana-server
```
Open your browser and go to the Grafana UI by clicking on the following link: http://localhost:3000

Sign in with the following credentials:

Username: admin
Password: admin
You have now successfully configured Grafana to use Prometheus as a data source using a provisioning file. You can now start creating dashboards to visualize the metrics collected by Prometheus.

## Step 4: Creating a Basic Dashboard

In this step, we will create a basic dashboard in Grafana to visualize the metrics collected by Prometheus. We will start by adding a new dashboard and then create a simple graph to display the CPU usage of our virtual environment.

### Adding a New Dashboard
Open your browser and go to the Grafana UI by clicking on the following link: http://localhost:3000
Sign in with the following credentials:
Username: admin
Password: admin
Note: If you are already logged in, you can skip this step.

Once logged in, open the left hand side menu, then click on Dashboards. Next select the button + Create Dashboard.

### Creating a Graph Panel

<!-- Click on the Add visualization button to add a new panel to the dashboard.
Select Prometheus from the list of data sources.
Select the Time Series visualization from the right handside panel.
In the middle of the screen, Query section, set the following fields:
Data Source: Prometheus
In the Query tab, toggle from Builder to Code and enter the following promQL query, Enter a PromQL query_:
```yaml
Query: 100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

This query calculates the average CPU utilization (not in idle mode) for each instance over the past 5 minutes. It subtracts the average percentage of time each CPU instance has been idle (as measured over 5-minute intervals) from 100%, effectively giving the average active CPU usage.

Click on the Run button to execute the query and visualize the results.
Lets add some additional configuration to our time series graph:
Title: CPU Usage
Unit: percent(0-100)
Min: 0
Max: 100
search.png

Top Tip: these are all configured via the configuration panel on the right handside. You can make use of the search bar within this panel to access the above fields quickly

Click on the Apply button to apply the changes to the graph panel.
Click on the Save dashboard button to save the new dashboard.
Enter a name for the dashboard and click on the Save button. -->
Select New --> Import and type 1860 then click Load to create preconfigured Node Exporter Full Dasboard.


---

## Set up the sample application
This tutorial uses a sample application to demonstrate some of the features in Grafana. To complete the exercises in this tutorial, you need to download the files to your local machine.

In this step, you'll set up the sample application, as well as supporting services, such as Loki.

Note: Prometheus, a popular time series database (TSDB), has already been configured as a data source as part of this tutorial.

1. Clone the github.com/grafana/tutorial-environment repository.
```bash
git clone https://github.com/grafana/tutorial-environment.git
```
2. Change to the directory where you cloned this repository:
```bash
cd tutorial-environment
```
3. Make sure Docker is running:
```bash
docker ps
```
No errors means it is running. If you get an error, then start Docker and then run the command again.

4. Start the sample application:
```bash
docker-compose up -d
```
The first time you run docker-compose up -d , Docker downloads all the necessary resources for the tutorial. This might take a few minutes, depending on your internet connection.

Note: If you already have Grafana, Loki, or Prometheus running on your system, then you might see errors because the Docker image is trying to use ports that your local installations are already using. Stop the services, then run the command again.

5. Ensure all services are up-and-running:
```bash
docker-compose ps
```
In the State column, it should say Up for all services.

6. Browse to the sample application on http://localhost:8081.

### Grafana News
The sample application, Grafana News, lets you post links and vote for the ones you like.

To add a link:

1. In Title, enter Example.

2. In URL, enter https://example.com.

3. Click Submit to add the link.

The link appears in the list under the Grafana News heading.

To vote for a link, click the triangle icon next to the name of the link.


### Open Grafana
Grafana is an open source platform for monitoring and observability that lets you visualize and explore the state of your systems.

1. Browse to http://localhost:3000.

### Explore your metrics

Grafana Explore is a workflow for troubleshooting and data exploration. In this step, you'll be using Explore to create ad-hoc queries to understand the metrics exposed by the sample application. Specifically, you'll explore requests received by the sample application.

* Ad-hoc queries are queries that are made interactively, with the purpose of exploring data. An ad-hoc query is commonly followed by another, more specific query.
1. Click the menu icon and, in the sidebar, click Explore. A dropdown menu for the list of available data sources is on the upper-left side. The Prometheus data source should already be selected. If not, choose Prometheus.

2. Confirm that you're in code mode by checking the Builder/Code toggle at the top right corner of the query panel.

3. In the query editor, where it says Enter a PromQL query…, enter 
```text 
tns_request_duration_seconds_count
``` 
and then press Shift + Enter. A graph appears.

4. In the top right corner, click the dropdown arrow on the Run Query button, and then select 5s. Grafana runs your query and updates the graph every 5 seconds.

You just made your first PromQL query! PromQL is a powerful query language that lets you select and aggregate time series data stored in Prometheus.

tns_request_duration_seconds_count is a counter, a type of metric whose value only ever increases. Rather than visualizing the actual value, you can use counters to calculate the rate of change, i.e. how fast the value increases.

5. Add the rate function to your query to visualize the rate of requests per second. Enter the following in the query editor and then press Shift + Enter.
```text
rate(tns_request_duration_seconds_count[5m])
```
Immediately below the graph there's an area where each time series is listed with a colored icon next to it. This area is called the legend.

PromQL lets you group the time series by their labels, using the sum aggregation operator.

6. Add the sum aggregation operator to your query to group time series by route:
```text
sum(rate(tns_request_duration_seconds_count[5m])) by(route)
```
7. Go back to the sample application and generate some traffic by adding new links, voting, or just refresh the browser.

8. Back in Grafana, in the upper-right corner, click the time picker, and select Last 5 minutes. By zooming in on the last few minutes, it's easier to see when you receive new data.

Depending on your use case, you might want to group on other labels. Try grouping by other labels, such as status_code , by changing the by(route) part of the query to by(status_code) .

### Add a logging data source

Grafana supports log data sources, like Loki. Just like for metrics, you first need to add your data source to Grafana.

1. Click the menu icon and, in the sidebar, click Connections and then Data sources.
2. Click + Add new data source.
3. In the list of data sources, click Loki.
4. In the URL box, enter http://loki:3100
5. Scroll to the bottom of the page and click Save & Test to save your changes.
You should see the message "Data source successfully connected." Loki is now available as a data source in Grafana.

### Explore your logs

Grafana Explore not only lets you make ad-hoc queries for metrics, but lets you explore your logs as well.

1. Click the menu icon and, in the sidebar, click Explore.

2. In the data source list at the top, select the Loki data source.

3. Confirm that you're in code mode by checking the Builder/Code toggle at the top right corner of the query panel.

4. Enter the following in the query editor, and then press Shift + Enter:
```text
{filename="/var/log/tns-app.log"}
```
5. Grafana displays all logs within the log file of the sample application. The height of each bar in the graph encodes the number of logs that were generated at that time.

6. Click and drag across the bars in the graph to filter logs based on time.

Not only does Loki let you filter logs based on labels, but on specific occurrences.

Let's generate an error, and analyze it with Explore.

1. In the sample application, post a new link without a URL to generate an error in your browser that says empty url .

2. Go back to Grafana and enter the following query to filter log lines based on a substring:
```text
{filename="/var/log/tns-app.log"} |= "error"
```
3. Click the log line that says level=error msg="empty url" to see more information about the error.

Note: If you're in Live mode, clicking logs does not show more information about the error. Instead, stop and exit the live stream, then click the log line there.
Logs are helpful for understanding what went wrong. Later in this tutorial, you'll see how you can correlate logs with metrics from Prometheus to better understand the context of the error.

### Build a dashboard
A dashboard gives you an at-a-glance view of your data and lets you track metrics through different visualizations.

Dashboards consist of panels, each representing a part of the story you want your dashboard to tell.

Every panel consists of a query and a visualization. The query defines what data you want to display, whereas the visualization defines how the data is displayed.

1. Click the menu icon and, in the sidebar, click Dashboards.

2. On the Dashboards page, click New in top right corner and select New Dashboard in the drop-down.

3. Click + Add visualization.

4. In the modal that opens, select the Prometheus data source that you just added.

5. In the Query tab below the graph, enter the query from earlier and then press Shift + Enter:
```text
sum(rate(tns_request_duration_seconds_count[5m])) by(route)
```
6. In the panel editor on the right, under Panel options, change the panel title to "Traffic".

7. Click Apply in the top-right corner to save the panel and go back to the dashboard view.

8. Click the Save dashboard (disk) icon at the top of the dashboard to save your dashboard.

9. Enter a name in the Dashboard name field and then click Save.

You should now have a panel added to your dashboard.

### Annotate events

When things go bad, it often helps if you understand the context in which the failure occurred. Time of last deploy, system changes, or database migration can offer insight into what might have caused an outage. Annotations allow you to represent such events directly on your graphs.

In the next part of the tutorial, we simulate some common use cases that someone would add annotations for.

1. To manually add an annotation, click anywhere in your graph, then click Add annotation. Note: you might need to save the dashboard first.

2. In Description, enter Migrated user database.

3. Click Save.

* Grafana adds your annotation to the graph. Hover your mouse over the base of the annotation to read the text.

Grafana also lets you annotate a time interval, with region annotations.

Add a region annotation:

1. Press Ctrl (or Cmd on macOS) and hold, then click and drag across the graph to select an area.
2. In Description, enter Performed load tests.
3. In Tags, enter testing.
4. Click Save.

### Using annotations to correlate logs with metrics
Manually annotating your dashboard is fine for those single events. For regularly occurring events, such as deploying a new release, Grafana supports querying annotations from one of your data sources. Let's create an annotation using the Loki data source we added earlier.

1. At the top of the dashboard, click the Dashboard settings (gear) icon.

2. Go to Annotations and click Add annotation query.

3. In Name, enter Errors.

4. In Data source, select Loki.

5. In Query, enter the following query:
```text
{filename="/var/log/tns-app.log"} |= "error"
```
6. Click Apply. Grafana displays the Annotations list, with your new annotation.

7. Click your dashboard name to return to your dashboard.

8. At the top left of your dashboard, there is now a toggle to display the results of the newly created annotation query. Press it if it's not already enabled.

9. Click the Save dashboard (disk) icon to save the changes.

10. To test the changes, go back to the sample application, post a new link without a URL to generate an error in your browser that says empty url .

The log lines returned by your query are now displayed as annotations in the graph.

Being able to combine data from multiple data sources in one graph allows you to correlate information from both Prometheus and Loki.

Annotations also work very well alongside alert rules. In the next and final section, we set up an alert rules for our app grafana.news and then we trigger it. This provides a quick intro to our new Alerting platform.

### Create a Grafana-managed alert rule
Alert rules allow you to identify problems in your system moments after they occur. By quickly identifying unintended changes in your system, you can minimize disruptions to your services.

Grafana's new alerting platform debuted with Grafana 8. A year later, with Grafana 9, it became the default alerting method. In this step we create a Grafana-managed alert rule. Then we trigger our new alert rule and send a test message to a dummy endpoint.

The most basic alert rule consists of two parts:

1. A Contact point - A Contact point defines how Grafana delivers an alert instance. When the conditions of an alert rule are met, Grafana notifies the contact points, or channels, configured for that alert rule.

* An alert instance is a specific occurrence that matches a condition defined by an alert rule, such as when the rate of requests for a specific route suddenly increases.
Some popular channels include:

* Email
* Webhooks
* Telegram
* Slack
* PagerDuty

2. An Alert rule - An Alert rule defines one or more conditions that Grafana regularly evaluates. When these evaluations meet the rule's criteria, the alert rule is triggered.

To begin, let's set up a webhook contact point. Once we have a usable endpoint, we write an alert rule and trigger a notification.

### Create a contact point for Grafana-managed alert rules
In this step, we set up a new contact point. This contact point uses the webhooks channel. In order to make this work, we also need an endpoint for our webhook channel to receive the alert notification. We can use Webhook.site to quickly set up that test endpoint. This way we can make sure that our alert manager is actually sending a notification somewhere.

1. Browse to Webhook.site.
2. Copy Your unique URL.
Your webhook endpoint is now waiting for the first request.

Next, let's configure a Contact Point in Grafana's Alerting UI to send notifications to our webhook endpoint.

1. Return to Grafana. In Grafana's sidebar, hover over the Alerting (bell) icon and then click Manage Contact points.
2. Click + Add contact point.
3. In Name, write Webhook.
4. In Integration, choose Webhook.
5. In URL, paste the endpoint to your webhook endpoint.
6. Click Test, and then click Send test notification to send a test alert notification to your webhook endpoint.
7. Navigate back to the webhook endpoint you created earlier. On the left side, there's now a POST / entry. Click it to see what information Grafana sent.
8. Return to Grafana and click Save contact point.
We have now created a dummy webhook endpoint and created a new Alerting Contact Point in Grafana. Now we can create an alert rule and link it to this new channel.

### Add an alert rule to Grafana
Now that Grafana knows how to notify us, it's time to set up an alert rule:

1. In Grafana's sidebar, hover over the Alerting (bell) icon and then click Alert rules.

In this tutorial, we use the advanced options for Grafana-managed alert rule creation. The advanced options let us define queries, expressions (used to manipulate the data), and the condition that must be met for the alert to be triggered (the default condition is the threshold).

2. Click + New alert rule.

3. For Section 1, name the rule fundamentals-test .

4. For Section 2, toggle the Advanced options button.

5. Find the query A box, and choose your Prometheus data source.

6. Enter the same Prometheus query that we used in our earlier panel:
```text
sum(rate(tns_request_duration_seconds_count[5m])) by(route)
```
7. Keep expressions B and C as they are. These expressions (Reduce and Threshold, respectively) are included by default when creating a new rule. Enter 0.2 as threshold value. You can read more about queries and conditions here.

8. Scroll down to bottom of section #2 and click the Preview button. You should see some data returned.

9. In Section 3, in Folder, create a new folder, by clicking New folder and typing a name for the folder. This folder contains our alert rules. For example: fundamentals . Then, click create .

10. In the Evaluation group, repeat the above step to create a new one. Name it fundamentals too.

11. Choose an Evaluation interval (how often the alert rule are evaluated). For example, every 10s (10 seconds).

12. Set the pending period. This is the time that a condition has to be met until the alert instance enters in Firing state and a notification is sent. Enter 0s . For the purposes of this tutorial, the evaluation interval is intentionally short. This makes it easier to test. This setting makes Grafana wait until an alert instance has fired for a given time before Grafana sends the notification.

13. In Section 4, choose Webhook as the Contact point.

14. Click Save rule and exit at the top of the page.

### Trigger a Grafana-managed alert rule
We have now configured an alert rule and a contact point. Now let's see if we can trigger a Grafana-managed alert rule by generating some traffic on our sample application.

1. Browse to localhost:8081.
2. Add a new title and URL, repeatedly click the vote button, or refresh the page to generate a traffic spike.
Once the query sum(rate(tns_request_duration_seconds_count[5m])) by(route) returns a value greater than 0.2 Grafana triggers our alert rule. Browse to the webhook endpoint we created earlier and find the sent Grafana alert notification with details and metadata.

Let's see how we can configure this.

1. In Grafana's sidebar, hover over the Alerting (bell) icon and then click Alert rules.

2. Expand the fundamentals > fundamentals folder to view our created alert rule.

3. Click the Edit icon and scroll down to Section 5.

4. Click the Link dashboard and panel button and select the dashboard and panel to which you want the alert instance to be added as an annotation.

5. Click Confirm and Save rule and exit to save all the changes.

6. In Grafana's sidebar, navigate to the dashboard by clicking Dashboards and selecting the dashboard you created.

7. To test the changes, follow the steps listed to trigger a Grafana-managed alert rule.

You should now see a red, broken heart icon beside the panel name, indicating that the alert rule has been triggered. An annotation for the alert instance, represented as a vertical red line, is also displayed.
