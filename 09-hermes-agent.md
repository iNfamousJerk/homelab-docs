# 09 - Hermes Agent

The AI assistant running on container 105 (10.2.7.107). Provides autonomous assistant capabilities integrated with the homelab.

## Access

| Method | URL |
|--------|-----|
| **Health Endpoint** | http://10.2.7.107:8642/health |
| **Chat** | Via Discord DM |
| **Display UI** | http://10.2.7.107:8080 (animated robot) |

## Capabilities

- 💬 **Chat interface** — Discord, Telegram, SMS
- 🖥️ **Proxmox API** — Manage containers, monitor resources
- 📊 **Health checks** — Pollable JSON endpoint
- ⏰ **Cron jobs** — Automated server status reports every 6h via Discord
- 🎨 **Display page** — Animated robot UI on port 8080
- 💡 **Smart home** — Philips Hue control (via skill)

## Integration Points

### Homarr Widget
Add as Custom API widget at `http://10.2.7.107:8642/health` for green checkmark status.

### Automated Server Reports
Cron job checks Proxmox every 6 hours (midnight, 6am, noon, 6pm PDT) and reports via Discord:
- PVE host: CPU, RAM, disk usage, uptime
- Container statuses (running/stopped across all CTs)
- Health flags (high disk, stopped containers)

## Display Page

The animated robot display runs on port 8080:
- Cute robot avatar with animated eyes and mouth
- Real-time online/offline status
- Connection metrics (latency, uptime)
- Auto-polling health endpoint
- Dark theme with scanlines and particles
