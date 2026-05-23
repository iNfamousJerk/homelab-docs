# 05 - Immich

## Overview

Real-world use: Immich replaces Google Photos / iCloud Photos. You install the app on your phone, it auto-uploads every photo you take to your homelab. Facial recognition automatically groups photos of the same person. You can create shared albums, search for photos by content (machine learning), and access them from any device. No monthly subscription, no privacy concerns. Your photos stay on your hardware.

## Access

- **Web UI:** http://10.2.7.44:2283 (or via Homarr)
- **Mobile app:** iOS/Android — available on app stores (search "Immich")
- **SSH:** `root@10.2.7.44` (password: [REDACTED — see CREDENTIALS.md])

## User Manual

- **Sign in:** Open a web browser → go to http://10.2.7.44:2283 → create an account (or use the admin account created during setup)
- **Mobile upload:** Install the app → enter server URL `http://10.2.7.44:2283` → sign in → enable background backup
- **Create album:** Web UI → **Albums** → **New Album** → select photos → share link with others
- **Search:** Type anything into the search bar — "dog", "beach 2024", "birthday" — machine learning finds matching photos automatically
- **Download:** Select one or more photos → click the **Download** button

## Maintenance

### Update Immich (Docker-based)

```bash
pct enter 101
cd /opt/immich  # or wherever the immich docker-compose.yml lives
docker compose pull
docker compose up -d
```

### Update container OS

```bash
pct enter 101
apt update && apt upgrade -y
```

### Backup photos

Immich stores data in Docker volumes. Back up the volume directory regularly using your standard backup process.

## Tips

- Enable **Background Backup** in the mobile app for automatic uploads
- Photos are stored at **full resolution** by default (no compression)
- Use **Shared Links** to share albums with people outside your homelab
- Storage location: check the Docker volume mounts in the Immich compose file to find where media is actually stored on disk
- If you have iTunes or Vudu digital movie purchases you'd like to add to your media server, those can be uploaded separately and organized into albums or later imported into Jellyfin

## Troubleshooting

- **Uploads stuck?** Check disk space (`df -h`) — Immich uses significant storage as photos accumulate
- **Web UI not loading?** Run `docker compose ps` inside the container to verify all services are running
- **Mobile app can't connect?** Make sure you're on the same local network or connected via Tailscale