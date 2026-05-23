# Homarr Dashboard Setup

**URL:** http://10.2.7.105:7575

Homarr is your **single-pane-of-glass dashboard** for the entire homelab. One place to find every service.

## Initial Setup

1. Open http://10.2.7.105:7575 in a browser
2. Create your admin account (this will be the first user)
3. You're in — start adding services

## Recommended Board Layout

### Row 1: Monitoring (top row, 3 columns)

| Service | Type | URL | Icon |
|---------|------|-----|------|
| **Grafana** | URL | http://10.2.7.108:3000 | grafana |
| **Wazuh** | URL | https://10.2.7.110 | wazuh |
| **Pi-hole** | URL | http://10.2.7.2/admin | pi-hole |

### Row 2: Media & Storage

| Service | Type | URL | Icon |
|---------|------|-----|------|
| **Immich** | URL | http://10.2.7.44:2283 | immich |
| **Nextcloud** | URL | http://10.2.7.99 | nextcloud |
| **Jellyfin** | URL *(when set up)* | http://10.2.7.X:8096 | jellyfin |

### Row 3: Infrastructure

| Service | Type | URL | Icon |
|---------|------|-----|------|
| **Proxmox** | URL | https://10.2.7.64:8006 | proxmox |
| **PiAlert** | URL | http://10.2.7.77 | pialert |
| **Homarr** | URL *(itself)* | http://10.2.7.105:7575 | homarr |

### Row 4: Security & Tools

| Service | Type | URL | Icon |
|---------|------|-----|------|
| **Wazuh Manager** | URL | https://10.2.7.110 | wazuh |
| **Hermes (CT 100)** | URL | http://10.2.7.107:XXX | terminal |
| **PVE Host** | URL | https://10.2.7.64:8006 | server |

## Widgets to Add (per row)

### Top Row: Monitoring Widgets
1. **Docker Container Monitor** — shows running containers on CT 106
2. **Resource Monitor** — CPU/RAM/Disk of CT 106 (or PVE host)

### Middle Section
3. **Date & Time** widget
4. **Weather** widget (Las Vegas, NV)

### Bottom
5. **Bookmarks** — common tools
   - GitHub: https://github.com/iNfamousJerk
   - Proxmox: https://10.2.7.64:8006
   - OPNsense: http://10.2.7.1

## Adding a Service

1. Click the **+** button in the top bar or the empty tile
2. Choose **"App"**
3. Fill in:
   - **Name:** Display name (e.g., "Grafana")
   - **URL:** The service URL (e.g., http://10.2.7.108:3000)
   - **Icon:** Search for the service name or upload a PNG
   - **Behaviour:** "Open in new tab" (recommended)
4. Click **Save**

## Adding the Docker Widget

1. Click **+** → **"Integration"**
2. Choose **"Docker"**
3. Set Docker socket path: `/var/run/docker.sock`
4. Save — now you see all containers on CT 106 running live

## Tips

- Drag tiles to reorder
- Resize by dragging bottom-right corner
- Change color scheme in **Settings → Appearance**
- Set dark mode for the full homelab vibe
- Enable **Search** for quick service switching
- If a service icon doesn't show, Homarr will fetch it from the URL favicon
