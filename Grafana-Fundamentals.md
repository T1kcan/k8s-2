# Set up the sample application
This tutorial uses a sample application to demonstrate some of the features in Grafana. To complete the exercises in this tutorial, you need to download the files to your local machine.

In this step, you’ll set up the sample application, as well as supporting services, such as Loki.

Note: Prometheus, a popular time series database (TSDB), has already been configured as a data source as part of this tutorial.

Clone the github.com/grafana/tutorial-environment repository.

```bash
git clone https://github.com/grafana/tutorial-environment.git
```
Change to the directory where you cloned this repository:

```bash
cd tutorial-environment
```
Make sure Docker is running:

```bash
docker ps
```
No errors means it is running. If you get an error, then start Docker and then run the command again.

Start the sample application:

```bash
docker-compose up -d
```
The first time you run docker-compose up -d, Docker downloads all the necessary resources for the tutorial. This might take a few minutes, depending on your internet connection.

Note: If you already have Grafana, Loki, or Prometheus running on your system, then you might see errors because the Docker image is trying to use ports that your local installations are already using. Stop the services, then run the command again.

Ensure all services are up-and-running:

```bash
docker-compose ps
```
In the State column, it should say Up for all services.

Browse to the sample application on http://localhost:8081.