# 10 - Grafana & Prometheus

Monitoring stack deployed on container 106 (10.2.7.108) for visualizing homelab metrics.

## Services

| Service | URL | Auth |
|---------|-----|------|
| **Grafana** | http://10.2.7.108:3000 | admin / GraphMyWorld |
| **Prometheus** | http://10.2.7.108:9090 | none |
| **Alertmanager** | http://10.2.7.108:9093 | none |
| **cAdvisor** | http://10.2.7.108:8080 | none |

## Scrape Targets (Prometheus)

| Job | Target | What It Monitors |
|-----|--------|------------------|
| `node_pve` | 10.2.7.64:9100 | PVE host CPU, RAM, disk |
| `node_containers` | 10.2.7.108:9100 | CT 106 resources |
| `cadvisor` | 10.2.7.108:8080 | All Docker containers |
| `pihole` | 10.2.7.108:9617 | DNS queries, top domains |
| `prometheus` | localhost:9090 | Prometheus self |

## Dashboards

| Dashboard | Description |
|-----------|-------------|
| **Node Exporter Full** | Host-level CPU, RAM, disk, network (PVE + CT 106) |
| **Docker Container & Host Metrics** | Container-level metrics via cAdvisor |
| **Pi-hole Exporter** | DNS query volume, top blocked domains |

All dashboards auto-provisioned from `/opt/monitoring/dashboards/`.

## Alerts

Prometheus alerts evaluated every 30s, forwarded to Discord via Alertmanager:

| Alert | Threshold | Severity |
|-------|-----------|----------|
| InstanceDown | `up == 0` for 1m | 🔴 Critical |
| HighDiskUsage | Disk >85% for 5m | 🟡 Warning |
| DiskAlmostFull | Disk >95% for 2m | 🔴 Critical |
| HighMemoryUsage | RAM >90% for 5m | 🟡 Warning |
| HighCPUUsage | CPU >90% for 10m | 🟡 Warning |

Alert configs:
- **Rules:** `/opt/monitoring/homelab_alerts.yml`
- **Alertmanager:** `/opt/monitoring/alertmanager.yml`
- **Webhook:** `/opt/monitoring/webhook/receiver.py`

## Docker Compose

Services run via Docker Compose at `/opt/monitoring/docker-compose.yml`:
- **grafana** - Dashboards (port 3000)
- **prometheus** - Metrics + alerts (port 9090)
- **alertmanager** - Alert routing (port 9093)
- **cadvisor** - Container metrics (port 8080)
- **pihole-exporter** - Pi-hole metrics (port 9617)
- **webhook** - Discord webhook forwarder (port 5000)

## Installation

Deployed via Docker Compose on Ubuntu 24.04 LXC container.

### Docker Setup Notes

Requires specific config for unprivileged LXC:

```json
{ "apparmor": false, "storage-driver": "overlay2" }
```
