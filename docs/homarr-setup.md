# Homarr Dashboard Setup

**URL:** http://10.2.7.105:7575

Homarr is your **single-pane-of-glass dashboard** for the entire homelab. One place to find every service.

## Current Service Layout

### Row 1: Uncategorized (Top Row)

| Service | Type | URL | Icon |
|---------|------|-----|------|
| **Proxmox PVE** | URL | https://10.2.7.64:8006 | proxmox |
| **PBS** | URL | https://10.2.7.65:8007 | proxmox |

### Row 2: Monitoring

| Service | Type | URL | Icon |
|---------|------|-----|------|
| **Uptime Kuma** | URL | http://10.2.7.111:3001 | uptime-kuma |
| **Wazuh** | URL | https://10.2.7.110 | wazuh |
| **Grafana** | URL | http://10.2.7.108:3000 | grafana |
| **Prometheus** | URL | http://10.2.7.108:9090 | prometheus |
| **Hermes Agent** | URL | http://10.2.7.107:8642/health | terminal |
| **Alertmanager** | URL | http://10.2.7.108:9093 | prometheus |

### Row 3: Networking

| Service | Type | URL | Icon |
|---------|------|-----|------|
| **OPNsense** | URL | https://10.2.7.1 | opnsense |
| **Pi-hole** | URL | http://10.2.7.2/admin | pi-hole |
| **Pi.Alert** | URL | http://10.2.7.77 | pialert |

### Row 4: Storage & Tools

| Service | Type | URL | Icon |
|---------|------|-----|------|
| **Nextcloud** | URL | https://10.2.7.99 | nextcloud |
| **Immich** | URL | http://10.2.7.44:2283 | immich |
| **GitHub** | URL | https://github.com/iNfamousJerk?tab=repositories | github |
| **Gitea** | URL | http://10.2.7.108:3002 | gitea |

### (Future) Media Section — empty until CT 108 is deployed

| Service | Type | URL | Icon |
|---------|------|-----|------|
| **Jellyfin** | *(when set up)* | `http://10.2.7.X:8096` | jellyfin |

> **Removed from CT 104:** OnlyOffice, Vaultwarden, Actual Budget, LanguageTool — Docker stack cleaned up on 2026-05-27

## Adding a New Service

1. Click the **+** button in the top bar or the empty tile
2. Choose **"App"**
3. Fill in:
   - **Name:** Display name (e.g., "Grafana")
   - **URL:** The service URL
   - **Icon:** Search for the service name or upload a PNG
   - **Behaviour:** "Open in new tab" (recommended)
4. Click **Save**
5. Drag tiles to arrange your layout

## Maintenance

### Update Homarr
```bash
pct enter 103
source /opt/homarr.env
cd /opt/homarr
# Homarr runs as a systemd service, update instructions pending
```

### Update container OS
```bash
pct enter 103
apt update && apt upgrade -y
```

## Tips
- Drag tiles to reorder
- Resize by dragging bottom-right corner
- Change color scheme in **Settings → Appearance**
- Set dark mode for the full homelab vibe
- Enable **Search** for quick service switching
- If a service icon doesn't show, Homarr will fetch it from the URL favicon
- Homarr stores data in a SQLite DB at `/opt/homarr_db/db.sqlite`
- API available at `/api/openapi` with API key auth