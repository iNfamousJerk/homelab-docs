# 🏴‍☠️ CT 108 — Pirate Media Stack

> **Purpose:** Media automation, streaming, and self-hosted password management
> **Host:** CT 108 (Proxmox container on PVE1)
> **IP:** `10.2.7.109`
> **OS:** Debian 13 (Trixie)
> **Specs:** 2 vCPUs · 2GB RAM · 10GB disk (7.3GB used)

---

## 📦 Services

| Container | Port | Purpose |
|-----------|------|---------|
| **Nginx Proxy Manager** (npm) | `80/443` (Web), `81` (Admin UI) | Reverse proxy, subdomain routing `*.pirate.lan` → CT 108 |
| **qBittorrent** | `8080` (Web UI), `6881` (TCP/UDP) | Torrent client |
| **Prowlarr** | `9696` | Indexer aggregator |
| **Flaresolverr** | `8191` | Cloudflare bypass for indexers |
| **Radarr** | `7878` | Movie automation |
| **Sonarr** | `8989` | TV show automation |
| **Lidarr** | `8686` | Music automation |
| **Bazarr** | `6767` | Subtitle management |
| **Jellyfin** | `8096` | Media streaming server |
| **Jellyseerr** | `5055` | Media request portal |

---

## 📂 Data Layout

```
/data/
├── media/       ← Jellyfin reads from here
│   ├── movies/
│   ├── tv/
│   └── music/
└── torrents/    ← qBittorrent downloads here
    ├── movies/
    ├── tv/
    └── music/
```

All *arr services have `- /data:/data` bind mounts so they share the same filesystem. Jellyfin only maps `/data/media` to avoid exposing incomplete downloads.

---

## 🌐 Network Access

| Endpoint | URL |
|----------|-----|
| NPM Admin | `http://10.2.7.109:81` |
| qBittorrent | `http://10.2.7.109:8080` |
| Prowlarr | `http://10.2.7.109:9696` |
| Flaresolverr | `http://10.2.7.109:8191` |
| Radarr | `http://10.2.7.109:7878` |
| Sonarr | `http://10.2.7.109:8989` |
| Lidarr | `http://10.2.7.109:8686` |
| Bazarr | `http://10.2.7.109:6767` |
| Jellyfin | `http://10.2.7.109:8096` |
| Jellyseerr | `http://10.2.7.109:5055` |
| Vaultwarden | Via NPM — `https://vaultwarden.pirate.lan` (internal) |

> **Pi-hole DNS:** Add a wildcard A record `*.pirate.lan → 10.2.7.109` for subdomain access.

---

## 🔐 Credentials

| Service | Login |
|---------|-------|
| NPM Admin | `http://10.2.7.109:81` — see [`CREDENTIALS.md`](CREDENTIALS.md) |
| Vaultwarden Admin | `/admin` — uses `ADMIN_TOKEN` from `.env` |
| Vaultwarden Users | Open signup (can be disabled in docker-compose.yml) |

All passwords stored in [`CREDENTIALS.md`](CREDENTIALS.md) and Hermes memory.

---

## 🐳 Docker Setup

Compose file: `/opt/media-stack/docker-compose.yml`
Config directory: `/opt/media-stack/config/`

```bash
# Start all services
cd /opt/media-stack && docker compose up -d

# Check status
docker ps

# View logs
docker compose logs -f <service_name>

# Stop everything
docker compose down
```

---

## 🛡️ Security Notes

- **No VPN yet** — qBittorrent runs without a tunnel. Add Gluetun when ProtonVPN arrives.
- **AppArmor** — set to `unconfined` for *arr services to avoid filesystem permission issues.
- **NPM** — provides SSL termination via Nginx. Currently uses self-signed certs; Let's Encrypt ready.
- **Vaultwarden** — behind NPM only. No public exposure. Admin panel on `/admin` with token auth.

---

## 🔮 Future Plans

| Feature | Status | Notes |
|---------|--------|-------|
| Gluetun + ProtonVPN | ⬜ Planned | Routes qBittorrent through WireGuard tunnel |
| Let's Encrypt SSL | ⬜ Planned | Via NPM's built-in certificate management |
| Public Jellyfin | ⬜ Planned | Via Cloudflare Tunnel or VPN only |
| Move to PVE2 | ⬜ Planned | Media stack will migrate to dedicated PVE2 node |
| Hardware transcoding | ⬜ Future | Requires GPU passthrough (Intel QSV possible on PVE2) |

---

## 🛠️ Troubleshooting

| Issue | Fix |
|-------|-----|
| *Arr can't see downloads | Check `/data` bind mounts — qBittorrent downloads to `/data/torrents/`, *arrs expect `/data` |
| Jellyfin playback fails | Check file permissions — all containers run as PUID 1000 |
| NPM proxy not working | Verify Pi-hole wildcard `*.pirate.lan → 10.2.7.109` is set |
| Port conflict | CT 108 has ports 80/443 — cannot coexist with another container hosting a reverse proxy on same ports |
| Disk full | `df -h` — CT 108 has only 10GB; `/data` is inside the container rootfs. No separate mount yet. |

---

## Related

- [`docker-compose.yml`](configs/pirate-docker-compose.yml) — Full compose file
- [`services-roadmap.md`](services-roadmap.md) — Planned service upgrades
- [`SECURITY-CHECKLIST.md`](SECURITY-CHECKLIST.md) — Expansion security protocol
