# Project Setup Guide

## Credentials

Use your own credentials and replace these placeholders with secure values.

- Jenkins user: `JENKINS_USERNAME` / `JENKINS_PASSWORD`
- Sonar user: `SONAR_USERNAME` / `SONAR_PASSWORD`
- Grafana user: `GRAFANA_USERNAME` / `GRAFANA_PASSWORD`
- Nexus user: `NEXUS_USERNAME` / `NEXUS_PASSWORD`

> Warning: Do not include real passwords or sensitive data in this README.

## Infrastructure

- Jenkins instance type: `c5.xlarge` (4 CPU, 8 GB RAM)
- Storage: `30 GB EBS`
- Assumes Ubuntu/Debian-like environment and `sudo` privileges.

## Security Groups

### Jenkins Security Group

| Tool             | Default Port |
| ---------------- | ------------ |
| Jenkins          | `8080`       |
| SonarQube        | `9000`       |
| Prometheus       | `9090`       |
| Grafana          | `3000`       |
| Nexus Repository | `8081`       |
| Node Exporter    | `9100`       |

### Nexus Artifact Security Group

| Service       | Port |
| ------------- | ---- |
| SSH           | `22` |
| Nexus         | `8081` |
| node_exporter | `9100` |

## System Update & Common Packages

```bash
sudo apt update
sudo apt upgrade -y

sudo apt install -y bash-completion wget git zip unzip curl jq net-tools build-essential ca-certificates apt-transport-https gnupg fontconfig
```

Reload bash completion if needed:

```bash
source /etc/bash_completion
```

Install latest Git:

```bash
sudo add-apt-repository ppa:git-core/ppa
sudo apt update
sudo apt install -y git
```

## Java

Install OpenJDK 21:

```bash
sudo apt install -y openjdk-21-jdk
```

Verify Java:

```bash
java --version
```

## Jenkins

Official docs: https://www.jenkins.io/doc/book/installing/linux/

```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install -y jenkins
sudo systemctl enable --now jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
```

View the initial admin password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Then open:

```text
http://your-server-ip:8080
```

> Note: Jenkins requires a compatible Java runtime. Verify supported Java versions in Jenkins documentation.

## Docker

Official docs: https://docs.docker.com/engine/install/ubuntu/

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Add your user to the Docker group:

```bash
sudo usermod -aG docker $USER
newgrp docker
docker ps
```

If Jenkins needs Docker access:

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
sudo systemctl status docker
```

## Trivy (Vulnerability Scanner)

Docs: https://trivy.dev/v0.65/getting-started/installation/

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
trivy --version
```

## Prometheus

Official downloads: https://prometheus.io/download/

### Install Prometheus

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin prometheus
wget -O prometheus.tar.gz "https://github.com/prometheus/prometheus/releases/download/v3.5.3/prometheus-3.5.3.linux-amd64.tar.gz"
tar -xvf prometheus.tar.gz
cd prometheus-*/

sudo mkdir -p /data /etc/prometheus
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
sudo chown -R prometheus:prometheus /etc/prometheus /data
```

### Prometheus systemd service

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

Reload and start Prometheus:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```

Access Prometheus at:

```text
http://ip-address:9090
```

## Node Exporter

Docs: https://prometheus.io/docs/guides/node-exporter/

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin node_exporter
wget -O node_exporter.tar.gz "https://github.com/prometheus/node_exporter/releases/download/v1.11.1/node_exporter-1.11.1.linux-amd64.tar.gz"
tar -xvf node_exporter.tar.gz
sudo mv node_exporter-*/node_exporter /usr/local/bin/
rm -rf node_exporter*
```

### Node Exporter systemd service

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

Enable and start Node Exporter:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```

## Prometheus Scrape Configuration

Add the following jobs to `/etc/prometheus/prometheus.yml`:

```yaml
- job_name: "node_exporter"
  static_configs:
    - targets:
      - "NODE_EXPORTER_HOST_1:9100"
      - "NODE_EXPORTER_HOST_2:9100"

- job_name: "jenkins"
  metrics_path: /prometheus
  static_configs:
    - targets: ["JENKINS_HOST:8080"]
```

> Replace `NODE_EXPORTER_HOST_1`, `NODE_EXPORTER_HOST_2`, and `JENKINS_HOST` with your actual hostnames or IP addresses.

Validate Prometheus config:

```bash
promtool check config /etc/prometheus/prometheus.yml
sudo systemctl restart prometheus
```

## Jenkins Prometheus Plugin

If Jenkins target is `DOWN (403 Forbidden)`, install and enable the Prometheus Metrics Plugin.

1. Open Jenkins: `http://<JENKINS-IP>:8080`
2. Go to: Manage Jenkins ? Plugins ? Available
3. Search `Prometheus Metrics`
4. Install the plugin and restart Jenkins
5. Go to Manage Jenkins ? System ? Prometheus
6. Enable the metrics endpoint and save
7. Verify: `http://<JENKINS-IP>:8080/prometheus`
8. Reload Prometheus targets at `http://<PROMETHEUS-IP>:9090/targets`

## Grafana

Docs: https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/

```bash
sudo apt-get install -y apt-transport-https software-properties-common wget
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get install -y grafana
sudo systemctl daemon-reload
sudo systemctl enable --now grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server
```

Access Grafana at:

```text
http://ip-address:3000
```

Datasource: `http://prometheus-ip:9090`

## Nexus Setup

Nexus should run on a separate machine to store artifacts.

Docs:
- https://help.sonatype.com/en/sonatype-nexus-repository.html
- https://help.sonatype.com/en/download.html

Instance type: `t3.medium` (2 CPU, 4 GB RAM)

Create `nexus.sh`, make it executable and run it:

```bash
chmod +x nexus.sh
./nexus.sh
```

### Example `nexus.sh`

```bash
#!/bin/bash

# Update package list
sudo apt-get update
sudo apt-get install -y wget apt-transport-https software-properties-common
sudo apt-get update
sudo apt-get install -y openjdk-17-jdk

# Create directories
sudo mkdir -p /opt/nexus/
sudo mkdir -p /tmp/nexus/
cd /tmp/nexus/

# Download and extract Nexus
NEXUSURL="https://download.sonatype.com/nexus/3/nexus-3.85.0-03-linux-x86_64.tar.gz"
sudo wget "$NEXUSURL" -O nexus.tar.gz
sleep 10
EXTOUT=$(sudo tar xzvf nexus.tar.gz)
NEXUSDIR=$(echo "$EXTOUT" | cut -d '/' -f1)
sleep 5
sudo rm -rf /tmp/nexus/nexus.tar.gz
sudo cp -r /tmp/nexus/* /opt/nexus/
sleep 5

# Create nexus user and set permissions
sudo useradd --system --no-create-home --shell /bin/false nexus
sudo chown -R nexus:nexus /opt/nexus

# Create systemd service file
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

# Configure Nexus to run as nexus user
sudo sh -c 'echo "run_as_user=\"nexus\"" > /opt/nexus/$NEXUSDIR/bin/nexus.rc'

# Reload systemd and start Nexus
sudo systemctl daemon-reload
sudo systemctl enable --now nexus
sudo systemctl start nexus
sudo systemctl status nexus
```
