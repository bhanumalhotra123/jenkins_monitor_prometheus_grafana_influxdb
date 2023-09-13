# Monitoring With Grafana | Monitoring Production grade Jenkins using Prometheus, Grafana & InfluxDB

Created an EC2 instance and setup security group inbound rules.

ssh into the machine


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

Prometheus

cd /home/ubuntu

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

 Additional configuration options may be added as needed.


docker run -d --name prometheus-container -v /home/ubuntu/prometheus.yml:/etc/prometheus/prometheus.yml -e TZ=UTC -p 9090:9090 ubuntu/prometheus:2.33-22.04_beta


2. Grafana
docker run -d --name=grafana -p 3000:3000 grafana/grafana:8.5.5


3. InfluxDb
docker run -d -p 8086:8086 --name influxdb2 influxdb:1.8.6-alpine

![m1](https://github.com/bhanumalhotra123/jenkins_monitor_prometheus_grafana_influxdb/assets/144083659/5a122e19-d43a-483a-afa2-59a2b3c8bad9)


Jenkins UI:
http://34.202.233.172:8080/ 

jenkins@d076b2553ff9:/$ cat /var/jenkins_home/secrets/initialAdminPassword
4ef4936d283f43a3a8275aaaf03997a7

Grafana UI

http://34.202.233.172:3000/login

u - admin 
p - admin

set your own Password 



For fetching the data from jenkins to influxDB:

Connect to container and execute below Influx Commands

![m2](https://github.com/bhanumalhotra123/jenkins_monitor_prometheus_grafana_influxdb/assets/144083659/43db8dc6-6603-425f-bd4e-f4f6ff7c963f)


CREATE DATABASE "jenkins" WITH DURATION 1825d REPLICATION 1 NAME "jenkins-retention"

This command configures InfluxDB to establish a new database called "jenkins." Within this database, it defines a data retention policy that specifies a data retention period of 5 years. This retention policy essentially means that data older than 5 years will be automatically removed from the database to conserve storage space.

Furthermore, the command sets the replication factor to "1," which means that there will be one copy of the data kept for redundancy and fault tolerance purposes.

Lastly, it provides a name for this retention policy, which is "jenkins-retention." This naming convention is valuable for managing and categorizing different data retention policies within the database.




Now go to jenkins dashboard:

Install the following plugins:

![m3](https://github.com/bhanumalhotra123/jenkins_monitor_prometheus_grafana_influxdb/assets/144083659/06232e74-3d8a-4a1b-a9dc-81981a88399e)


RESTART THE JENKINS
http://34.202.233.172:8080/restart

docker restart jenkins-container-id


To store the pipeline data:

Dashboard > Manage Jenkins > Configure System > InfluxDB targets

Description: InfluxDB
URL
DATABASE: jenkins
RETENTION POLICY: jenkins-retention

Test Connection: success!

Select
Job schedule time as timestamp 
global listener(you can filter as well)


![m5](https://github.com/bhanumalhotra123/jenkins_monitor_prometheus_grafana_influxdb/assets/144083659/5e331a06-b1c3-4ead-be4b-1668bc5658b5)

![m4](https://github.com/bhanumalhotra123/jenkins_monitor_prometheus_grafana_influxdb/assets/144083659/4c8867c7-7949-47e9-94e5-eeba97ac7574)

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

Now /home/ubuntu/prometheus.yml
![m6](https://github.com/bhanumalhotra123/jenkins_monitor_prometheus_grafana_influxdb/assets/144083659/4f91aad4-5ca6-4212-8483-4ec526e7465c)


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

