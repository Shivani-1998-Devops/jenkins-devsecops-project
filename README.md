DevSecOps CI/CD Setup Guide
Overview

This guide explains how to set up a complete DevSecOps CI/CD environment using:

Jenkins
SonarQube
Prometheus
Grafana
Nexus Repository
Docker
Trivy
OWASP ZAP
Infrastructure Requirements
Jenkins / Monitoring Server
Resource	Specification
Instance Type	c5.xlarge
CPU	4 vCPU
RAM	8 GB
Storage	30 GB EBS
Nexus Server
Resource	Specification
Instance Type	t3.medium
CPU	2 vCPU
RAM	4 GB
Security Group Ports
Jenkins Server Security Group
Service	Port
SSH	22
Jenkins	8080
SonarQube	9000
Prometheus	9090
Grafana	3000
Nexus Repository	8081
Node Exporter	9100
Nexus Server Security Group
Service	Port
SSH	22
Nexus Repository	8081
Node Exporter	9100
System Update & Common Packages
sudo apt update
sudo apt upgrade -y

sudo apt install -y \
bash-completion \
wget \
git \
zip \
unzip \
curl \
jq \
net-tools \
build-essential \
ca-certificates \
apt-transport-https \
gnupg \
fontconfig

Reload bash completion:

source /etc/bash_completion
Install Latest Git
sudo add-apt-repository ppa:git-core/ppa -y
sudo apt update
sudo apt install git -y

Verify:

git --version
Install Java
OpenJDK 21
sudo apt install -y openjdk-21-jdk

Verify:

java --version
Jenkins Installation

Official Documentation:

Jenkins Linux Installation Guide

Add Jenkins Repository
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
https://pkg.jenkins.io/debian-stable binary/ | \
sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
Install Jenkins
sudo apt update
sudo apt install -y jenkins
Enable & Start Jenkins
sudo systemctl enable --now jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
Get Initial Admin Password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

Access Jenkins:

http://<jenkins-server-ip>:8080
Docker Installation

Official Documentation:

Docker Engine Installation Guide

Install Docker
sudo apt-get update

sudo apt-get install -y ca-certificates curl

sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
-o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc
Add Docker Repository
echo \
"deb [arch=$(dpkg --print-architecture) \
signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
Install Docker Packages
sudo apt-get install -y \
docker-ce \
docker-ce-cli \
containerd.io \
docker-buildx-plugin \
docker-compose-plugin
Add User to Docker Group
sudo usermod -aG docker $USER
newgrp docker

Verify:

docker ps
Allow Jenkins to Access Docker
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
Trivy Installation

Official Documentation:

Trivy Installation Guide

sudo apt update

sudo apt install -y wget gnupg lsb-release curl
Add Trivy Repository
curl -fsSL https://aquasecurity.github.io/trivy-repo/deb/public.key | \
gpg --dearmor | \
sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] \
https://aquasecurity.github.io/trivy-repo/deb \
$(lsb_release -sc) main" | \
sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt update
sudo apt install -y trivy

Verify:

trivy --version
SonarQube Setup Using Docker

Official Documentation:

SonarQube Documentation

Run SonarQube Container
docker run -d --name sonarqube \
-p 9000:9000 \
-v sonarqube_data:/opt/sonarqube/data \
-v sonarqube_logs:/opt/sonarqube/logs \
-v sonarqube_extensions:/opt/sonarqube/extensions \
sonarqube:<version>-community

Access SonarQube:

http://<server-ip>:9000
Prometheus Installation

Official Documentation:

Prometheus Downloads

Create Prometheus User
sudo useradd --system --no-create-home \
--shell /usr/sbin/nologin prometheus
Download Prometheus
wget -O prometheus.tar.gz \
"https://github.com/prometheus/prometheus/releases/download/<version>/prometheus-<version>.linux-amd64.tar.gz"

