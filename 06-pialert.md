# 06 - PiAlert

## Overview
Real-world use: PiAlert watches your network like a security camera for devices. When a new phone, laptop, or unknown device connects to your Wi-Fi, PiAlert notices and alerts you. It scans the full 10.2.7.0/24 subnet and tracks MAC addresses. Real-world analogy: a neighborhood watch that logs every car that enters your street and flags ones it's never seen before.

## Access
- Web UI: http://10.2.7.77
- SSH: root@10.2.7.77

## User Manual
- How to view known devices: Web UI → Devices section
- How to check recent alerts: Web UI → Events/History
- How to mark a device as known/trusted: click device → Mark as Known
- How to set up notifications: Settings → Notification config

## Maintenance
### Update PiAlert
```bash
pct enter 102
cd /opt/pialert  # or wherever installed
# If installed via git:
git pull
# Restart service
docker compose restart  # if docker-based
```

### Update container OS
```bash
pct enter 102
apt update && apt upgrade -y
```

### Disk info
Total: 5GB (resized from 2GB on 2026-05-22)
Check: df -h
Usage: ~31% (~3.3GB free)

## Logs
- PiAlert logs: /opt/pialert/log/ (or wherever logs are stored)
- Check Web UI for event history

## Troubleshooting
- No devices showing? Check if ARP scan is working: arp -a
- Container run out of disk? Was resized from 2GB to 5GB - if filling up again, check pialert logs for excessive logging
- Can't reach web UI? Check service is running