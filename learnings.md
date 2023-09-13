Monitoring With Grafana | Monitoring Production grade Jenkins using Prometheus, Grafana & InfluxDB

Created an EC2 instance.


Installed Docker:

sudo apt-get update
sudo apt-get install apt-transport-https ca-certificates curl gnupg-agent software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-key fingerprint 0EBFCD88
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io ( to install latest version )
sudo docker run hello-world

Jenkins container:

docker run -d -p 8080:8080 jenkins/jenkins:lts-jdk11


To bring up monitoring stack using docker

1. Prometheus

cd /home/bhanu

vi prometheus.yml
root@ip-172-31-48-76:/home/bhanu# cat prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting: 
  alertmanagers:
  -   static_configs:
      - targets: 
        #['localhost:8080'] # Replace with the actual service address and port

rule_files:
  - 'alerts.yml'



scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090'] # Prometheus itself

# Additional configuration options may be added as needed.




docker run -d --name prometheus-container -v /home/bhanu/prometheus.yml:/etc/prometheus/prometheus.yml -e TZ=UTC -p 9090:9090 ubuntu/prometheus:2.33-22.04_beta
Note:- For more info prometheus

2. Grafana
docker run -d --name=grafana -p 3000:3000 grafana/grafana:8.5.5
Note:- For more info Grafana

3. InfluxDb
docker run -d -p 8086:8086 --name influxdb2 influxdb:1.8.6-alpine


root@ip-172-31-48-76:/home/bhanu# docker ps
CONTAINER ID   IMAGE                               COMMAND                  CREATED             STATUS             PORTS                                                  NAMES
e4f572c20dbd   influxdb:1.8.6-alpine               "/entrypoint.sh infl…"   7 minutes ago       Up 7 minutes       0.0.0.0:8086->8086/tcp, :::8086->8086/tcp              influxdb2
eefeddc2b88f   grafana/grafana:8.5.5               "/run.sh"                7 minutes ago       Up 7 minutes       0.0.0.0:3000->3000/tcp, :::3000->3000/tcp              grafana
7b70d91f5cee   ubuntu/prometheus:2.33-22.04_beta   "/usr/bin/prometheus…"   9 minutes ago       Up 9 minutes       0.0.0.0:9090->9090/tcp, :::9090->9090/tcp              prometheus-container
d076b2553ff9   jenkins/jenkins:lts-jdk11           "/usr/bin/tini -- /u…"   About an hour ago   Up About an hour   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp, 50000/tcp   quirky_hugle






Jenkins UI:
http://34.202.233.172:8080/ 

jenkins@d076b2553ff9:/$ cat /var/jenkins_home/secrets/initialAdminPassword
4ef4936d283f43a3a8275aaaf03997a7

Grafana UI

http://34.202.233.172:3000/login

u - admin 
p - admin

set your own Password 



For fetching the data from jenkins to influxDB



Connect to container and execute below Influx Commands

root@ip-172-31-48-76:/home/bhanu# docker exec -it e4f572c20dbd  bash
bash-5.0# influx
Connected to http://localhost:8086 version 1.8.6
InfluxDB shell version: 1.8.6
> SHOW DATABASES
name: databases
name
----
_internal
> CREATE DATABASE "jenkins" WITH DURATION 1825d REPLICATION 1 NAME "jenkins-retention"
> SHOW DATABASES
name: databases
name
----
_internal
jenkins
> USE jenkins
Using database jenkins
> 


Now go to jenkins dashboard:

Install the following plugins:

InfluxDBVersion
3.5
Build Reports Other Post-Build Actions
This plugin allows sending build results to InfluxDB. It was inspired by https://github.com/jrajala-eficode/jenkins-ci.influxdb-plugin
2 mo 13 days ago

Job and Stage monitoringVersion
3.6.2
Watches pipeline jobs and provides job and stage stats such as time and pass/fail. Can be configured to update GitHub commit status (one status per stage) and send stats to an InfluxDB instance, or StatsD collector, for build health monitoring of job/stage timing and success rate.
3 yr 6 mo ago


Prometheus metricsVersion
2.3.1
monitoring Miscellaneous
Jenkins Prometheus Plugin expose an endpoint (default /prometheus) with metrics where a Prometheus Server can scrape.

RESTART THE JENKINS
http://34.202.233.172:8080/RESTART

docker restart jenkins

To store the pipeline data:

Dashboard > Manage Jenkins > Configure System > InfluxDB targets

Description: InfluxDB
URL
DATABASE: jenkins
RETENTION POLICY: jenkins-retention

Test Connection: success!

select
job schedule time as timestamp 
global listener(you can filter as well)


How much time each stage is taking? How much time each pipeline is taking?
Dashboard > Manage Jenkins > Configure System > Autostatus Config > Send to InfluxDB

URL
DATABASE: jenkins
RETENTION POLICY: jenkins-retention

Save!



When you install prometheus plugin:

http://34.202.233.172:8080/prometheus/
Prometheus can scrap the data from here anytime it wants.

You don't have to do anything more for prometheus in jenkins
But you have to setup jenkins in prometheus

Now /home/bhanu/prometheus.yml


root@ip-172-31-48-76:/home/bhanu# cat prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting: 
  alertmanagers:
  -   static_configs:
      - targets: 
        #['localhost:8080'] # Replace with the actual service address and port

rule_files:
  - 'alerts.yml'



scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090'] # Prometheus itself

  - job_name: 'jenkins'                                   #updated the location from where prometheus will scrap jenkins data.
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['34.202.233.172:8080'] # Prometheus itself   
# Additional configuration options may be added as needed.

Restart Prometheus container
docker restart id

Prometheus UI:
http://34.202.233.172:9090/targets

Check targets here: Jenkins is added


Now we have configured jenkins and prometheus 
also jenkins and influxDB

Now?
Let's create some data


Create folders and jobs within them:

Folder                    jobs     
call-booking-user         user-api,user-ui
call-booking-admin        admin-api,admin-ui

In the end of each console-output of the job:

[Pipeline] // node
[Pipeline] End of Pipeline
[InfluxDB Plugin] Collecting data...
[InfluxDB plugin] Metrics plugin data found. Writing to InfluxDB...
[InfluxDB Plugin] Publishing data to target 'InfluxDB' (url='http://34.202.233.172:8086', database='jenkins')
[InfluxDB Plugin] Completed.
Finished: SUCCESS

Sends the data to influxDB