tar -xvf prometheus.tar.gz
cd prometheus-*/
Configure Prometheus
sudo mkdir -p /data /etc/prometheus
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
sudo chown -R prometheus:prometheus /etc/prometheus /data
Create Systemd Service

File:

/etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
RestartSec=5s

ExecStart=/usr/local/bin/prometheus \
--config.file=/etc/prometheus/prometheus.yml \
--storage.tsdb.path=/data \
--web.console.templates=/etc/prometheus/consoles \
--web.console.libraries=/etc/prometheus/console_libraries \
--web.listen-address=0.0.0.0:9090

[Install]
WantedBy=multi-user.target
Enable & Start Prometheus
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus

Access:

http://<server-ip>:9090
Node Exporter Setup

Official Documentation:

Node Exporter Guide

Install Node Exporter on both Jenkins and Nexus servers.

Create User
sudo useradd --system --no-create-home \
--shell /usr/sbin/nologin node_exporter
Download & Install
wget -O node_exporter.tar.gz \
"https://github.com/prometheus/node_exporter/releases/download/<version>/node_exporter-<version>.linux-amd64.tar.gz"

tar -xvf node_exporter.tar.gz

sudo mv node_exporter-*/node_exporter /usr/local/bin/

rm -rf node_exporter*
Create Systemd Service

File:

/etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure

ExecStart=/usr/local/bin/node_exporter --collector.logind

[Install]
WantedBy=multi-user.target
Enable & Start
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
Prometheus Scrape Configuration

Edit:

sudo vim /etc/prometheus/prometheus.yml

Add:

- job_name: "node_exporter"
  static_configs:
    - targets:
        - "<jenkins-server-private-ip>:9100"
        - "<nexus-server-private-ip>:9100"

- job_name: "jenkins"
  metrics_path: /prometheus
  static_configs:
    - targets:
        - "<jenkins-server-private-ip>:8080"

Validate configuration:

promtool check config /etc/prometheus/prometheus.yml

Restart Prometheus:

sudo systemctl restart prometheus

Check targets:

http://<prometheus-server-ip>:9090/targets
Fix Jenkins Prometheus 403 Error
Problem
403 Forbidden
Solution

Install:

Prometheus Metrics Plugin
Steps
Jenkins → Manage Jenkins → Plugins

Search and install:

Prometheus Metrics Plugin
Enable Metrics

Go to:

Manage Jenkins → System → Prometheus

Enable Prometheus metrics and save.

Verify endpoint:

http://<jenkins-ip>:8080/prometheus
Grafana Installation

Official Documentation:

Grafana Installation Guide

Install Grafana
sudo apt-get install -y \
apt-transport-https \
software-properties-common \
wget
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | \
gpg --dearmor | \
sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] \
https://apt.grafana.com stable main" | \
sudo tee -a /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get install -y grafana
Enable & Start Grafana
sudo systemctl daemon-reload
sudo systemctl enable --now grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server

Access:

http://<server-ip>:3000
Grafana Dashboard Configuration
Add Prometheus Datasource

Datasource URL:

http://<prometheus-server-ip>:9090
Import Dashboards
Dashboard	ID
Node Exporter Full	1860
Jenkins Dashboard	9964

Official Dashboard Library:

Grafana Dashboard Library

Nexus Repository Setup

Official Documentation:

Nexus Repository Documentation

Create Installation Script

Create file:

nexus.sh

Make executable:

chmod +x nexus.sh

Run:

./nexus.sh
Nexus Installation Script
#!/bin/bash

sudo apt-get update

sudo apt-get install -y \
wget \
apt-transport-https \
software-properties-common \
openjdk-17-jdk

sudo mkdir -p /opt/nexus/
sudo mkdir -p /tmp/nexus/

cd /tmp/nexus/

NEXUSURL="https://download.sonatype.com/nexus/3/nexus-<version>-linux-x86_64.tar.gz"

sudo wget $NEXUSURL -O nexus.tar.gz

sleep 10

EXTOUT=$(sudo tar xzvf nexus.tar.gz)

