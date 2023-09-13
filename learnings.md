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
Let's create some data:


Create folders and jobs within them:

Folder                    jobs     
call-booking-user         user-api,user-ui
call-booking-admin        admin-api,admin-ui

We can use hello world script for now:


pipeline {
    agent any // This means the pipeline can run on any available agent (slave/executor)

    stages {
        stage('Hello') {
            steps {
                // This is where you put your actual build or script steps
                echo 'Hello, World!' // This will print "Hello, World!" to the console
            }
        }
    }
}



In the end of each console-output of the job:

![m7](https://github.com/bhanumalhotra123/jenkins_monitor_prometheus_grafana_influxdb/assets/144083659/f6d3b97b-64df-474d-a26c-4779acf58f48)

Sends the data to influxDB:

![m8](https://github.com/bhanumalhotra123/jenkins_monitor_prometheus_grafana_influxdb/assets/144083659/f53585d7-c109-4a62-a350-e4f14519b9e1)


After this we have to add prometheus and InfluxDB as datasources in Grafana
In grafana > Configuration button(left side) > Add datasource:

![m9](https://github.com/bhanumalhotra123/jenkins_monitor_prometheus_grafana_influxdb/assets/144083659/b6138480-2c32-4c6e-9eec-95e897c54aa8)



Now to create dashboard in grafana:
On the left menu select + button > Create Dashboard 
Now we can create as many panels as we want here:


![m10](https://github.com/bhanumalhotra123/jenkins_monitor_prometheus_grafana_influxdb/assets/144083659/7c18e2ce-96a5-4139-8b4b-1731c889b7d1)


1st Panel: Jenkins Health

UP or DOWN

To create a panel > Add new panel
Select the data source: Prometheus
Provide prometheus query (to get this we can search for up in prometheus queries and it will give us for targets jenkins and itself, pick the jenkins one)
Select Type of Panel: Stat
When we put the query in grafana, we get the value 1.
Under Stat Panel > Value Mapping > Here Map 1 to UP and colour Green and 0 to DOWN and Red colour.
Stat Panel > Stat Styles > Color mode > Background (we want the background colors not the color for value)

2nd Panel:
To create a panel > Add new panel.
Select the data source: Prometheus.
Provide prometheus query (In  prometheus queries: jenkins_executor_count_value, jenkins_node_count_value, jenkins_queue_size_value these will give you results
which we can place in grafana).
Select Type of Panel: Time Series

![m101](https://github.com/bhanumalhotra123/jenkins_monitor_prometheus_grafana_influxdb/assets/144083659/84d4b41a-a19c-4e01-8185-21a2608ee8f1)


To test this we added an agent name to one of the created jobs but that agent didn't exist. So the pipeline kept looking on for the agent and the job queue increased.

![m102](https://github.com/bhanumalhotra123/jenkins_monitor_prometheus_grafana_influxdb/assets/144083659/229e71b6-7d62-4175-b4e1-ad0dd0cf832b)


Creating Drop Downs in Grafana:
Go to Dashboard Settings on top menu > Variables > Add Variable

Name: Folder Label: folder DataSource: InfluxDB  


Query: SHOW TAG VALUES FROM job WITH KEY = "owner"

In InfluxDB, data is organized into "measurement" and "tag" sets. A "measurement" is like a table in a traditional database, and "tags" are key-value pairs associated with data points in that measurement. 

SHOW TAG VALUES: This is a command in InfluxDB used to retrieve the unique values associated with tags in a measurement.

FROM job: Here, "job" is the name of the measurement you want to work with. Think of it as the dataset you want to query.

WITH KEY="owner": This part specifies that you want to focus on the tag with the key "owner" within the "job" measurement. Tags are metadata associated with data points, and in this case, you're interested in the "owner" tag specifically.

So, the query is asking InfluxDB to show you all the unique values of the "owner" tag within the "job" measurement. It helps you understand who the different owners are for the data points in that measurement.


Name: Job  Label: job  DataSource: InfluxDB 

Query: SHOW TAG VALUES FROM job WITH KEY = repo WHERE "owner" =~ /^($folder)$/

Here ~ /^($folder)$/ this part grafana will replace with whatever i select in Dropdown

![m104](https://github.com/bhanumalhotra123/jenkins_monitor_prometheus_grafana_influxdb/assets/144083659/71edee93-64a7-43dc-8c9f-cd7616021d3f)


![m103](https://github.com/bhanumalhotra123/jenkins_monitor_prometheus_grafana_influxdb/assets/144083659/f14a96b3-306c-4e23-bccf-683d42e2301a)

Remember to select all include option



Overall Panel:

Add new panel:

Type: Pie Chart (Donut)
Data Source: InfluxDB  (all the pipeline data we are getting from influxdb)
Title: Overall

Queries:

1. Success
SELECT count(build_number) FROM "jenkins_data" WHERE ("project_name" =~ /^(?i)$job$/ AND "project_path" =~ /.*(?i)$folder.*$/) AND ("build_result" = 'SUCCESS' OR "build_result" = 'CompletedSuccess' ) AND $timeFilter

Explanation:

SELECT count(build_number): This part of the query instructs InfluxDB to count the occurrences of the "build_number" field in the "jenkins_data" measurement. It's essentially asking how many records meet the criteria specified in the query.

FROM "jenkins_data": This specifies the measurement from which you want to retrieve data. In this case, it's "jenkins_data."

WHERE: This clause allows you to filter the data based on specific conditions.

"project_name" =~ /^(?i)$job$/: This condition is checking if the "project_name" field matches the value stored in the variable $job. The =~ operator is used for regular expression matching. ^(?i)$job$ is a regular expression that matches the value in $job case-insensitively. So, it's filtering records where "project_name" matches the value in $job.

"project_path" =~ /.*(?i)$folder.*$/: Similar to the previous condition, this checks if the "project_path" field matches the value stored in the variable $folder, again case-insensitively. The .* before and after $folder means that it can be part of a larger string, and (?i) makes the matching case-insensitive.

"build_result" = 'SUCCESS' OR "build_result" = 'CompletedSuccess': This condition checks if the "build_result" field is either 'SUCCESS' or 'CompletedSuccess'. Records meeting either of these conditions are included in the count.

$timeFilter: This appears to be a placeholder for a time-based filter that might be set elsewhere in your query or code. In InfluxDB, you can specify time-based filters to retrieve data for a specific time range.


2. Failure
SELECT count(build_number) FROM "jenkins_data" WHERE ("project_name" =~ /^(?i)$job$/ AND "project_path" =~ /.*(?i)$folder.*$/) AND ("build_result" = 'FAILURE' OR "build_result" = 'CompletedError' ) AND $timeFilter 


3. Aborted
SELECT count(build_number) FROM "jenkins_data" WHERE ("project_name" =~ /^(?i)$job$/ AND "project_path" =~ /.*(?i)$folder.*$/) AND ("build_result" = 'ABORTED' OR "build_result" = 'Aborted' ) AND $timeFilter 

4. Unstable
SELECT count(build_number) FROM "jenkins_data" WHERE ("project_name" =~ /^(?i)$job$/ AND "project_path" =~ /.*(?i)$folder.*$/) AND ("build_result" = 'UNSTABLE' OR "build_result" = 'Unstable' ) AND $timeFilter

![m105](https://github.com/bhanumalhotra123/jenkins_monitor_prometheus_grafana_influxdb/assets/144083659/8a90b927-90e4-425c-bf8c-59f7b1a39c63)


For testing purpose: I  edited script of 1 job and put something random so it fails, aborted you can do by stopping any build in between , for unstable i used 

pipeline {
    agent any 
    stages {
        stage('Stage 1') {
            steps {
                script{
                echo 'Hello world!'
                currentBuild.result='Unstable'
                }
            }
        }
    }
}


I also created a table for it:
Under Pie Chart > All > Legend  > Legend Mode > Table
Under Pie Chart > All > Legend  > Legend Placement > Right
Under Pie Chart > All > Legend  > Legend Values > Select Percent and Value

Now for fixing colours >
Under Pie Chart > Overrides>  Add field Override > Field with Name > Success > Add override property > Green colour
Same for others.

![m106](https://github.com/bhanumalhotra123/jenkins_monitor_prometheus_grafana_influxdb/assets/144083659/3ccafe42-3976-4a23-b947-d9620beb09e3)



Panel: No.of pipelines ran
Source: InfluxDB
Type: Stat
Query: select count(DISTINCT project_name) FROM jenkins_data WHERE ("project_name" =~ /^(?i)$job$/ AND "project_path" =~ /.*(?i)$folder.*$/) AND $timeFilter 

Can add colors based on threshold


Panel: Total number of builds (Can duplicate the previous one and make changes to it) 
Query: SELECT count(build_number) FROM "jenkins_data" WHERE ("project_name" =~ /^(?i)$job$/ AND "project_path" =~ /.*(?i)$folder.*$/) AND $timeFilter 


Panel: Average Build Time
Type: Stat
DataSource: InfluxDB
Query:
select build_time/1000 FROM jenkins_data WHERE ("project_name" =~ /^(?i)$job$/ AND "project_path" =~ /.*(?i)$folder.*$/) AND $timeFilter 


