# 10 - Grafana & Prometheus

Monitoring stack deployed on container 106 (10.2.7.108) for visualizing homelab metrics.

## Services

| Service | URL | Auth |
|---------|-----|------|
| **Grafana** | http://10.2.7.108:3000 | admin / GraphMyWorld |
| **Prometheus** | http://10.2.7.108:9090 | none |
| **Alertmanager** | http://10.2.7.108:9093 | none |
| **cAdvisor** | http://10.2.7.108:8080 | none |
| **Pi-hole Exporter** | http://10.2.7.108:9617 | none |

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

## Alerts (Discord Notifications)

Prometheus alerts evaluated every 30s, forwarded to Discord via Alertmanager:

| Alert | Threshold | Severity |
|-------|-----------|----------|
| InstanceDown | `up == 0` for 1m | 🔴 Critical |
| HighDiskUsage | Disk >85% for 5m | 🟡 Warning |
| DiskAlmostFull | Disk >95% for 2m | 🔴 Critical |
| HighMemoryUsage | RAM >90% for 5m | 🟡 Warning |
| HighCPUUsage | CPU >90% for 10m | 🟡 Warning |
| NodeExporterDown | node_exporter data missing | 🔴 Critical |
| PiholeExporterDown | Pi-hole metrics missing | 🟡 Warning |

---

## How to Deploy the Full Monitoring Stack

Follow these steps to recreate the monitoring stack on a fresh container.

### Step 1: Prerequisites

```bash
# SSH into CT 106 (or your target container)
ssh root@10.2.7.108

# Verify Docker is installed
docker ps
docker compose version
```

### Step 2: Create Directory Structure

```bash
mkdir -p /opt/monitoring/{dashboards,webhook,provisioning/datasources,provisioning/dashboards}
```

### Step 3: Prometheus Config

Create `/opt/monitoring/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s

rule_files:
  - /etc/prometheus/homelab_alerts.yml

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_pve'
    static_configs:
      - targets: ['10.2.7.64:9100']

  - job_name: 'node_containers'
    static_configs:
      - targets: ['10.2.7.108:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['10.2.7.108:8080']

  - job_name: 'pihole'
    static_configs:
      - targets: ['10.2.7.108:9617']
```

### Step 4: Alert Rules

Create `/opt/monitoring/homelab_alerts.yml`:

```yaml
groups:
  - name: homelab_alerts
    interval: 30s
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.job }} is DOWN"
          description: "{{ $labels.job }} ({{ $labels.instance }}) has been unreachable for over 1 minute."

      - alert: HighDiskUsage
        expr: (1 - (node_filesystem_avail_bytes{fstype!="tmpfs",fstype!="overlay"} / node_filesystem_size_bytes{fstype!="tmpfs",fstype!="overlay"})) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk >85% full on {{ $labels.instance }}"
          description: "{{ $labels.mountpoint }} is {{ $value | humanizePercentage }} full"

      - alert: DiskAlmostFull
        expr: (1 - (node_filesystem_avail_bytes{fstype!="tmpfs",fstype!="overlay"} / node_filesystem_size_bytes{fstype!="tmpfs",fstype!="overlay"})) * 100 > 95
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "🚨 DISK CRITICAL on {{ $labels.instance }}"
          description: "{{ $labels.mountpoint }} is {{ $value | humanizePercentage }} full!"

      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "RAM >90% used on {{ $labels.instance }}"
          description: "Memory usage at {{ $value | humanizePercentage }}"

      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 90
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "CPU >90% on {{ $labels.instance }}"
          description: "CPU usage at {{ $value | humanizePercentage }} for 10+ minutes"
```

### Step 5: Alertmanager Config

Create `/opt/monitoring/alertmanager.yml`:

```yaml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'job']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'discord'

receivers:
  - name: 'discord'
    webhook_configs:
      - url: 'http://webhook:5000/alert'
        send_resolved: true
```

### Step 6: Discord Webhook Receiver

Create `/opt/monitoring/webhook/receiver.py`:

