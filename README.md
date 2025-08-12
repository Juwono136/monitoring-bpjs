# IP Network Monitoring of BPJS for Reporting + Alert Notification

This application is designed to monitor [BPJS](https://new-api.bpjs-kesehatan.go.id/) IP networks to detect anomalies, connectivity issues, and other network-related events. The project monitors 4 BPJS IP addresses and 1 dummy IP for error testing.

The monitoring process is based on 4 key metrics:
- **Ping Response Time**: Measures latency between the BPJS IP host server and the monitoring system.
- **Ping Throughput**: Evaluates the amount of data successfully transmitted over the BPJS IP network.
- **Ping Status (UP/DOWN)**: Indicates network availability; 1 denotes a successful ping, 0 indicates a failed ping (firing).
- **Uptime Percentage**: Calculates the percentage of time the BPJS IP network is operational and accessible.


### 🧑‍💻 Technologies:
- ➡️ Docker Compose
- ➡️ Prometheus
- ➡️ Blackbox-exporter
- ➡️ Alertmanager
- ➡️ Grafana
- ➡️ n8n workflow

### 🖥️ Requirements:
- ➡️ Ubuntu 24.04
- ➡️ [Docker on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- ➡️ [WSL2 for Windows 11](https://learn.microsoft.com/en-us/windows/wsl/install) (optional, if you are using windows as an operating system in your computer)


