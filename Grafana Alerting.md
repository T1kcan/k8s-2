## Grafana Alerting

In this tutorial, we walk you through the process of setting up your first alert in just a few minutes. You'll witness your alert in action with real-time data, as well as sending alert notifications.

In this tutorial you will:

* Create a contact point.
* Set up an alert rule.
* Receive firing and resolved alert notifications in a public webhook.

To demonstrate the observation of data using the Grafana stack, download and run the following files.

1. Clone the tutorial environment repository.
```bash
git clone https://github.com/grafana/tutorial-environment.git
```
2. Change to the directory where you cloned the repository:
```bash
cd tutorial-environment
```
3. Run the Grafana stack:
```bash
docker-compose up -d
```
The first time you run docker compose up -d , Docker downloads all the necessary resources for the tutorial. This might take a few minutes, depending on your internet connection.

NOTE:

If you already have Grafana, Loki, or Prometheus running on your system, you might see errors, because the Docker image is trying to use ports that your local installations are already using. If this is the case, stop the services, then run the command again.

## Create a contact point

Besides being an open-source observability tool, Grafana has its own built-in alerting service. This means that you can receive notifications whenever there is an event of interest in your data, and even see these events graphed in your visualizations.

In this step, we set up a new contact point. This contact point uses the webhook integration. In order to make this work, we also need an endpoint for our webhook integration to receive the alert. We can use `Webhook.site` to quickly set up that test endpoint. This way we can make sure that our alert is actually sending a notification somewhere.

In your browser, sign in to your Grafana Cloud account.

To log in, navigate to http://localhost:3000, where Grafana is running.

In another tab, go to `Webhook.site`.

Copy Your unique URL.

Your webhook endpoint is now waiting for the first request.

Next, let's configure a contact point in Grafana's Alerting UI to send notifications to our webhook endpoint.

Return to Grafana. In Grafana's sidebar, hover over the Alerting (bell) icon and then click Contact points.

Click + Create contact point.

In Name, write Webhook.

In Integration, choose Webhook.

In URL, paste the endpoint to your webhook endpoint.

Click Test, and then click Send test notification to send a test alert to your webhook endpoint.

Navigate back to Webhook.site. On the left side, there's now a POST / entry. Click it to see what information Grafana sent.

Return to Grafana and click Save contact point.

We have created a dummy Webhook endpoint and created a new Alerting contact point in Grafana. Now, we can create an alert rule and link it to this new integration.

## Create an alert

Next, we establish an alert rule within Grafana Alerting to notify us whenever alert rules are triggered and resolved.

1. In Grafana, navigate to Alerts & IRM > Alerting > Alert rules. Click on + New alert rule.

2. Enter alert rule name for your alert rule. Make it short and descriptive as this appears in your alert notification. For instance, database-metrics

### Define query and alert condition

In this section, we use the default options for Grafana-managed alert rule creation. The default options let us define the query, a expression (used to manipulate the data -- the WHEN field in the UI), and the condition that must be met for the alert to be triggered (in default mode is the threshold).

Grafana includes a test data source that creates simulated time series data. This data source is included in the demo environment for this tutorial. If you're working in Grafana Cloud or your own local Grafana instance, you can add the data source through the Connections menu.

Select the `TestData` data source from the drop-down menu.

In the Alert condition section:

Keep Random Walk as the Scenario.

Keep Last as the value for the reducer function (WHEN ), and IS ABOVE 0 as the threshold value. This is the value above which the alert rule should trigger.
Click Preview alert rule condition to run the query.

It should return random time series data. The alert rule state should be `Firing` .


### Add folders and labels

In Folder, click + New folder and enter a name. For example: `metric-alerts` . This folder contains our alert rules.

### Set evaluation behavior

The alert rule evaluation defines the conditions under which an alert rule triggers, based on the following settings:

- Evaluation group: every alert rule is assigned to an evaluation group. You can assign the alert rule to an existing evaluation group or create a new one.

- Evaluation interval: determines how frequently the alert rule is checked. For instance, the evaluation may occur every 10s, 30s, 1m, 10m, etc.
Pending period: how long the condition must be met to trigger the alert rule.