```python
from http.server import HTTPServer, BaseHTTPRequestHandler
import json
import urllib.request
import time

DISCORD_WEBHOOK = "https://discord.com/api/webhooks/YOUR_WEBHOOK_URL"

def send_discord(embed):
    payload = {
        "username": "Homelab Alerts",
        "embeds": [embed]
    }
    req = urllib.request.Request(
        DISCORD_WEBHOOK,
        data=json.dumps(payload).encode(),
        headers={"Content-Type": "application/json"},
        method="POST"
    )
    try:
        urllib.request.urlopen(req)
        return True
    except Exception as e:
        print(f"Discord send failed: {e}")
        return False

def format_alert(alert):
    status = alert.get("status", "firing")
    labels = alert.get("labels", {})
    annotations = alert.get("annotations", {})

    if status == "firing":
        color = 0xFF0000
        title = f"🚨 FIRING: {annotations.get('summary', labels.get('alertname', 'Unknown'))}"
    else:
        color = 0x00FF00
        title = f"✅ RESOLVED: {annotations.get('summary', labels.get('alertname', 'Unknown'))}"

    embed = {
        "title": title,
        "description": annotations.get("description", ""),
        "color": color,
        "fields": [
            {"name": "Alert", "value": labels.get("alertname", "N/A"), "inline": True},
            {"name": "Severity", "value": labels.get("severity", "N/A"), "inline": True},
            {"name": "Job", "value": labels.get("job", "N/A"), "inline": True},
            {"name": "Instance", "value": labels.get("instance", "N/A"), "inline": True},
        ]
    }
    return embed

class AlertHandler(BaseHTTPRequestHandler):
    def do_POST(self):
        content_len = int(self.headers.get("Content-Length", 0))
        body = self.rfile.read(content_len)
        data = json.loads(body)
        for alert in data.get("alerts", []):
            embed = format_alert(alert)
            send_discord(embed)
            time.sleep(0.5)
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.end_headers()
        self.wfile.write(json.dumps({"status": "ok"}).encode())

    def log_message(self, fmt, *args):
        if args:
            print(f"[Webhook] {fmt % args}")
        else:
            print(f"[Webhook] {fmt}")

if __name__ == "__main__":
    server = HTTPServer(("0.0.0.0", 5000), AlertHandler)
    print("Webhook receiver listening on :5000")
    server.serve_forever()
```

### Step 7: Provisioning Configs

Create `/opt/monitoring/provisioning/datasources/prometheus.yml`:

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://10.2.7.108:9090
    isDefault: true
```

Create `/opt/monitoring/provisioning/dashboards/dashboards.yml`:

```yaml
apiVersion: 1
providers:
  - name: Default
    orgId: 1
    folder: 
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    options:
      path: /var/lib/grafana/dashboards
```

### Step 8: Download Dashboard JSONs

```bash
# Node Exporter Full (ID: 1860)
curl -s -o /opt/monitoring/dashboards/node-exporter-full.json \
  https://grafana.com/api/dashboards/1860/revisions/latest/download

# Docker Container & Host Metrics (ID: 10619)  
curl -s -o /opt/monitoring/dashboards/docker-monitoring.json \
  https://grafana.com/api/dashboards/10619/revisions/latest/download
