# Homelab Service Ideas (Future Roadmap)

Services to consider adding to the homelab stack, beyond what's already deployed.

## High Priority

### 🛡️ Vaultwarden
- Self-hosted Bitwarden-compatible password manager
- Single Docker container, ~50-100MB RAM
- Bitwarden apps/extension connect to your own server
- Family sharing for the household
- **Plan:** Docker on CT 106 or dedicated CT

### 🎬 Jellyseerr
- Media request management for Jellyfin/Radarr/Sonarr
- Family opens a web page, requests a movie/show, it auto-grabs
- Only makes sense *after* media stack (CT 108) is live
- **Plan:** Docker on CT 106 or media stack CT

## Medium Priority

### 📄 Paperless-ngx
- Document management: scan → OCR → auto-tag → searchable
- Receipts, tax forms, insurance, manuals
- Needs persistent storage for documents
- **Plan:** Docker or dedicated CT when needed

### 📡 Nginx Proxy Manager
- Reverse proxy with web UI + automatic Let's Encrypt SSL
- Makes accessing services (Nextcloud, Immich, Jellyfin) cleaner
- Can replace port-forwarding to individual services
- **Plan:** Docker on CT 106, before exposing anything publicly

## Lower Priority / Nice-to-Have

### 🖥️ Uptime Kuma
- Pretty uptime dashboard with push notifications
- Complements Grafana — simpler heartbeat view
- **Plan:** Docker on CT 106

### 📦 Gitea/Forgejo
- Self-hosted Git service for private repos
- Alternative to relying on GitHub for configs/dotfiles
- **Plan:** Docker on CT 106

### ⛏️ Minecraft Server
- Family Minecraft world for the household
- Minimal resources when idle
- **Plan:** Dedicated CT for performance when gaming

### 💰 Actual Budget
- Self-hosted budgeting app
- Sync across devices, offline-capable
- **Plan:** Docker on CT 106