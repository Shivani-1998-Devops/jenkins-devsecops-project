# DevSecOps CI/CD Setup Guide

## Overview

This guide explains how to set up a complete DevSecOps CI/CD environment using:

- Jenkins
- SonarQube
- Prometheus
- Grafana
- Nexus Repository
- Docker
- Trivy
- OWASP ZAP

## Infrastructure Requirements

### Jenkins / Monitoring Server

| Resource | Specification |
| --- | --- |
| Instance Type | `c5.xlarge` |
| CPU | `4 vCPU` |
| RAM | `8 GB` |
| Storage | `30 GB EBS` |

### Nexus Server

| Resource | Specification |
| --- | --- |
| Instance Type | `t3.medium` |
| CPU | `2 vCPU` |
| RAM | `4 GB` |

## Security Group Ports

### Jenkins Server Security Group

| Service | Port |
| --- | --- |
| SSH | `22` |
| Jenkins | `8080` |
| SonarQube | `9000` |
| Prometheus | `9090` |
| Grafana | `3000` |
| Nexus Repository | `8081` |
| Node Exporter | `9100` |

### Nexus Server Security Group

| Service | Port |
| --- | --- |
| SSH | `22` |
| Nexus Repository | `8081` |
| Node Exporter | `9100` |

## System Update & Common Packages

```bash
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
```

Reload bash completion:

```bash
source /etc/bash_completion
```

## Install Latest Git

```bash
sudo add-apt-repository ppa:git-core/ppa -y
sudo apt update
sudo apt install -y git
```

Verify:

```bash
git --version
```

## Install Java

Install OpenJDK 21:

```bash
sudo apt install -y openjdk-21-jdk
```

Verify:

```bash
java --version
```

## Jenkins Installation

Official Documentation: https://www.jenkins.io/doc/book/installing/linux/

### Add Jenkins Repository

```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
```

### Install Jenkins

```bash
sudo apt update
sudo apt install -y jenkins
```

### Enable & Start Jenkins

```bash
sudo systemctl enable --now jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
```

### Get Initial Admin Password

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Access Jenkins:

```
http://<jenkins-server-ip>:8080
```

## Docker Installation

Official Documentation: https://docs.docker.com/engine/install/ubuntu/

### Install Docker

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

### Add Docker Repository

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

### Install Docker Packages

```bash
sudo apt-get install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

### Add User to Docker Group

```bash
sudo usermod -aG docker $USER
newgrp docker
```

Verify:

```bash
docker ps
```

### Allow Jenkins to Access Docker

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

## Trivy Installation

Official Documentation: https://trivy.dev/v0.65/getting-started/installation/

```bash
sudo apt update
sudo apt install -y wget gnupg lsb-release curl

curl -fsSL https://aquasecurity.github.io/trivy-repo/deb/public.key | \
  gpg --dearmor | \
  sudo tee /usr/share/keyrings/trivy.gpg > /dev/null

echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | \
  sudo tee /etc/apt/sources.list.d/trivy.list

sudo apt update
sudo apt install -y trivy
```

Verify:

```bash
trivy --version
```

## SonarQube Setup Using Docker

Official Documentation: https://docs.sonarqube.org/latest/setup/install-server/

### Run SonarQube Container

```bash
docker run -d --name sonarqube \
  -p 9000:9000 \
  -v sonarqube_data:/opt/sonarqube/data \
  -v sonarqube_logs:/opt/sonarqube/logs \
  -v sonarqube_extensions:/opt/sonarqube/extensions \
  sonarqube:<version>-community
```

Access SonarQube:

```
http://<server-ip>:9000
```

## Prometheus Installation

Official Documentation: https://prometheus.io/download/

### Create Prometheus User

```bash
sudo useradd --system --no-create-home \
  --shell /usr/sbin/nologin prometheus
```

### Download Prometheus

```bash
wget -O prometheus.tar.gz \
  "https://github.com/prometheus/prometheus/releases/download/<version>/prometheus-<version>.linux-amd64.tar.gz"

tar -xvf prometheus.tar.gz
cd prometheus-*/
```

### Configure Prometheus

```bash
sudo mkdir -p /data /etc/prometheus
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
sudo chown -R prometheus:prometheus /etc/prometheus /data
```

### Create Prometheus Systemd Service

Edit `/etc/systemd/system/prometheus.service`:

```ini
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
```

### Enable & Start Prometheus

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```

Access Prometheus:

```
http://<server-ip>:9090
```

## Node Exporter Setup

Official Documentation: https://prometheus.io/docs/guides/node-exporter/

> Install Node Exporter on both Jenkins and Nexus servers.

### Create Node Exporter User

```bash
sudo useradd --system --no-create-home \
  --shell /usr/sbin/nologin node_exporter
```

### Download & Install Node Exporter

```bash
wget -O node_exporter.tar.gz \
  "https://github.com/prometheus/node_exporter/releases/download/<version>/node_exporter-<version>.linux-amd64.tar.gz"

tar -xvf node_exporter.tar.gz
sudo mv node_exporter-*/node_exporter /usr/local/bin/
rm -rf node_exporter*
```

