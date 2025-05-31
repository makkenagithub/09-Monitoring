# 09-Monitoring
1. Blackbox testing - we do not know what is inside - end users can test it without knowing internal details. (like FB website working or not login working or not etc)
2. Whitebox testing - we know what is inside - internal user test this. (like CPU, RAM, network requests etc)

Google set some standarrds for monitoring, Industry follows these standards  https://sre.google/sre-book/monitoring-distributed-systems/

We are good if we monitor 4 golden signas given by google - 
The four golden signals of monitoring are latency, traffic, errors, and saturation. If you can only measure four metrics of your user-facing system, focus on these four.

1. Latency - how fast our system is responding to requests
2. Traffic - number of requests to system
3. Errors - monitor for 5XX errors
4. Saturation - measure system resources like cpu, ram etc

We use prometheus and grafana for some metrics and ELK for some metrics.

Prometheus:

![image](https://github.com/user-attachments/assets/8d344492-fff2-4f24-a443-8f51e1934a22)

http server is prometheus server using which we open console. service discovery is scrape config component. So service discovery discovers whatt all nodes are under monitoring of prometheus. TSD is time-series DB. Grafana is for visualisation purpose.

We need to install the agent software "node exporter" in the instances(ec2) which needs to be monitored. Node exporter passes the information to prometheus server continuously. All that information will be stored in a database names TDS (time-series data base) in prometheus server

Configuring prometheus:
download prometheus for linux amd: https://prometheus.io/download/

https://github.com/prometheus/prometheus/releases/download/v3.4.0/prometheus-3.4.0.linux-amd64.tar.gz

create an ec2 with t3.medium and 20GB space
```
sudo su -
cd /opt
wget https://github.com/prometheus/prometheus/releases/download/v3.4.0/prometheus-3.4.0.linux-amd64.tar.gz
tar -xvf prometheus-3.4.0.linux-amd64.tar.gz
mv prometheus-3.4.0.linux-amd64 prometheus
cd prometheus
ls -lrt
```
Now create a prometheus systemctl service 
```
vim /etc/systemd/system/prometheus.service
```
and enter below config info
```
[Unit]
Description=Prometheus Server

[Service]
ExecStart=/opt/prometheus/prometheus --config.file=/opt/prometheus/prometheus.yml

[Install]
WantedBy=multi-user.target
```
```
systemctl start prometheus
systemctl status prometheus
systemctl enable prometheus
netstat -lntp
```
Prometheus runs on port 9090.
Then open promethues cosnole using url http://(public-ip-of-prometheus-ec2):9090

Prometheus is a time series DB. Its simple but very powerful.


Next see how prometheus connects to an ec2 and fecth information?

Create an ec2 with t3.micro and install an agent node exporter and connect to prometheus server.
https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-amd64.tar.gz
https://prometheus.io/download/

```
sudo su -
cd /opt
wget https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-amd64.tar.gz
tar -xvf node_exporter-1.9.1.linux-amd64.tar.gz
mv node_exporter-1.9.1.linux-amd64 node_exporter
cd node_exporter
ls -lrt
```
Then configure node exporter service in /etc/systemd. There is no configuration file in node exporter
```
vim /etc/systemd/system/node_exporter.service
```
```
[Unit]
Description=Node exporter

[Service]
ExecStart=/opt/node_exporter/node_exporter

[Install]
WantedBy=multi-user.target
```
```
systemctl restart node_exporter
systemctl status node_exporter
systemctl enable node_exporter
netstat -lntp
```
node exporter runs on 9100  port.

Now we created prometheus server and a node exporter. But they are not connected yet

Connecting node exporter to prometheus:
Goto prometheus server and edit the prometheus config file with node exporer ip and port, in the scrape config field.
```
vi /opt/prometheus/prometheus.yml
```
```
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
        labels:
          name: "prometheus"

  - job_name: "node-1"
    static_configs:
      - targets: ["172.31.42.27:9100"]
        labels:
          name: "node-1"
```
```
systemctl restart prometheus
systemctl status prometheus
```
In prometheus console, give up and see, node exporter also added there. Also we can see lot of metrics exists for monitoring here.

Visualisation in prometheus is not good. Fot visuvalisation we use Grafana. So we install grafana in prometheus server.

Grafana:
https://grafana.com/docs/grafana/latest/setup-grafana/installation/redhat-rhel-fedora/
```
sudo su -
wget -q -O gpg.key https://rpm.grafana.com/gpg.key
sudo rpm --import gpg.key
```
Create /etc/yum.repos.d/grafana.repo with the following content:
```
vi /etc/yum.repos.d/grafana.repo
```
```
[grafana]
name=grafana
baseurl=https://rpm.grafana.com
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://rpm.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
```
```
sudo dnf install grafana
```
start grafana server :   https://grafana.com/docs/grafana/latest/setup-grafana/start-restart-grafana/
```
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl status grafana-server
sudo systemctl enable grafana-server.service
netstat -lntp
```
grafana works on port 3000
open grafana console with url http://(public-ip):3000
default id/pwd = admin/admin

Grafana does not has any DB in it. We need to connect it to Prometheus server. Graafana can be connected to any DB (not only prometheus db), so that it can fetch the data and display.

grafana home -> connections -> add new connection -> prometheus -> add new data source -> connection url is http://localhost:9090 ( as prometheus and grafana are in same server) -> save & test.

Its better to install prometheus and grafana in same server.

Create dashboard:

grafana home -> dashboard -> create dash board -> add visualisation -> select prometheus -> code -> (enter the queries in matrics browser field like up etc) -> run queries -> save dashboard
then it displays the queries in visual form. On the right side we choose visualisation form like timeseries graph, bar chart etc

In the scrape config filed, we added node exporter with IP and port, so that node exporter send data to prometheus. But in case of k8s cluster, worker node IPs changes frequently , so can't add IPs in scrape config field. This can be solved using tags, called dynamic scrapping

dynamic scrapping:
1. our target ec2's should have node exporter installed.
2. prometheus server should have permission to describe ec2 instances. (whatevr the tags/availability zone filters we added in scrape config field, prometheus server should have permission to desrcribe that). In IAM policies, we have DescribeInstances policy. (1. create a ploicy with instance describe permission. 2. Then create an IAM rule and attach the policy to this rule. 3. Attach this rule to prometheus ec2 server.)
3. we should filter the target instances based on the tags, region, az etc.
4. 
ec2_sd_config
https://aws.amazon.com/blogs/mt/automating-amazon-ec2-instance-monitoring-with-prometheus-ec2-service-discovery-and-aws-distro-for-opentelemetry/
```
# ADOT Collector configuration to scrape targets from specific Availability Zone "ap-south-1a"
---
ec2_sd_configs:
  - region: ap-south-1
    port: 9100
    filters:
      - name: __meta_ec2_availability_zone
        values:
          - ap-south-1a
relabel_configs:
  - source_labels:
      - __meta_ec2_instance_id
    target_label: instance_id
```
https://prometheus.io/docs/prometheus/latest/configuration/configuration/#ec2_sd_config

Alert management:

Alert rules:
https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/

Once we get the alerts, what we have to do ? Its done with alert manager. 
Aletr manager will take care of what to do after getting the alert.

https://github.com/prometheus/alertmanager/releases/download/v0.28.1/alertmanager-0.28.1.linux-amd64.tar.gz
https://prometheus.io/download/

```
cd /opt
wget https://github.com/prometheus/alertmanager/releases/download/v0.28.1/alertmanager-0.28.1.linux-amd64.tar.gz
tar -xvf alertmanager-0.28.1.linux-amd64.tar.gz
mv alertmanager-0.28.1.linux-amd64 alertmanager
```
Then configure alert manager service in /etc/systemd. There is no configuration file in node exporter
```
vim /etc/systemd/system/alertmanager.service
```
```
[Unit]
Description=Alert manager

[Service]
ExecStart=/opt/alertmanager/alertmanager --config.file=/opt/alertmanager/alertmanager.yml

[Install]
WantedBy=multi-user.target
```
```
systemctl start alertmanager
netstat -lntp
systemctl enable alertmanager
```
alert managers opened 2 ports 9093, 9094. we can use 9093 port in prometheus.yaml config file.

open alert manager console with http://(public-ip):9093

Metrics:
Two type of metrics.
1. counter - It will only increase. (like no. of KMs travelled by car)
2. gauge - I can increase / decrease (like CPU utilisation)

Import grafana dash boards from outside:
https://grafana.com/grafana/dashboards/1860-node-exporter-full/

We can import the grafana dashboard from ouside also. We can give the dashboard ID in the grafana dashboard -> import dashboard -> give ID (say 1860 for node exporter full) -> load -> import.

It displays dashboard with all items for our EC2 instances which we need to monitor.


### ELK (Elastic search, Logstash and Kibana)
Usually a separate will be there for ELK. Its b big think.

Elastic search is a database for logs. 

Our worker nodes will be having application logs or system logs etc. Certain agents will run in worker nodes as sidecars. Examples for these agents are Filebeat, fluentd etc. Filebeat is a popular sidecar agent. These sidecars collect the logs from the path given to them like eg: /var/log/nginx/access.log 

Whenevr there are new entries in the log path, sidecars will take them and push to elastic search. Inbetween agents and elastic search we have a component called logstash. Kibana is to show the logs from elastic search in UI to users. Kibana acts as UI. 

ELK is a big thing. To practice this elk, we need t3.medium instance and 50GB storage while creating ec2. 

In monitoring we saw on saturation. Now by using ELK, we can see remaining things like Latency, Errors, Traffic by analysing ELK logs.

Filebeat is a popular agent for log shipping to elastic search. We call filebeat as log shipper. 




