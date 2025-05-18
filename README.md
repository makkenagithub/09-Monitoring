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

http server is prometheus server using which we open console. service discovery is scrape config component. So service discovery discovers whatt all nodes are under monitoring of prometheus. TSD is time-series DB.

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
Then open promethues cosnole using url http://<public-ip-of-prometheus-ec2>:9090

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
In prometheus console, give up and see, node exporter also added there