### Create Node Exporter Systemd Service

Edit `/etc/systemd/system/node_exporter.service`:

```ini
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
```

### Enable & Start Node Exporter

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```

## Prometheus Scrape Configuration

Edit `/etc/prometheus/prometheus.yml` and add:

```yaml
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
```

Validate configuration:

```bash
promtool check config /etc/prometheus/prometheus.yml
sudo systemctl restart prometheus
```

Check targets:

```
http://<prometheus-server-ip>:9090/targets
```

## Fix Jenkins Prometheus 403 Error

If Jenkins returns `403 Forbidden`, install and enable the Prometheus Metrics Plugin.

### Steps

1. Go to Jenkins → Manage Jenkins → Plugins
2. Search for `Prometheus Metrics Plugin`
3. Install the plugin and restart Jenkins
4. Go to Jenkins → Manage Jenkins → System → Prometheus
5. Enable Prometheus metrics and save
6. Verify endpoint:

```
http://<jenkins-ip>:8080/prometheus
```

## Grafana Installation

Official Documentation: https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/

### Install Grafana

```bash
sudo apt-get install -y \
  apt-transport-https \
  software-properties-common \
  wget

sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | \
  gpg --dearmor | \
  sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | \
  sudo tee -a /etc/apt/sources.list.d/grafana.list

sudo apt-get update
sudo apt-get install -y grafana
```

### Enable & Start Grafana

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server
```

Access Grafana:

```
http://<server-ip>:3000
```

## Grafana Dashboard Configuration

### Add Prometheus Datasource

```
http://<prometheus-server-ip>:9090
```

### Import Dashboards

| Dashboard | ID |
| --- | --- |
| Node Exporter Full | `1860` |
| Jenkins Dashboard | `9964` |

See: https://grafana.com/grafana/dashboards/

## Nexus Repository Setup

Official Documentation:
- https://help.sonatype.com/en/sonatype-nexus-repository.html
- https://help.sonatype.com/en/download.html

### Create Installation Script

Create file `nexus.sh`, make it executable, and run:

```bash
chmod +x nexus.sh
./nexus.sh
```

### Example Nexus Installation Script

```bash
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
sudo wget "$NEXUSURL" -O nexus.tar.gz
sleep 10

EXTOUT=$(sudo tar xzvf nexus.tar.gz)
NEXUSDIR=$(echo "$EXTOUT" | cut -d '/' -f1)
sleep 5

sudo rm -rf /tmp/nexus/nexus.tar.gz
sudo cp -r /tmp/nexus/* /opt/nexus/
sleep 5

sudo useradd --system --no-create-home \
  --shell /bin/false nexus
sudo chown -R nexus:nexus /opt/nexus

sudo tee /etc/systemd/system/nexus.service > /dev/null <<'EOT'
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

sudo sh -c 'echo "run_as_user=\"nexus\"" > /opt/nexus/$NEXUSDIR/bin/nexus.rc'

sudo systemctl daemon-reload
sudo systemctl start nexus
sudo systemctl enable nexus
sudo systemctl status nexus
```

Access Nexus:

```
http://<nexus-server-ip>:8081
```

## Jenkins Plugins to Install

- Eclipse Temurin Installer
- Email Extension Plugin
- OWASP Dependency-Check Plugin
- Pipeline Stage View
- SonarQube Scanner for Jenkins
- Prometheus Metrics Plugin
- NodeJS Plugin
- Nexus Artifact Uploader
- Pipeline Maven Integration
- Pipeline Utility Steps
- Slack Notification
- Amazon Web Services SDK
- Amazon ECR
- Pipeline AWS Steps
- Docker Pipeline
- CloudBees Docker Build and Publish
- OWASP ZAP

## Jenkins Credentials Configuration

| Purpose | Credential ID | Type |
| --- | --- | --- |
| Email | `mail-cred` | Username + App Password |
| SonarQube Token | `sonar-token` | Secret Text |
| Docker Registry | `docker-cred` | Username + Password |
| Nexus Repository | `nexuslogin` | Username + Password |
| AWS Credentials | `awscreds` | Access Key + Secret Key |
| Slack Token | `slackcred` | Secret Text |

## Jenkins Global Tool Configuration

Configure the following tools in Manage Jenkins → Tools:

| Tool | Name |
| --- | --- |
| JDK | `JDK17` |
| JDK | `JDK21` |
| Sonar Scanner | `sonar-scanner` |
| NodeJS | `node16` |
| Dependency Check | `dp-check` |
| Maven | `MAVEN3` |

## SonarQube Configuration in Jenkins

Configure SonarQube server under Manage Jenkins → System by adding your SonarQube server details and credentials.

## Next Steps

1. Configure Jenkins pipeline jobs to use these tools
2. Set up webhook integrations with your Git repository
3. Configure SonarQube Quality Gates for code quality enforcement
4. Set up email and Slack notifications
5. Create backup and disaster recovery procedures