NEXUSDIR=$(echo $EXTOUT | cut -d '/' -f1)

sleep 5

sudo rm -rf /tmp/nexus/nexus.tar.gz

sudo cp -r /tmp/nexus/* /opt/nexus/

sleep 5

sudo useradd --system --no-create-home \
--shell /bin/false nexus

sudo chown -R nexus:nexus /opt/nexus

sudo cat <<EOT>> /etc/systemd/system/nexus.service
[Unit]
Description=nexus service
After=network.target

[Service]
Type=forking
LimitNOFILE=65536
ExecStart=/opt/nexus/$NEXUSDIR/bin/nexus start
ExecStop=/opt/nexus/$NEXUSDIR/bin/nexus stop
User=nexus
Restart=on-abort

[Install]
WantedBy=multi-user.target
EOT

sudo echo 'run_as_user="nexus"' > \
/opt/nexus/$NEXUSDIR/bin/nexus.rc

sudo systemctl daemon-reload

sudo systemctl start nexus
sudo systemctl enable nexus

sudo rm -rf /tmp/nexus/

echo "Nexus installation completed!"

Access:

http://<nexus-server-ip>:8081
Jenkins Plugins to Install
Plugin
Eclipse Temurin Installer
Email Extension Plugin
OWASP Dependency-Check Plugin
Pipeline Stage View
SonarQube Scanner for Jenkins
Prometheus Metrics Plugin
NodeJS Plugin
Nexus Artifact Uploader
Pipeline Maven Integration
Pipeline Utility Steps
Slack Notification
Amazon Web Services SDK
Amazon ECR
Pipeline AWS Steps
Docker Pipeline
CloudBees Docker Build and Publish
OWASP ZAP
Jenkins Credentials Configuration
Purpose	Credential ID	Type
Email	mail-cred	Username + App Password
SonarQube Token	sonar-token	Secret Text
Docker Registry	docker-cred	Username + Password
Nexus Repository	nexuslogin	Username + Password
AWS Credentials	awscreds	Access Key + Secret Key
Slack Token	slackcred	Secret Text
Jenkins Global Tool Configuration
Configure Tools
Tool	Name
JDK	JDK17
JDK	JDK21
Sonar Scanner	sonar-scanner
NodeJS	node16
Dependency Check	dp-check
Maven	MAVEN3
SonarQube Configuration in Jenkins
Configure SonarQube Server

Go to:

Manage Jenkins → System

Add SonarQube server:

Field	Value
Name	sonar-server
URL	http://<sonarqube-ip>:9000
Credentials	sonar-token
Email Configuration
SMTP Configuration
Field	Value
SMTP Server	smtp.gmail.com
SMTP Port	465 / 587
Authentication	Enabled
SSL/TLS	Enabled
Email Suffix	@gmail.com

Use Gmail App Password for authentication.

Slack Configuration
Steps
Create Slack workspace
Create channel
Install Jenkins CI app from Slack Marketplace
Copy generated token
Configure in Jenkins
SonarQube Webhook

Webhook URL:

http://<jenkins-ip>:8080/sonarqube-webhook/
AWS Setup
Create IAM User

User:

jenkins
Required Permissions
AmazonEC2ContainerRegistryFullAccess
Create ECR Repository

Example:

<application-image-repository>
Nexus Repository Setup

Go to:

Settings → Repositories

Create:

maven2 (hosted)

Example Repository:

<artifact-repository-name>
OWASP ZAP

Official Documentation:

OWASP ZAP Downloads

Cleanup

After project completion:

Terminate EC2 instances
Delete ECR repositories
Remove IAM users
Delete Slack channels/workspaces
Remove unused Docker volumes/images
Useful URLs
Tool	URL
Jenkins	http://<jenkins-ip>:8080
SonarQube	http://<server-ip>:9000
Prometheus	http://<server-ip>:9090
Grafana	http://<server-ip>:3000
Nexus Repository	http://<server-ip>:8081