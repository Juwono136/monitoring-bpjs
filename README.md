# IP Network Monitoring of BPJS for Reporting + Alert Notification
![monitoring-bpjs](https://github.com/user-attachments/assets/f09f6fef-da47-4527-9937-411aeb0c5557)

This project is designed to monitor IP networks from [BPJS API](https://new-api.bpjs-kesehatan.go.id/) to detect anomalies, connectivity issues, and other network-related events in real time. It continuously monitors 4 BPJS IP addresses and 1 dummy IP (for error simulation and testing) to ensure network reliability, availability, and performance. The monitoring system is capable of visualizing metrics, sending alerts, and automating incident response through various notification channels.

The monitoring process is based on 4 key metrics:
- **Ping Response Time**: Measures latency between the BPJS IP host server and the monitoring system.
- **Average of Latency**: Average delay of ICMP (ping) packets sent and received.
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

## 📂 Project Structure
```bash
monitoring-bpjs/
├── alertmanager
│   └── alertmanager.yml
├── blackbox
│   └── config.yml
├── docker-compose.yml
└── prometheus
    ├── alerts.yml
    └── prometheus.yml
```

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
=> inside the prometheus folder, create a file named `alerts.yml`
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
- if the service is successfully created, there will be active containers for Prometheus, Blackbox Exporter, Alertmanager, and Grafana. Run the following command:
```bash
sudo docker ps
```
<img width="1892" height="202" alt="image" src="https://github.com/user-attachments/assets/c5645410-3fa8-4317-a429-af08f286a87b" />

👉 Now, check in your browser (for example: Google Chrome), then type in the following localhost links to verify whether the services are running successfully or not:
- Prometheus: http://localhost:9090
<img width="1907" height="950" alt="image" src="https://github.com/user-attachments/assets/31de6db1-df8d-4704-aed8-fe6b3e0ba6c2" />

- Alertmanager: http://localhost:9093
<img width="1916" height="967" alt="image" src="https://github.com/user-attachments/assets/ed4e2b96-2448-4ed4-bbd0-956b805793b7" />

- Blackbox exporter: http://localhost:9115
<img width="1912" height="967" alt="image" src="https://github.com/user-attachments/assets/d077fa4d-b249-459a-b958-d3b398699b97" />

- Grafana: http://localhost:3005
<img width="1917" height="863" alt="image" src="https://github.com/user-attachments/assets/7ae08e8a-5943-4375-a123-81d57388b91b" />

### 8. Create a grafana dashboard for monitoring
- In the browser, type `http://localhost:3005`, log in with the default username and password (username: admin, password: admin), then change the default password to a new password of your choice.
- In the Grafana dashboard sidebar menu, select Add new connection → search for Prometheus → Add new data source.
- Set up Prometheus:
  - Name: prometheus
  - Under Connection → Prometheus server URL: `http://prometheus:9090`
  - Then click the **Save & Test** button.
- Next, in the Grafana sidebar menu, select Dashboard → Create new dashboard → Add visualization.
- In the Select data source section, choose **Prometheus**.
- We will create 4 graph visualizations for the following parameters/metrics:
  - Response Ping Time
  - Throughput / Rate Ping
  - Ping UP/DOWN
  - Uptime Percentage (Last 5 Minutes)

#### => Response Ping Time
- In the Queries tab, under the metrics browser, type: `probe_duration_seconds`, then click "Run queries", and the graph will appear in the visualization component.
- On the right sidebar, there are several settings to edit the graph. Change them with the following information:
  - Visualization: Time Series
  - Panel options → Title: Response Ping Time
  - Standard options → Unit: seconds (s)
- Then, click the "Save dashboard" button → give the dashboard a name (e.g., BPJS Monitoring) → click Save.
- Click the "Back to dashboard" button to create or add another visualization.
<img width="1917" height="867" alt="image" src="https://github.com/user-attachments/assets/53845dfa-2b51-4dd6-ae26-49b1b031f4e3" />

#### => Average of Latency
- In the Queries tab, under the metrics browser, type: `avg_over_time(probe_duration_seconds[1m])`, then click "Run queries".
- On the right sidebar, there are several settings to edit the graph. Change them with the following information:
  - Visualization: Time Series
  - Panel options → Title: Average of Latency
- Then, click the "Save dashboard" button → click Save.
<img width="1906" height="865" alt="image" src="https://github.com/user-attachments/assets/d6ecb2a8-5020-420d-bb13-cbb05aefff64" />

#### => Ping UP/DOWN
- In the Queries tab, under the metrics browser, type: `probe_success`, then click "Run queries".
- On the right sidebar, there are several settings to edit the graph. Change them with the following information:
  - Visualization: Stat
  - Panel options → Title: Ping UP / DOWN
  - Stat styles → Change Layout orientation to "Horizontal"
- Then, click the "Save dashboard" button → click Save.
<img width="1915" height="861" alt="image" src="https://github.com/user-attachments/assets/8fb6bf3c-1f7a-4b84-9299-cde89f3b6e52" />

#### => Uptime Percentage (Last 5 Minutes)
- In the Queries tab, under the metrics browser, type: `avg_over_time(probe_success[5m]) * 100`, then click "Run queries".
- On the right sidebar, there are several settings to edit the graph. Change them with the following information:
  - Visualization: Gauge
  - Panel options → Title: Uptime Percentage (Last 5 Minutes)
  - Standard options → Unit: Percent (0-100)
- Then, click the "Save dashboard" button → click Save.
<img width="1918" height="868" alt="image" src="https://github.com/user-attachments/assets/6a19cc31-8dde-4a1f-86ec-45a6e8921419" />

- 👉 Arrange the position of each graph component to look proportional by using drag and drop. Then click "Save dashboard".
<img width="1898" height="866" alt="image" src="https://github.com/user-attachments/assets/db3cc6b2-4c7f-454a-bcb8-0e0fe3079e99" />

## 📡 Metrics Collected
| Metric                        | Description                             |
| ----------------------------- | --------------------------------------- |
| `probe_success`               | IP status → 1 = UP, 0 = DOWN            |
| `probe_duration_seconds`      | Ping response time (latency in seconds) |

## 🔄 n8n Workflow
<img width="1158" height="545" alt="image" src="https://github.com/user-attachments/assets/f86e91ab-1bce-4d51-9e9a-347182716364" />

- Webhook node functions to retrieve the n8n-webhook endpoint (https://n8n.csbihub.id/webhook-test/dbaa60d4-0d0f-4d58-90a0-7ba4141964d6) that was previously created in `alertmanager.yml`. Its output is the rules from alerts.yml in JSON format:
```json
[
  {
    "headers": {
      "host": "n8n.csbihub.id",
      "user-agent": "Alertmanager/0.28.1",
      "content-length": "2326",
      "accept-encoding": "gzip, br",
      "cdn-loop": "cloudflare; loops=1",
      "cf-connecting-ip": "180.254.65.85",
      "cf-ipcountry": "ID",
      "cf-ray": "970b16625ada0516-HKG",
      "cf-visitor": "{\"scheme\":\"https\"}",
      "cf-warp-tag-id": "118b5930-d741-4342-90e7-620c5d355661",
      "connection": "keep-alive",
      "content-type": "application/json",
      "x-forwarded-for": "180.254.65.85",
      "x-forwarded-proto": "https"
    },
    "params": {},
    "query": {},
    "body": {
      "receiver": "n8n-webhook",
      "status": "firing",
      "alerts": [
        {
          "status": "firing",
          "labels": {
            "alertname": "BPJS_IP_Up",
            "instance": "118.97.79.198",
            "job": "blackbox",
            "service": "bpjs-ping",
            "severity": "info"
          },
          "annotations": {
            "description": "Ping to 118.97.79.198 successful (probe_success=1).",
            "ip": "118.97.79.198",
            "status": "UP",
            "summary": "BPJS IP 118.97.79.198 is UP"
          },
          "startsAt": "2025-08-17T18:03:08.37Z",
          "endsAt": "0001-01-01T00:00:00Z",
          "generatorURL": "http://31a467779b75:9090/graph?g0.expr=probe_success+%3D%3D+1&g0.tab=1",
          "fingerprint": "98721d24261df883"
        },
        {
          "status": "firing",
          "labels": {
            "alertname": "BPJS_IP_Up",
            "instance": "160.25.178.35",
            "job": "blackbox",
            "service": "bpjs-ping",
            "severity": "info"
          },
          "annotations": {
            "description": "Ping to 160.25.178.35 successful (probe_success=1).",
            "ip": "160.25.178.35",
            "status": "UP",
            "summary": "BPJS IP 160.25.178.35 is UP"
          },
          "startsAt": "2025-08-17T18:03:08.37Z",
          "endsAt": "0001-01-01T00:00:00Z",
          "generatorURL": "http://31a467779b75:9090/graph?g0.expr=probe_success+%3D%3D+1&g0.tab=1",
          "fingerprint": "b75ff9fda0b3d445"
        },
        {
          "status": "firing",
          "labels": {
            "alertname": "BPJS_IP_Up",
            "instance": "160.25.179.35",
            "job": "blackbox",
            "service": "bpjs-ping",
            "severity": "info"
          },
          "annotations": {
            "description": "Ping to 160.25.179.35 successful (probe_success=1).",
            "ip": "160.25.179.35",
            "status": "UP",
            "summary": "BPJS IP 160.25.179.35 is UP"
          },
          "startsAt": "2025-08-17T18:03:08.37Z",
          "endsAt": "0001-01-01T00:00:00Z",
          "generatorURL": "http://31a467779b75:9090/graph?g0.expr=probe_success+%3D%3D+1&g0.tab=1",
          "fingerprint": "0fb0bfed9f95f358"
        },
        {
          "status": "firing",
          "labels": {
            "alertname": "BPJS_IP_Up",
            "instance": "36.67.140.135",
            "job": "blackbox",
            "service": "bpjs-ping",
            "severity": "info"
          },
          "annotations": {
            "description": "Ping to 36.67.140.135 successful (probe_success=1).",
            "ip": "36.67.140.135",
            "status": "UP",
            "summary": "BPJS IP 36.67.140.135 is UP"
          },
          "startsAt": "2025-08-17T18:03:08.37Z",
          "endsAt": "0001-01-01T00:00:00Z",
          "generatorURL": "http://31a467779b75:9090/graph?g0.expr=probe_success+%3D%3D+1&g0.tab=1",
          "fingerprint": "bf9cfc8682f241a3"
        }
      ],
      "groupLabels": {
        "alertname": "BPJS_IP_Up"
      },
      "commonLabels": {
        "alertname": "BPJS_IP_Up",
        "job": "blackbox",
        "service": "bpjs-ping",
        "severity": "info"
      },
      "commonAnnotations": {
        "status": "UP"
      },
      "externalURL": "http://87a93f754f43:9093",
      "version": "4",
      "groupKey": "{}:{alertname=\"BPJS_IP_Up\"}",
      "truncatedAlerts": 0
    },
    "webhookUrl": "https://n8n.csbihub.id/webhook/dbaa60d4-0d0f-4d58-90a0-7ba4141964d6",
    "executionMode": "production"
  }
]
```
- Then, a condition is created, when status = DOWN, an alert notification is sent to a Telegram message.
<img width="1370" height="858" alt="image" src="https://github.com/user-attachments/assets/5e00d2dc-4648-4292-b207-cb0b56437e79" />

- All the information obtained from the webhook node, before being inserted into Google Sheets, is cleaned up using JavaScript code as follows:
```javascript
const alerts = items[0].json.body.alerts;

const status = alerts[0].annotations.status;

let results = [];

if (status === "UP") {
  results = alerts
    .filter(alert => alert.annotations.status === "UP")
    .map(alert => ({
      ip: alert.annotations.ip,
      status: alert.annotations.status,
      summary: alert.annotations.summary,
      timeUp: alert.startsAt,
    }));
} else if (status === "DOWN") {
  const alert = alerts[0];
  results = [{
    ip: alert.annotations.ip,
    status: alert.annotations.status,
    summary: alert.annotations.summary,
    timeDown: alert.startsAt,
  }];
}

return results.map(r => ({ json: r }));

```
👉 This code will produce a new JSON data format that is easier to read:
```json
[
  {
    "ip": "118.97.79.198",
    "status": "UP",
    "summary": "BPJS IP 118.97.79.198 is UP",
    "timeUp": "2025-08-17T18:03:08.37Z"
  },
  {
    "ip": "160.25.178.35",
    "status": "UP",
    "summary": "BPJS IP 160.25.178.35 is UP",
    "timeUp": "2025-08-17T18:03:08.37Z"
  },
  {
    "ip": "160.25.179.35",
    "status": "UP",
    "summary": "BPJS IP 160.25.179.35 is UP",
    "timeUp": "2025-08-17T18:03:08.37Z"
  },
  {
    "ip": "36.67.140.135",
    "status": "UP",
    "summary": "BPJS IP 36.67.140.135 is UP",
    "timeUp": "2025-08-17T18:03:08.37Z"
  }
]
```
- Next, the result from the "fetch the data" code node will be forwarded to Google Sheets to be stored as a log report.
<img width="847" height="245" alt="image" src="https://github.com/user-attachments/assets/6c497aa5-637e-46ad-bf1e-03d314d3d9d9" />

## 📝 Documentation notes
- start docker compose
```bash
sudo docker compose start
```
- stop docker compose
```bash
sudo docker compose stop
```
- remove service of docker compose
```bash
sudo docker compose down
```
- view created docker images
```bash
sudo docker images
```

## 🚀 Future Improvements
- Integrate Slack / Microsoft Teams / Gmail notifications.
- Store long-term uptime data in PostgreSQL or any databases.
- Add other log information such as throughput, uptime, and so on to Google Sheets for a more complete report.
- Secure monitoring endpoints with Zero Trust Network Access (ZTNA) (e.g., Cloudflare Zero Trust, Twingate, etc).

## 🤝 Project Members
- Juwono (https://github.com/Juwono136)
