# 10 - Grafana & Prometheus

Monitoring stack deployed on container 106 (10.2.7.108) for visualizing homelab metrics.

## Services

| Service | URL | Auth |
|---------|-----|------|
| **Grafana** | http://10.2.7.108:3000 | admin / admin |
| **Prometheus** | http://10.2.7.108:9090 | none |

## Installation

Deployed via Docker Compose on Ubuntu 24.04 LXC container.

### Docker Setup Notes

The container required specific configuration due to the unprivileged LXC environment:

- **AppArmor:** Disabled in `/etc/docker/daemon.json`:
  ```json
  { "apparmor": false, "storage-driver": "overlay2" }
  ```
- **Storage:** `overlay2` driver for compatibility

### Docker Compose (grafana.yml)

```yaml
services:
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - prometheus-data:/etc/prometheus
      - prometheus-data:/prometheus

volumes:
  grafana-data:
  prometheus-data:
```

## Future Dashboards

- [ ] Proxmox host metrics (CPU, RAM, disk)
- [ ] Container resource usage
- [ ] Pi-hole query stats
- [ ] Network throughput
- [ ] Uptime monitoring alerts
