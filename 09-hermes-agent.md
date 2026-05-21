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
- ⏰ **Cron jobs** — Scheduled tasks
- 🎨 **Display page** — Animated robot UI on port 8080
- 💡 **Smart home** — Philips Hue control (via skill)

## Integration Points

### Homarr Widget
Add as Custom API widget at `http://10.2.7.107:8642/health` for green checkmark status.

### Grafana/Proxmox Monitoring
Cron job checks Proxmox every 6 hours and reports:
- CPU, RAM, disk, swap usage
- Container online/offline status
- Uptime alerts

## Display Page

The animated robot display runs on port 8080:
- Cute robot avatar with animated eyes and mouth
- Real-time online/offline status
- Connection metrics (latency, uptime)
- Auto-polling health endpoint
- Dark theme with scanlines and particles
