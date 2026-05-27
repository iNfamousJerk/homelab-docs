# 📊 CT 109 — Uptime Kuma Monitoring

> **Purpose:** Self-hosted uptime monitoring for all homelab services
> **Host:** CT 109 (Proxmox container on PVE1)
> **IP:** `10.2.7.111`
> **OS:** Debian 13 (Trixie)
> **Specs:** 1 vCPU · 2GB RAM · 20GB disk

---

## 📋 Overview

Uptime Kuma monitors all homelab services and sends Discord alerts on downtime. Previously ran inside the Grafana Docker stack on CT 106; now standalone on its own CT.

---

## 🐳 Docker Setup

Compose file: `/opt/kuma/docker-compose.yml`
Data directory: `/opt/kuma/data/`

```bash
# Start
cd /opt/kuma && docker-compose up -d

# Check status
docker ps

# View logs
docker-compose logs -f

# Stop
docker-compose down
```

### Compose
```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:latest
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - "3001:3001"
    volumes:
      - ./data:/app/data
```

---

## 🌐 Network Access

| Endpoint | URL | Notes |
|----------|-----|-------|
| Direct | `http://10.2.7.111:3001` | Internal access only |
| Via NPM | `https://kuma.pirate.lan` | SSL via Nginx Proxy Manager, resolves to 10.2.7.111:3001 |

---

## 🔐 Credentials

| Service | Login |
|---------|-------|
| Kuma Admin | **admin / kumapower** (change after initial setup) |

---

## 🔔 Notifications

- **Discord webhook** — pre-configured with the homelab alerts webhook
- Uses the same Discord channel as Grafana alerts
- Notification name: "Discord Alerts" — username: "UptimeKuma-Homelab"

---

## 🛡️ Security Notes

- Exposed only via NPM at `kuma.pirate.lan` with SSL
- Direct port 3001 is LAN-only (no firewall rules blocking inter-CT traffic on this subnet)
- Admin credentials should be changed from defaults

---

## 🛠️ Migration Notes

Kuma was migrated from CT 106 (Grafana stack) to standalone CT 109:

1. Old Kuma data volume (`monitoring_uptime_kuma_data`) still exists on CT 106 if monitor config needs extraction
2. Fresh install — no monitors migrated automatically; add via Kuma UI
3. Discord webhook pre-configured — just add monitors

---

## Related

- [`10-grafana.md`](10-grafana.md) — Main monitoring stack (Prometheus, Grafana, alerts)
- [`docs/homarr-setup.md`](docs/homarr-setup.md) — Dashboard with Kuma link