- Keep firing for: defines how long an alert should remain in the Firing state after the alert condition stops being true. During this time, the alert enters a Recovering state, suppressing additional notifications but keeping the alert active. It helps prevent alert flapping, where alerts rapidly switch between firing and resolved due to noisy or unstable metrics.

To set up the evaluation:

1. In the Evaluation group and interval, enter a name. For example: `1m-evaluation` .

2. Choose an Evaluation interval (how often the alert are evaluated). For example, every `1m` (1 minute).

3. Set the pending period to, `0s` (zero seconds), so the alert rule fires the moment the condition is met.

4. Set Keep firing for to, `0s` , so the alert stops firing immediately after the condition is no longer true. Use this when you want alerts to be resolved as soon as the system is healthy again.

## Configure notifications

Choose the contact point where you want to receive your alert notifications.

1. Under Contact point, select Webhook from the drop-down menu.

2. Click Save rule and exit at the top right corner.

---

## Trigger and resolve an alert

Now that the alert rule has been configured, you should receive alert notifications in the contact point whenever alerts trigger and get resolved.

## Trigger an alert

Since the alert rule that you have created has been configured to always fire, once the evaluation interval has concluded, you should receive an alert notification in the Webhook endpoint.


The alert notification details show that the alert rule state is Firing , and it includes the value that made the rule trigger by exceeding the threshold of the alert rule condition. The notification also includes links to see the alert rule details, and another link to add a Silence to it.

---

## Resolve an alert

To see how a resolved alert notification looks like, you can modify the current alert rule threshold.

To edit the Alert rule:

1. Navigate to Alerting > Alert rules.

2. Click on the metric-alerts folder to display the alert that you created earlier

3. Click the edit button on the right hand side of the screen

4. Increment the Keep firing for Threshold expression to 1.

5. Click Save rule and exit.

By incrementing the threshold, the condition is no longer met, and after the evaluation interval has concluded (1 minute approx.), you should receive an alert notification with status “Resolved”.

---


## Set up the Grafana stack
To observe data using the Grafana stack, download and run the following files.

1. Clone the tutorial environment repository.
```bash
git clone https://github.com/tonypowa/grafana-prometheus-alerting-demo.git
```
2. Change to the directory where you cloned the repository:
```bash
cd grafana-prometheus-alerting-demo
```
3. Build the Grafana stack:
```bash
docker-compose build
```
4. Bring up the containers:
```bash
docker-compose up -d
```
The first time you run docker compose up -d , Docker downloads all the necessary resources for the tutorial. This might take a few minutes, depending on your internet connection.

NOTE:

If you already have Grafana, Loki, or Prometheus running on your system, you might see errors, because the Docker image is trying to use ports that your local installations are already using. If this is the case, stop the services, then run the command again.

## Use case: monitoring and alerting for system health with Prometheus and Grafana
In this use case, we focus on monitoring the system's CPU, memory, and disk usage as part of a monitoring setup. The demo app, launches a stack that includes a Python script to simulate metrics, which Grafana collects and visualizes as a time-series visualization.

The script simulates random CPU and memory usage values (10% to 100%) every 10 seconds and exposes them as Prometheus metrics.

Objective
You'll build a time series visualization to monitor CPU and memory usage, define alert rules with threshold-based conditions, and link those alerts to your dashboards to display real-time annotations when thresholds are breached.

## Step 1: Create a visualization to monitor metrics
To keep track of these metrics you can set up a visualization for CPU usage and memory consumption. This will make it easier to see how the system is performing.

The time-series visualization supports alert rules to provide more context in the form of annotations and alert rule state. Follow these steps to create a visualization to monitor the application’s metrics.

1. Log in to Grafana:

* Navigate to http://localhost:3000, where Grafana should be running.
* Username and password: admin

2. Create a time series panel:

* Navigate to Dashboards.
* Click + Create dashboard.
* Click + Add visualization.
* Select Prometheus as the data source (provided with the demo).
* Enter a title for your panel, e.g., CPU and Memory Usage.
3. Add queries for metrics:

* In the query area, copy and paste the following PromQL query:

** switch to Code mode if not already selected **
```text
flask_app_cpu_usage{instance="flask-prod:5000"}
```
* Click Run queries.

