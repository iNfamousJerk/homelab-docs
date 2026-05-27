# 07 - Homarr

## Overview
Real-world use: Homarr is your homelab's homepage. Instead of remembering 10 different IP addresses and ports, you open one dashboard and see every service with its current status. Think of it as a mission control center for your homelab. Shows live status indicators — green = working, red = something's down.

## Access
- Web UI: http://10.2.7.105:7575
- SSH: root@10.2.7.105

## User Manual
First-time setup or adding new services:
1. Open http://10.2.7.105:7575
2. Click the edit/pencil icon (top-right)
3. Click "+" to add a new tile
4. Choose widget type:
   - URL/iframe: embed a service page (e.g. http://10.2.7.108:3000 for Grafana)
   - API: ping a health endpoint for live status (e.g. http://10.2.7.107:8642/health)
5. Name it, enter the URL, save
6. Drag tiles to arrange your layout

## Current Layout

Homarr is organized into category sections (top to bottom):

| Section | Services | Count |
|---------|----------|-------|
| **Monitoring** | Uptime Kuma, Wazuh, Grafana, Prometheus, Hermes, Alertmanager | 6 |
| **Storage** | NextCloud, Immich, GitHub, Gitea | 4 |
| **Networking** | OPNsense, Pi-hole, PiAlert, Nginx Proxy Manager, PBS | 5 |
| **Media** | qBittorrent, Radarr, Sonarr, Prowlarr, Bazarr, Lidarr, FlareSolverr, Jellyfin, Jellyseerr | 9 |
| **Office** | OnlyOffice, Vaultwarden, Actual Budget, LanguageTool | 4 |

All services have health ping URLs — Homarr shows green/red status dots next to each tile.

### Quick Add (via SQLite)
Services can be added directly to the SQLite DB for automation:
```bash
ssh root@10.2.7.105 "sqlite3 /opt/homarr_db/db.sqlite"
# See the homelab-dashboard skill for full schema and examples
```

## Maintenance
### Update Homarr
```bash
pct enter 103
# If Docker-based:
cd /opt/homarr
docker compose pull
docker compose up -d
```

### Update container OS
```bash
pct enter 103
apt update && apt upgrade -y
```

## Adding New Services to the Homelab
When you deploy a new service, add it to Homarr so you can always find it from one place.

## Custom Theme
A cyberpunk dark theme is available in the homelab-docs repo:
- File: `themes/homarr-cyberpunk.css`
- Paste the entire contents into Homarr → Settings → Custom CSS → Save
- Cyan-accent glass cards, JetBrains Mono font — matches the rest of the homelab aesthetic

## Troubleshooting
- Dashboard not loading? Check Homarr is running
- Widget showing red/offline? The target service might be down — check that service directly
- SSH into container and check Docker status: `docker ps`

## Logs
- Homarr logs: `docker compose logs` (inside the container, in the Homarr directory)