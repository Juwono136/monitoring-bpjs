# IP Network Monitoring of BPJS for Reporting + Alert Notification

This project is designed to monitor IP networks from [BPJS API](https://new-api.bpjs-kesehatan.go.id/) to detect anomalies, connectivity issues, and other network-related events in real time. It continuously monitors 4 BPJS IP addresses and 1 dummy IP (for error simulation and testing) to ensure network reliability, availability, and performance. The monitoring system is capable of visualizing metrics, sending alerts, and automating incident response through various notification channels.

The monitoring process is based on 4 key metrics:
- **Ping Response Time**: Measures latency between the BPJS IP host server and the monitoring system.
- **Ping Throughput**: Evaluates the amount of data successfully transmitted over the BPJS IP network.
- **Ping Status (UP/DOWN)**: Indicates network availability.
  - 1 = UP (reachable)
  - 0 = DOWN (unreachable / firing alert) 
- **Uptime Percentage**: Calculates the percentage of time the BPJS IP network is operational and accessible.

These metrics will be used as parameters, which will be sent to the n8n workflow (as the automation system) to send alert notifications if an error (DOWN) occurs on the IP network, delivered as a chat message via Telegram. In addition, it will automatically generate a report for each IP network in the form of a log history and record it into Google Sheets.

## 🧑‍💻 Tech Stack:
- ➡️ Docker Compose: Container orchestration
- ➡️ Prometheus: Time-series monitoring & alerting
- ➡️ Blackbox-exporter: Probes BPJS IPs using ICMP (Internet Control Message Protocol) ping
- ➡️ Alertmanager: Handles alert rules & notifications
- ➡️ Grafana: Dashboard visualization & uptime reporting
- ➡️ n8n workflow: Automate sending alerts to Telegram messages and save the log history into Google Sheet for reporting

## 🖥️ Requirements:
- ➡️ [Ubuntu 24.04](https://ubuntu.com/download/desktop)
- ➡️ [Docker on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- ➡️ [WSL2 for Windows 11](https://learn.microsoft.com/en-us/windows/wsl/install) (optional, if you are using windows as an operating system in your computer)

## 🏗️ System Architecture
![bpjs-monitoring](https://github.com/user-attachments/assets/c235c770-ef7f-4a07-8d6c-77d840e7bb4d)

- Blackbox Exporter → Performs ICMP ping probes.
- Prometheus → Scrapes probe results & applies alert rules.
- Alertmanager → Dispatches alerts to notification channels.
- Grafana → Displays real-time dashboards & historical reports.
- n8n → Automates alert delivery to Telegram messages and save the log history into Google Sheet for reporting.

## ⚙️ Installation & Setup
### 1. Clone Repository
```bash
git clone https://github.com/Juwono136/monitoring-bpjs
cd monitoring-bpjs
```

### 2. Configure Prometheus Targets
=> create a folder named prometheus inside the monitoring-bpjs folder
```bash
cd monitoring-bpjs
sudo mkdir -p prometheus
```

=> inside the prometheus folder, create a file named `prometheus.yml`
```bash
sudo nano prometheus.yml
```
- `prometheus.yml` file:
```bash
global:
  scrape_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - "alerts.yml"

scrape_configs:
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [icmp]
    static_configs:
      - targets:
          - 36.67.140.135
          - 118.97.79.198
          - 160.25.178.35
          - 160.25.179.35
          - 192.0.2.1 # dummy ip (for testing only. If you want to deploy the project, please remove this ip from the line)
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```
- save the file by pressing CTRL + X, then press Y and Enter.

==> 👉 Here, the short and simple explanation of the `prometheus.yml` file:
- global: sets the default scrape interval (15 seconds). Prometheus will collect metrics every 15s.
- alerting: tells Prometheus where to send alerts, in this case to an Alertmanager running at `alertmanager:9093`.
- rule_files: loads alerting rules from `alerts.yml` (we will create this file later).
- scrape_configs: defines what Prometheus should monitor.
  - job_name: 'blackbox' → A monitoring job using the Blackbox Exporter.
  - metrics_path: /probe & params: [icmp] → It probes targets using [ICMP](https://en.wikipedia.org/wiki/Internet_Control_Message_Protocol) (ping).
  - static_configs: lists IPs to monitor (some real IPs, plus a dummy IP for testing).
  - relabel_configs: rewrites labels so Prometheus sends probe requests correctly to the Blackbox Exporter at `blackbox-exporter:9115`.

### 3. Define Alerting Rules
=> inside the prometheus folder, create a file named `alrts.yml`
- `alerts.yml` file:
```bash
groups:
  - name: bpjs-ip-monitoring
    rules:
      - alert: BPJS_IP_Down
        expr: probe_success == 0
        for: 30s
        labels:
          severity: critical
          service: bpjs-ping
        annotations:
          summary: "BPJS IP {{ $labels.instance }} is DOWN"
          description: "Ping to {{ $labels.instance }} failed (probe_success=0)."
          ip: "{{ $labels.instance }}"
          status: "DOWN"
          triggered_at: "{{ $value }} at {{ $externalLabels.time }}"
          notes: "Use Prometheus API to fetch throughput and uptime."

      - alert: BPJS_IP_Up
        expr: probe_success == 1
        for: 30s
        labels:
          severity: info
          service: bpjs-ping
        annotations:
          summary: "BPJS IP {{ $labels.instance }} is UP"
          description: "Ping to {{ $labels.instance }} successful (probe_success=1)."
          ip: "{{ $labels.instance }}"
          status: "UP"
          triggered_at: "{{ $value }} at {{ $externalLabels.time }}"
          notes: "Use Prometheus API to fetch throughput and uptime."
```
👉 This code creates two alerts, one when an IP goes down and another when it comes back up, with labels and messages for easy tracking. Both of these alerts will be sent and read in the n8n workflow in JSON format.

### 4. Configure alertManager
- exit the prometheus folder, then create a new folder named `alertmanager`.
```bash
cd ..
sudo mkdir -p alertmanager
```
- inside the `alertmanager` folder, create a file named `alertmanager.yml`.
```bash
sudo nano alertmanager.yml
```
- `alertmanager.yml` file:
```bash
global:
  resolve_timeout: 1m

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 30s
  repeat_interval: 5m
  receiver: 'n8n-webhook'

receivers:
  - name: 'n8n-webhook'
    webhook_configs:
      - url: 'https://n8n.csbihub.id/webhook-test/dbaa60d4-0d0f-4d58-90a0-7ba4141964d6'
        send_resolved: true
```
👉 in short:
- global → resolve_timeout: 1m → If an alert is resolved, Alertmanager waits 1 minute before marking it as cleared.
- route: defines how alerts are grouped and sent.
  - group_by ['alertname'] → alerts with the same name are grouped together.
  - group_wait: 10s → waits 10s before sending the first alert (to group similar ones).
  - group_interval: 30s → sends new alerts in the same group every 30s
  - repeat_interval: 5m → repeats the alert every 5 minutes if still active.
  - receiver: 'n8n-webhook' → sends alerts to the receiver named `n8n-webhook`.
- receivers → n8n-webhook:
  - sends alerts via webhook to your n8n workflow URL.
  - send_resolved: true → also notifies when the issue is resolved (not just when it’s down).

### 5. Configure blackbox exporter config
- exit the alertmanager folder, then create a new folder named `blackbox`.
```bash
cd ..
sudo mkdir -p blackbox
```
- inside the `blackbox` folder, create a file named `config.yml`.
```bash
sudo nano config.yml
```
- `config.yml` file:
```bash
modules:
  icmp:
    prober: icmp
    timeout: 5s
    icmp:
      preferred_ip_protocol: "ip4"
```
👉 in short:
- modules → icmp: defines a probe module named icmp.
- prober: icmp → uses ICMP (ping) to check targets.
- timeout: 5s → each ping probe will time out if no response within 5 seconds.
- preferred_ip_protocol: "ip4" → forces the probe to use IPv4 instead of IPv6.

### 6. Create a docker-compose file as a multi-container app
- exit the blackbox folder, then create a new folder named `docker-compose.yml` in the main project folder (monitoring-bpjs).
```bash
cd ..
sudo nano docker-compose.yml
```
- `docker-compose.yml` file:
```bash
version: "3.8"

services:
  prometheus:
    image: prom/prometheus
    container_name: prometheus
    volumes:
      - ./prometheus:/etc/prometheus
    ports:
      - "9090:9090"
    networks:
      - monitor-net
    restart: unless-stopped

  blackbox-exporter:
    image: prom/blackbox-exporter
    container_name: blackbox-exporter
    volumes:
      - ./blackbox:/etc/blackbox_exporter
    ports:
      - "9115:9115"
    networks:
      - monitor-net
    restart: unless-stopped

  alertmanager:
    image: prom/alertmanager
    container_name: alertmanager
    volumes:
      - ./alertmanager:/etc/alertmanager
    ports:
      - "9093:9093"
    networks:
      - monitor-net
    restart: unless-stopped

  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - "3005:3000"
    networks:
      - monitor-net
    restart: unless-stopped

networks:
  monitor-net:
```
- save the file by pressing CTRL + X, then press Y and Enter.

👉 *we are not create the n8n workflow because it is already created and running on a different server: https://n8n.csbihub.id/. If you want to install it as well, you can add the n8n image to the docker-compose file (https://hub.docker.com/r/n8nio/n8n).*

### 7. Start Services
```bash
sudo docker compose up -d
```
- if the service is successfully created, there will be active containers for Prometheus, Blackbox, Alertmanager, and Grafana. Run the following command:
```bash
sudo docker ps
```
<img width="1892" height="202" alt="image" src="https://github.com/user-attachments/assets/c5645410-3fa8-4317-a429-af08f286a87b" />