This query should display the simulated CPU usage data for the prod environment.

4. Add memory usage query:

* Click + Add query.

* In the query area, paste the following PromQL query:
```text
flask_app_memory_usage{instance="flask-prod:5000"}
```

5. Click Save dashboard. Name it: cpu-and-memory-metrics .

We have our time-series panel ready. Feel free to combine metrics with labels such as flask_app_cpu_usage{instance=“flask-staging:5000”} , or other labels like deployment .

## Step 2: Create alert rules to monitor CPU and memory usage
Follow these steps to manually create alert rules and link them to a visualization.

### Create an alert rule for CPU usage
1. Navigate to Alerts & IRM > Alerting > Alert rules from the Grafana sidebar.
2. Click + New alert rule rule to create a new alert.
### Enter alert rule name
Make it short and descriptive, as this will appear in your alert notification. For instance, cpu-usage .

### Define query and alert condition
1. Select Prometheus data source from the drop-down menu.

2. In the query section, enter the following query:

** switch to Code mode if not already selected **
```text
flask_app_cpu_usage{instance="flask-prod:5000"}
```
3. Alert condition

* Enter 75 as the value for WHEN QUERY IS ABOVE to set the threshold for the alert.

* Click Preview alert rule condition to run the queries.

The query returns the CPU usage of the Flask application in the production environment. In this case, the usage is 86.01% , which exceeds the configured threshold of 75% , causing the alert to fire.


### Add folders and labels
1. In Folder, click + New folder and enter a name. For example: system-metrics . This folder contains our alert rules.
### Set evaluation behaviour
1. Click + New evaluation group. Name it system-usage .
2. Choose an Evaluation interval (how often the alert will be evaluated). Choose 1m .
3. Set the pending period to 0s (None), so the alert rule fires the moment the condition is met (this minimizes the waiting time for the demonstration.).
4. Set Keep firing for to, 0s , so the alert stops firing immediately after the condition is no longer true.
### Configure notifications
* Select a Contact point. If you don’t have any contact points, add a Contact point.

For a quick test, you can use a public webhook from webhook.site to capture and inspect alert notifications. If you choose this method, select Webhook from the drop-down menu in contact points.

### Configure notification message
To link this alert rule to our visualization click Link dashboard and panel

* Select the folder that contains the dashboard. In this case: system-metrics
* Select the cpu-and-memory-metrics visualization
* Click confirm

You have successfully linked this alert rule to your visualization!

When the CPU usage exceeds the defined threshold, an annotation should appear on the graph to mark the event. Similarly, when the alert is resolved, another annotation is added to indicate the moment it returned to normal.

### (Optional) Step 3: Create a second alert rule for memory usage
1. Duplicate the existing alert rule (More > Duplicate), or create a new alert rule for memory usage, defining a threshold condition (e.g., memory usage exceeding 60% ).
2. Give it a name. For example: memory-usage
3. Query: flask_app_memory_usage{instance="flask-prod:5000"}
4. Link to the same visualization to obtain memory usage annotations

Check how your dashboard looks now that both alerts have been linked to your dashboard panel.

### Visualizing metrics and alert annotations

After the alert rules are created, they should appear as health indicators (colored heart icons: a red heart when the alert is in Alerting state, and a green heart when in Normal state) on the linked panel. In addition, annotations provide helpful context, such as the time the alert was triggered.

Finally, as part of the alerting process, you should receive notifications at the associated contact point.

```yaml
{
  "receiver": "prod-alerts",
  "status": "firing",
  "alerts": [
    {
      "status": "firing",
      "labels": {
        "alertname": "cpu-usage",
        "deployment": "prod-us-cs30",
        "grafana_folder": "sys-metrics",
        "instance": "flask-prod:5000",
        "job": "flask"
      },
      "annotations": {},
      "silenceURL": "http://localhost:3000/alerting/silence/new?
      "dashboardURL": "http://localhost:3000/d/dc203378-1ef9-410b-a636-b533a0dd3bd8?from=1748934450000&orgId=1&to=1748938080006",
      "panelURL": "http://localhost:3000/d/dc203378-1ef9-410b-a636-b533a0dd3bd8?from=1748934450000&orgId=1&to=1748938080006&viewPanel=2",

... }
```

