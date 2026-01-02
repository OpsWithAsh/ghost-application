# Ghost Application – Production Grade CI/CD Project

## Overview
This project demonstrates a complete production-grade DevOps setup for the Ghost blogging platform using:

- Docker & Docker Compose
- AWS EC2 & ECR
- GitHub Actions CI/CD
- Prometheus, Blackbox Exporter, Alertmanager, Grafana
- Staging & Production environments

---

## Architecture

### Servers
| Purpose | IP |
|------|----|
| DevOps / Monitoring | 54.166.223.112 |
| Staging | 18.212.187.102 |
| Production | 13.222.29.97 |

---

## CI/CD Flow
1. Code push to `main`
2. SonarQube analysis
3. Docker image build
4. Push image to Amazon ECR
5. Deploy to Staging (Docker Compose)
6. Smoke test
7. Deploy to Production

---

## Repository Structure
# Ghost Application – Production Grade CI/CD Project

## Overview
This project demonstrates a complete production-grade DevOps setup for the Ghost blogging platform using:

- Docker & Docker Compose
- AWS EC2 & ECR
- GitHub Actions CI/CD
- Prometheus, Blackbox Exporter, Alertmanager, Grafana
- Staging & Production environments

---

## Architecture

### Servers
| Purpose | IP |
|------|----|
| DevOps / Monitoring | 54.166.223.112 |
| Staging | 18.212.187.102 |
| Production | 13.222.29.97 |

---

## CI/CD Flow
1. Code push to `main`
2. SonarQube analysis
3. Docker image build
4. Push image to Amazon ECR
5. Deploy to Staging (Docker Compose)
6. Smoke test
7. Deploy to Production

---

## Repository Structure
ghost-application/
├── .github/workflows/ci-cd.yaml
├── deploy/
│ ├── docker-compose.staging.yml
│ └── docker-compose.production.yml
├── config/
│ ├── config.staging.json
│ └── config.production.json
├── monitoring/
│ ├── prometheus.yml
│ ├── blackbox.yml
│ ├── alertmanager.yml
│ └── alert-rules.yml
└── ghost/
└── Dockerfile


---

## Installation – Servers

### Common
```bash
sudo apt update
sudo apt install -y docker.io docker-compose-plugin awscli
sudo usermod -aG docker ubuntu

| Component         | Port |
| ----------------- | ---- |
| Prometheus        | 9090 |
| Blackbox Exporter | 9115 |
| Alertmanager      | 9093 |
| Grafana           | 3000 |

Staging: http://publicip:port

Production: http://publicip:port

Rollback Strategy

Docker tag :previous
Re-deploy previous image
Zero downtime using Docker Compose

1️⃣ Install Prometheus
Download & Extract

cd /opt
sudo curl -LO https://github.com/prometheus/prometheus/releases/download/v2.54.1/prometheus-2.54.1.linux-amd64.tar.gz
sudo tar -xvf prometheus-2.54.1.linux-amd64.tar.gz
sudo mv prometheus-2.54.1.linux-amd64 /usr/local/prometheus

Prometheus Configuration

Create config file:

sudo nano /usr/local/prometheus/prometheus.yml
global:
  scrape_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - localhost:9093

rule_files:
  - "/usr/local/prometheus/alert-rules.yml"

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "ghost-uptime"
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - http://18.212.187.102   # staging
          - http://13.222.29.97     # production
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9115


Prometheus Systemd Service
sudo nano /etc/systemd/system/prometheus.service

[Unit]
Description=Prometheus
After=network.target

[Service]
User=root
ExecStart=/usr/local/prometheus/prometheus \
  --config.file=/usr/local/prometheus/prometheus.yml \
  --storage.tsdb.path=/usr/local/prometheus/data
Restart=always

[Install]
WantedBy=multi-user.target


Start Prometheus:
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus

Access:
http://publicip:9090

2️⃣ Install Blackbox Exporter
cd /opt
sudo curl -LO https://github.com/prometheus/blackbox_exporter/releases/download/v0.25.0/blackbox_exporter-0.25.0.linux-amd64.tar.gz
sudo tar -xvf blackbox_exporter-0.25.0.linux-amd64.tar.gz
sudo mv blackbox_exporter-0.25.0.linux-amd64 /usr/local/blackbox_exporter

Blackbox Config
sudo nano /usr/local/blackbox_exporter/blackbox.yml

modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_status_codes: [200]

Systemd Service
sudo nano /etc/systemd/system/blackbox_exporter.service

[Unit]
Description=Blackbox Exporter
After=network.target

[Service]
User=root
ExecStart=/usr/local/blackbox_exporter/blackbox_exporter \
  --config.file=/usr/local/blackbox_exporter/blackbox.yml
Restart=always

[Install]
WantedBy=multi-user.target

sudo systemctl daemon-reload
sudo systemctl enable blackbox_exporter
sudo systemctl start blackbox_exporter

ACCESS:
http://publicip:9115

3️⃣ Install Alertmanager

cd /opt
sudo curl -LO https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz
sudo tar -xvf alertmanager-0.27.0.linux-amd64.tar.gz
sudo mv alertmanager-0.27.0.linux-amd64 /usr/local/alertmanager

Alertmanager Config
sudo nano /usr/local/alertmanager/alertmanager.yml

route:
  receiver: "default"

receivers:
  - name: "default"


Systemd Service
sudo nano /etc/systemd/system/alertmanager.service

[Unit]
Description=Alertmanager
After=network.target

[Service]
User=root
ExecStart=/usr/local/alertmanager/alertmanager \
  --config.file=/usr/local/alertmanager/alertmanager.yml \
  --storage.path=/usr/local/alertmanager/data
Restart=always

[Install]
WantedBy=multi-user.target


Start:

sudo systemctl daemon-reload
sudo systemctl enable alertmanager
sudo systemctl start alertmanager


Access:

http://54.166.223.112:9093

4️⃣ Install Grafana
sudo apt update
sudo apt install -y apt-transport-https software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
sudo apt update
sudo apt install grafana


Start Grafana:

sudo systemctl enable grafana-server
sudo systemctl start grafana-server


Access:

http://54.166.223.112:3000


Default login:

admin / admin

5️⃣ Grafana Configuration
Add Prometheus Data Source

URL: http://localhost:9090

Type: Prometheus

Import Dashboard

Dashboard ID: 7587 (Blackbox Exporter)

Dashboard ID: 1860 (Node Exporter – optional)

✅ Verification Commands
systemctl status prometheus
systemctl status blackbox_exporter
systemctl status alertmanager
systemctl status grafana-server
