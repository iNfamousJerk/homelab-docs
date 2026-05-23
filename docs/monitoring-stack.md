# Monitoring Stack (CT 106)

**IP:** 10.2.7.108

The monitoring container runs 8 Docker services on a single Debian container that monitor the entire homelab.

## Services

| Service | Port | Purpose |
|---------|------|---------|
| **Prometheus** | 9090 | Metrics collection + alert evaluation |
| **Grafana** | 3000 | Dashboards (infra + Wazuh security events) |
| **Alertmanager** | 9093 | Alert routing to Discord |
| **cAdvisor** | 8080 | Container-level metrics |
| **Node Exporter** | 9100 | Host-level metrics (CPU, RAM, Disk) |
| **Pi-hole Exporter** | 9617 | Pi-hole DNS metrics |
| **Blackbox Exporter** | 9115 | HTTP/TCP service health probes |
| **Webhook** | 5000 | Discord alert delivery |

## Access

- **Grafana:** http://10.2.7.108:3000
- **Prometheus:** http://10.2.7.108:9090
- **Alertmanager:** http://10.2.7.108:9093

## Prometheus Scrape Targets

### Standard Scrapes
| Job | Target | What It Monitors |
|-----|--------|-----------------|
| `prometheus` | localhost:9090 | Prometheus itself |
| `node_pve` | 10.2.7.64:9100 | PVE host (CPU, RAM, disk) |
| `node_containers` | 10.2.7.108:9100 | CT 106 host metrics |
| `cadvisor` | 10.2.7.108:8080 | All Docker containers |
| `pihole` | 10.2.7.108:9617 | Pi-hole DNS stats |

### Blackbox HTTP Probes
| Target | Service |
|--------|---------|
| http://10.2.7.44:2283 | Immich |
| http://10.2.7.77 | PiAlert |
| http://10.2.7.2 | Pi-hole Web |
| http://10.2.7.108:3000 | Grafana |

### Blackbox TCP Probes
| Target | Service |
|--------|---------|
| 10.2.7.110:9200 | Wazuh Indexer |
| 10.2.7.110:443 | Wazuh Dashboard |
| 10.2.7.110:55000 | Wazuh API |
| 10.2.7.2:53 | Pi-hole DNS |
| 10.2.7.107:22 | Hermes CT 100 SSH |
| 10.2.7.64:22 | PVE host SSH |
| 10.2.7.64:8006 | Proxmox Web UI |
| 10.2.7.99:443 | Nextcloud |

## Alert Rules (20 total)

### System (9 rules)
| Alert | Condition | Severity |
|-------|-----------|----------|
| **InstanceDown** | Target unreachable >1m | critical |
| **NodeExporterMissing** | Node exporter data absent | critical |
| **HighDiskUsage** | Disk >85% for 5m | warning |
| **DiskAlmostFull** | Disk >95% for 2m | critical |
| **HighMemoryUsage** | RAM >90% for 5m | warning |
| **CriticalMemoryUsage** | RAM >97% for 2m | critical |
| **HighCPUUsage** | CPU >90% for 10m | warning |
| **HighLoadAverage** | Load > cores×2 for 5m | warning |
| **ExtremeLoadAverage** | Load > cores×4 for 10m | critical |

### Container (5 rules)
| Alert | Condition | Severity |
|-------|-----------|----------|
| **ContainerOOM** | Any OOM events detected | critical |
| **ContainerHighMemoryMB** | Container >500MB RAM for 5m | info |
| **ContainerHighCPU** | Container >0.8 CPU cores for 10m | warning |
| **ContainerDiskUsage** | Container disk >85% for 5m | warning |
| **ContainerNetworkErrors** | Any network errors detected | warning |

### Services (6 rules)
| Alert | Condition | Severity |
|-------|-----------|----------|
| **PiholeStatus** | Pi-hole not "enabled" | warning |
| **PiholeExporterDown** | No Pi-hole metrics for 2m | warning |
| **PiholeBlockRateDrop** | Block rate <5% for 15m | info |
| **ServiceHTTPDown** | HTTP probe fails for 2m | critical |
| **ServiceTCPDown** | TCP probe fails for 2m | critical |
| **CadvisorDown** | No container metrics for 2m | critical |

## Alert Delivery

All alerts route through:
1. Prometheus evaluates rules every 30s
2. Alertmanager groups and deduplicates
3. Webhook (Python) formats as Discord embeds
4. Delivered to **Homelab Alerts** webhook in Discord

## Grafana Dashboards

1. **Node Exporter Full** — CPU/RAM/Disk/Network across PVE host + CT 106
2. **Docker Monitoring** — cAdvisor container resource usage
3. **Wazuh Security Events** — Wazuh alerts from OpenSearch datasource

## Configuration Files

All config lives in `/opt/monitoring/` on CT 106:
- `docker-compose.yml` — service definitions
- `prometheus.yml` — scrape targets
- `homelab_alerts.yml` — alert rules
- `alertmanager.yml` — Discord webhook routing
- `blackbox.yml` — HTTP/TCP probe modules
- `webhook/receiver.py` — Discord embed formatter
- `provisioning/` — Grafana datasource auto-config
- `dashboards/` — Grafana dashboard JSON

## Restart

```bash
cd /opt/monitoring && docker compose up -d
# Reload Prometheus config without restart:
docker exec prometheus kill -HUP 1
```