Received alert notification in webhook Contact point

It’s worth mentioning that alert rules that are linked to a panel include a link to said visualization in the alert notifications. In the alert notification example above, the message includes useful information such as the summary, description, and a link to the relevant dashboard for the firing or resolved alert (i.e. dashboardURL ). This helps responders quickly navigate to the appropriate context for investigation.

You can extend this functionality by adding a custom annotation to your alert rules and creating a notification template that includes a link to a dashboard with a time range.. The URL will include a time range based on the alert’s timing—starting from one hour before the alert started (from ) to either the alert’s end time or the current time (to ), depending on whether the alert is resolved or still firing.

The final URL is constructed using a custom annotation (e.g., MyDashboardURL ) along with the from and to parameters, which are calculated in the notification template.


---

## Create alert rules with logs

To demonstrate the observation of data using the Grafana stack, download and run the following files.

Download and save a Docker compose file to run Grafana, Loki and Promtail.
```bash
wget https://raw.githubusercontent.com/grafana/loki/refs/heads/main/production/docker-compose.yaml -O docker-compose.yaml
```
Run the Grafana stack.
```bash
docker-compose up -d
```
The first time you run docker-compose up -d , Docker downloads all the necessary resources for the tutorial. This might take a few minutes, depending on your internet connection.

If you already have Grafana, Loki, or Prometheus running on your system, you might see errors, because the Docker image is trying to use ports that your local installations are already using. If this is the case, stop the services, then run the command again.

### Generate sample logs

To demonstrate how to create alert rules based on logs, you’ll use a script that generates realistic log entries to simulate typical monitoring data in Grafana. Running this script outputs logs continuously, each containing a timestamp, HTTP method (either GET or POST), status code (200 for success or 500 for failures), and request duration in milliseconds.

1. Download and save a Python file that generates logs.
```bash
wget https://raw.githubusercontent.com/grafana/tutorial-environment/master/app/loki/web-server-logs-simulator.py
```
2. Execute the log-generating Python script.
```bash
python3 ./web-server-logs-simulator.py | sudo tee -a /var/log/web_requests.log
```

### Troubleshooting the script

If you don't see the sample logs in Explore:

* Does the output file exist, check /var/log/web_requests.log to see if it contains logs.
* If the file is empty, check that you followed the steps above to create the file.
* If the file exists, verify that promtail container is running.
* In Grafana Explore, check that the time range is only for the last 5 minutes.

### Create a contact point
Besides being an open-source observability tool, Grafana has its own built-in alerting service. This means that you can receive notifications whenever there is an event of interest in your data, and even see these events graphed in your visualizations.

In this step, we set up a new contact point. This contact point uses the webhook integration. This contact point uses the webhooks integration. In order to make this work, we also need an endpoint for our webhook integration to receive the alert. We can use Webhook.site to quickly set up that test endpoint. This way we can make sure that our alert is actually sending a notification somewhere.

1. Navigate to http://localhost:3000, where Grafana should be running.
2. In another tab, go to Webhook.site.
3. Copy Your unique URL.

Your webhook endpoint is now waiting for the first request.

Next, let's configure a contact point in Grafana's Alerting UI to send notifications to our webhook endpoint.

1. Return to Grafana. In Grafana's sidebar, hover over the Alerting (bell) icon and then click Contact points.

2. Click + Create contact point.

3. In Name, write Webhook.

4. In Integration, choose Webhook.

5. In URL, paste the endpoint to your webhook endpoint.

6. Click Test, and then click Send test notification to send a test alert to your webhook endpoint.

7. Navigate back to Webhook.site. On the left side, there's now a POST / entry. Click it to see what information Grafana sent.

8. Return to Grafana and click Save contact point.

We have created a dummy Webhook endpoint and created a new Alerting contact point in Grafana. Now, we can create an alert rule and link it to this new integration.

### Create an alert rule
Next, we'll establish an alert rule within Grafana Alerting to notify us whenever alert rules are triggered and resolved.

1. In Grafana, navigate to Alerting > Alert rules.
2. Click on New alert rule.
3. Enter alert rule name for your alert rule. Make it short and descriptive as this appears in your alert notification. For instance, web-requests-logs