```

**Note:** The Docker dashboard needs datasource references patched for provisioning.
Replace `${DS_PROMETHEUS}` with `{"type": "prometheus", "uid": "PBFA97CFB590B2093"}`
(either manually or via the patching script in the homelab monitoring skill).

### Step 9: Docker Compose

Create `/opt/monitoring/docker-compose.yml`:

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - 9090:9090
    volumes:
      - prometheus_data:/prometheus
      - /opt/monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - /opt/monitoring/homelab_alerts.yml:/etc/prometheus/homelab_alerts.yml:ro
    restart: unless-stopped
    security_opt:
      - apparmor:unconfined

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
      - 8080:8080
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    devices:
      - /dev/kmsg
    privileged: true
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - 3000:3000
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=GraphMyWorld
    volumes:
      - grafana_data:/var/lib/grafana
      - /opt/monitoring/provisioning/:/etc/grafana/provisioning/:ro
      - /opt/monitoring/dashboards/:/var/lib/grafana/dashboards:ro
    restart: unless-stopped
    security_opt:
      - apparmor:unconfined

  pihole-exporter:
    image: ekofr/pihole-exporter:latest
    container_name: pihole-exporter
    ports:
      - 9617:9617
    environment:
      - PIHOLE_HOSTNAME=10.2.7.209
      - PIHOLE_PASSWORD=PiHoleIsKing
      - PIHOLE_PROTOCOL=http
      - PIHOLE_PORT=80
    restart: unless-stopped
    security_opt:
      - apparmor:unconfined

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    ports:
      - 9093:9093
    volumes:
      - /opt/monitoring/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
    restart: unless-stopped
    security_opt:
      - apparmor:unconfined

  webhook:
    image: python:3-alpine
    container_name: webhook
    ports:
      - 5000:5000
    volumes:
      - /opt/monitoring/webhook/receiver.py:/app/receiver.py:ro
    command: python /app/receiver.py
    restart: unless-stopped
    security_opt:
      - apparmor:unconfined

volumes:
  prometheus_data:
  grafana_data:
```

### Step 10: Deploy

```bash
cd /opt/monitoring

# Start everything
docker compose up -d

# Monitor startup
docker compose logs -f

# Check status
docker compose ps

# Expected output — all 6 services up:
# NAMES             STATUS
# webhook           Up
# pihole-exporter   Up
# prometheus        Up
# grafana           Up
# alertmanager      Up
# cadvisor          Up
```

### Step 11: Verify

```bash
# Check Prometheus targets
curl -s http://10.2.7.108:9090/api/v1/targets | \
  python3 -c "import json,sys; data=json.load(sys.stdin); [print(t['labels']['job'],':',t['health']) for t in data['data']['activeTargets']]"

# Check Prometheus rules
curl -s http://10.2.7.108:9090/api/v1/rules

# Check Grafana
curl -s -u admin:GraphMyWorld http://10.2.7.108:3000/api/health
# Should return: {"database":"ok","version":"..."}

# Check Alertmanager
curl -s http://10.2.7.108:9093/api/v2/status

# List Grafana dashboards
curl -s -u admin:GraphMyWorld http://10.2.7.108:3000/api/search?type=dash-db

# Test Discord alert pipeline
curl -s -X POST http://localhost:5000/alert \
  -H 'Content-Type: application/json' \
  -d '{"status":"firing","alerts":[{"status":"firing","labels":{"alertname":"Test","severity":"info"},"annotations":{"summary":"🧪 Test alert"}}]}'
```

## Proxmox & CT 106 node_exporter

These run **outside Docker**, directly on the host OS:

```bash
# Install on PVE host (10.2.7.64)
ssh root@10.2.7.64
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
tar xvf node_exporter-1.8.2.linux-amd64.tar.gz
cp node_exporter-1.8.2.linux-amd64/node_exporter /usr/local/bin/

cat > /etc/systemd/system/node_exporter.service << 'SERVICE'
[Unit]
Description=Node Exporter
After=network.target
[Service]
User=root
ExecStart=/usr/local/bin/node_exporter
Restart=always
[Install]
WantedBy=multi-user.target
SERVICE

systemctl daemon-reload
systemctl enable --now node_exporter
systemctl status node_exporter
```

Repeat the same steps for CT 106 (10.2.7.108) to get container-level host metrics.

## Quick Reference Commands

```bash
# SSH into CT 106
ssh root@10.2.7.64 "pct enter 106"

# Restart one service
docker compose -f /opt/monitoring/docker-compose.yml restart prometheus

# View all logs
docker compose -f /opt/monitoring/docker-compose.yml logs -f

# Grafana login reset
docker compose -f /opt/monitoring/docker-compose.yml exec grafana \
  grafana-cli admin reset-admin-password NEWPASSWORD
```
