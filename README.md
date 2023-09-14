Monitoring With Grafana | Monitoring Production grade Jenkins using Prometheus, Grafana & InfluxDB

(learnings.md have all the steps with images of implementing this.)

![m111](https://github.com/bhanumalhotra123/jenkins_monitor_prometheus_grafana_influxdb/assets/144083659/b7a79988-8cc0-4bb2-8fed-83b57162ed27)

![m112](https://github.com/bhanumalhotra123/jenkins_monitor_prometheus_grafana_influxdb/assets/144083659/bb51c387-8b42-4d1a-b681-fa2745b0892c)





Jenkins:

Jenkins is an open-source automation server that facilitates continuous integration and continuous delivery (CI/CD) by automating building, testing, and deploying software.


Prometheus:

Prometheus is an open-source monitoring and alerting toolkit designed for reliability and scalability. It collects and stores time-series data from various sources, including applications and systems.


Grafana:

Grafana is an open-source platform for visualizing and monitoring time-series data. It provides customizable dashboards and supports various data sources, including Prometheus and InfluxDB.
InfluxDB:

InfluxDB is a high-performance, open-source time-series database designed for storing and querying large volumes of time-stamped data, making it ideal for metrics and monitoring data.


Connection:

Jenkins can export metrics related to builds and jobs in a format Prometheus can understand. Prometheus scrapes these metrics from Jenkins at regular intervals using HTTP endpoints.
Prometheus stores the scraped metrics and provides a querying and alerting system. It can also be configured to send alerts to external systems.

Grafana connects to Prometheus (or other data sources like InfluxDB) to create visual dashboards that display metrics in a user-friendly way. It allows users to customize and visualize data collected by Prometheus.

InfluxDB can be used as a long-term storage solution for metrics data collected by Prometheus or other monitoring tools. Grafana can also connect to InfluxDB as a data source to create dashboards.



Steps:

1. Install Docker

2. Run Prometheus, Grafana, InfluxDB and jenkins in docker containers.

3. Install required plugins: Influxdb and prometheus related

4. Configure prometheus and Influxdb to fetch data from jenkins.

5. Creating folders and jobs in jenkins.

6. Add data sources in Grafana

7. Build the panels in the dashboard
