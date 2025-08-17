# IP Network Monitoring of BPJS for Reporting + Alert Notification

This project is designed to monitor IP networks from [BPJS API](https://new-api.bpjs-kesehatan.go.id/) to detect anomalies, connectivity issues, and other network-related events in real time. It continuously monitors 4 BPJS IP addresses and 1 dummy IP (for error simulation and testing) to ensure network reliability, availability, and performance. The monitoring system is capable of visualizing metrics, sending alerts, and automating incident response through various notification channels.

The monitoring process is based on 4 key metrics:
- **Ping Response Time**: Measures latency between the BPJS IP host server and the monitoring system.
- **Ping Throughput**: Evaluates the amount of data successfully transmitted over the BPJS IP network.
- **Ping Status (UP/DOWN)**: Indicates network availability.
  - 1 = UP (reachable)
  - 0 = DOWN (unreachable / firing alert) 
- **Uptime Percentage**: Calculates the percentage of time the BPJS IP network is operational and accessible.


### 🧑‍💻 Tech Stack:
- ➡️ Docker Compose: Container orchestration
- ➡️ Prometheus: Time-series monitoring & alerting
- ➡️ Blackbox-exporter: Probes BPJS IPs using ICMP (Internet Control Message Protocol) ping
- ➡️ Alertmanager: Handles alert rules & notifications
- ➡️ Grafana: Dashboard visualization & uptime reporting
- ➡️ n8n workflow: Automates notifications (Telegram Message)

### 🖥️ Requirements:
- ➡️ Ubuntu 24.04
- ➡️ [Docker on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- ➡️ [WSL2 for Windows 11](https://learn.microsoft.com/en-us/windows/wsl/install) (optional, if you are using windows as an operating system in your computer)

### 🏗️ System Architecture




