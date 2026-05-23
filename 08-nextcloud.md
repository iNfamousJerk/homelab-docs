# 08 - Nextcloud

## Overview
Real-world use: Nextcloud is a self-hosted Dropbox + Google Drive alternative. You install the desktop/mobile app, pick a folder to sync, and your files stay in sync across all your devices. It also handles calendar sync (replaces Google Calendar), contacts sync (replaces Google Contacts), and file sharing via links. Key difference from Google Drive: your files never leave your hardware and there's no storage limit based on a paid subscription.

## Access
- Web UI: http://10.2.7.99
- Desktop apps: Windows/Mac/Linux (download from nextcloud.com)
- Mobile apps: iOS/Android (app stores)
- SSH: root@10.2.7.99

## User Manual
- First login: browser → http://10.2.7.99 → enter admin credentials
- Install desktop app: download from nextcloud.com → enter server URL (http://10.2.7.99) → choose sync folder
- Mobile app: install → enter server URL → enable photo backup (optional)
- Share a file: right-click file → Share → generate link → send link
- Calendar/Contacts: Settings → enable CalDAV/CardDAV → sync with phone calendar app

## Maintenance
### Update Nextcloud
Inside the container, the app update command:
```bash
pct enter 104
sudo -u www-data php /var/www/nextcloud/occ upgrade  # or similar path
```
Or if Docker-based (check setup method):
```bash
cd /opt/nextcloud
docker compose pull
docker compose up -d
```

### Update container OS
```bash
pct enter 104
apt update && apt upgrade -y
```

### Check disk usage
30GB disk, currently ~26GB free. Monitor via:
```bash
pct enter 104 -- df -h
```

## Tips
- Use desktop app for "local sync" — files available offline, syncs changes automatically
- Calendar/Contacts sync with phone: add CalDAV/CardDAV account, point to Nextcloud server
- File versioning: Nextcloud keeps file history — you can revert to older versions

## Troubleshooting
- Web page shows "Internal Server Error"? Check /var/log/nextcloud/ for logs
- Sync not working? Verify server URL is accessible (try accessing from browser)
- Running out of space? Check Nextcloud data directory size, clean up trash/deleted files
  In web UI: Settings → Administration → Trash bin → Empty