### Define query and alert condition
In this section, we use the default options for Grafana-managed alert rule creation. The default options let us define the query, a expression (used to manipulate the data -- the WHEN field in the UI), and the condition that must be met for the alert to be triggered (in default mode is the threshold).

1. Select the Loki datasource from the drop-down.

2. In the Query editor, switch to Code mode by clicking the button on the right.

3. Paste the query below.
```text
sum by (message)(count_over_time({filename="/var/log/web_requests.log"} != "status=200" | pattern "<_> <message> duration<_>" [10m]))
```

* This query counts the number of log lines with a status code that is not 200 (OK), then sum the result set by message type using an instant query and the time interval indicated in brackets. It uses the LogQL pattern parser to add a new label called message that contains the level, method, url, and status from the log line.

* You can use the explain query toggle button for a full explanation of the query syntax. The optional log-generating script creates a sample log line similar to the one below:
```text
2023-04-22T02:49:32.562825+00:00 level=info method=GET url=test.com status=200 duration=171ms
```
* If you're using your own logs, modify the LogQL query to match your own log message. Refer to the Loki docs to understand the pattern parser.

4. In the Alert condition section:

* Keep Last as the value for the reducer function (WHEN ), and 0 as the threshold value. This is the value above which the alert rule should trigger.

5. Click Preview alert rule condition to run the query.

* It should return alert instances from log lines with a status code that is not 200 (OK), and that has met the alert condition. The condition for the alert rule to fire is any occurrence that goes over the threshold of 0 . Since the Loki query has returned more than zero alert instances, the alert rule is Firing .


### Set evaluation behavior
An evaluation group defines when an alert rule fires, and it’s based on two settings:

`Evaluation group:` how frequently the alert rule is evaluated.
`Evaluation interval:` how long the condition must be met to start firing. This allows your data time to stabilize before triggering an alert, helping to reduce the frequency of unnecessary notifications.

To set up the evaluation:

1. In Folder, click + New folder and enter a name. For example: web-server-alerts. This folder contains our alerts.
2. In the Evaluation group, repeat the above step to create a new evaluation group. Name it 1m-evaluation.
3. Choose an Evaluation interval (how often the alert are evaluated). For example, every 1m (1 minute).
4. Set the pending period to, 0s (zero seconds), so the alert rule fires the moment the condition is met.

### Configure labels and notifications
Choose the contact point where you want to receive your alert notifications.

1. Under Contact point, select Webhook from the drop-down menu.
2. Click Save rule and exit at the top right corner.


### Trigger the alert rule
Since the Python script continues to generate log data that matches the alert rule condition, once the evaluation interval has concluded, you should receive an alert notification in the Webhook endpoint.






<!-- To demonstrate the observation of data using the Grafana stack, download and run the following files.

1. Clone the tutorial environment repository.
```bash
git clone https://github.com/grafana/tutorial-environment.git
```
2. Change to the directory where you cloned the repository:
```bash
cd tutorial-environment
```
3. Run the Grafana stack:
```bash
docker-compose up -d
```
The first time you run docker compose up -d , Docker downloads all the necessary resources for the tutorial. This might take a few minutes, depending on your internet connection.

NOTE:

If you already have Grafana, Loki, or Prometheus running on your system, you might see errors, because the Docker image is trying to use ports that your local installations are already using. If this is the case, stop the services, then run the command again. -->

## Optional Self Study
## Alert instances

An alert instance is an event that matches a metric returned by an alert rule query.

Let's consider a scenario where you're monitoring website traffic using Grafana. You've set up an alert rule to trigger an alert instance if the number of page views exceeds a certain threshold (more than `1000` page views) within a specific time period, say, over the past `5` minutes.

If the query returns more than one time-series, each time-series represents a different metric or aspect being monitored. In this case, the alert rule is applied individually to each time-series.

## Create notification policies

Create a notification policy if you want to handle metrics returned by alert rules separately by routing each alert instance to a specific contact point.

1. Visit http://localhost:3000, where Grafana should be running

2. Navigate to Alerts & IRM > Alerting > Notification policies.

3. In the Default policy, click + New child policy.

4. In the field Label enter `device` , and in the field Value enter `desktop` .

5. From the Contact point drop-down, choose Webhook.

If you don’t have any contact points, add a Contact point.

6. Click Save Policy.

This new child policy routes alerts that match the label device=desktop to the Webhook contact point.

7. Repeat the steps above to create a second child policy to match another alert instance. For labels use: `device=mobile` . Use the Webhook integration for the contact point. Alternatively, experiment by using a different Webhook endpoint or a different integration.

## Create an alert rule that returns alert instances

The alert rule that you are about to create is meant to monitor web traffic page views. The objective is to explore what an alert instance is and how to leverage routing individual alert instances by using label matchers and notification policies.

## Create an alert rule

1. Navigate to Alerts & IRM > Alerting > Alert rules.
2. Click + New alert rule.
### Enter an alert rule name
Make it short and descriptive as this will appear in your alert notification. For instance, `web-traffic` .

## Define query and alert condition
In this section, we use the default options for Grafana-managed alert rule creation. The default options let us define the query, a expression (used to manipulate the data -- the WHEN field in the UI), and the condition that must be met for the alert to be triggered (in default mode is the threshold).

Grafana includes a test data source that creates simulated time series data. This data source is included in the demo environment for this tutorial. If you're working in Grafana Cloud or your own local Grafana instance, you can add the data source through the Connections menu.

1. Select TestData data source from the drop-down menu.

2. From Scenario select CSV Content.

3. Copy in the following CSV data:
```text
device,views
desktop,1200
mobile,900
```
The above CSV data simulates a data source returning multiple time series, each leading to the creation of an alert instance for that specific time series. Note that the data returned matches the example in the Alert instance section.

4. In the Alert condition section:

* Keep Last as the value for the reducer function (WHEN ), and IS ABOVE `1000` as the threshold value. This is the value above which the alert rule should trigger.
* Click Preview alert rule condition to run the queries.

It should return two series: `desktop` in `Firing` state, and `mobile` in ``Normal` state. The values 1 , and 0 mean that the condition is either true or false .

## Add folders and labels
1. In Folder, click + New folder and enter a name. For example: web-traffic-alerts . This folder contains our alert rules.
## Set evaluation behavior
In the life cycle of alert instances, when an alert condition (threshold) is not met, the alert instance state is Normal. Similarly, when the condition is breached (for longer than the pending period, which in this tutorial will be 0), the alert instance state switches back to Alerting, which means that the alert rule state is Firing, and a notification is sent.

To set up evaluation behavior:

1. In the Evaluation group and interval, enter a name. For example:  1m-evaluation .

2. Choose an Evaluation interval (how often the alert will be evaluated). Choose 1m .

3. Set the pending period to 0s (zero seconds), so the alert rule fires the moment the condition is met.

4. Set Keep firing for to, 0s , so the alert stops firing immediately after the condition is no longer true.

## Configure notifications

In this section, you can select how you want to route your alert instances. Since we want to route by notification policy, we need to ensure that the labels match the alert instance.

1. Toggle the Advanced options button to display matching Notification policies.

2. Click Preview routing. Based on the existing labels, you should see a preview of what policies are matching with the alerts. There should be two alert instances matching the labels that were previously setup in each notification policy: device=desktop , device=mobile .

These types of labels are generated by the data source query and they can be leveraged to match our notification policies without needing to manually add them to the alert rule.

Even if both labels match the policies, only the alert instance in Firing state produces an alert notification.

3. Click Save rule and exit.

Now that we have set up the alert rule, it’s time to check the alert notification.

## Receive alert notifications

Now that the alert rule has been configured, you should receive alert notifications in the contact point whenever the alert triggers and gets resolved. In our example, each alert instance should be routed separately as we configured labels to match notification policies. Once the evaluation interval has concluded (1m), you should receive an alert notification in the Webhook endpoint.

The alert notification details show that the alert instance corresponding to the website views from desktop devices was correctly routed through the notification policy to the Webhook contact point. The notification also shows that the instance is in Firing state, as well as it includes the label device=desktop , which makes the routing of the alert instance possible.

Feel free to change the CSV data in the alert rule to trigger the routing of the alert instance that matches the label device=mobile .
