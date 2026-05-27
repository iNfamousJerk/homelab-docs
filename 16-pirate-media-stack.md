# 🏴‍☠️ CT 108 — Pirate Media Stack

> **Purpose:** Media automation, streaming, and reverse proxy management
> **Host:** CT 108 (Proxmox container on PVE1)
> **IP:** `10.2.7.109`
> **OS:** Debian 13 (Trixie)
> **Specs:** 2 vCPUs · 5.9Gi RAM · 99G disk (7.3G used)

---

## 📦 Services

**10 containers** running via Docker Compose at `/opt/media-stack/docker-compose.yml`:

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

> **Note:** Vaultwarden was previously part of this stack but was **migrated to CT 104** and then **removed entirely on 2026-05-27** along with the rest of the office stack. See [08-nextcloud.md](08-nextcloud.md) for current Nextcloud state.

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

> **Pi-hole DNS:** Add a wildcard A record `*.pirate.lan → 10.2.7.109` for subdomain access.

---

## 🔐 Credentials

| Service | Login |
|---------|-------|
| NPM Admin | `http://10.2.7.109:81` — login: `anthonypiper1@gmail.com` / `nppass` (stored in compose comment; needs rotation) |
| Vaultwarden Admin | **Migrated to CT 104** — see nextcloud docs |

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

### Compose file overview

The compose file uses YAML anchors for common settings (PUID=1000, PGID=1000, TZ=America/New_York, DNS 1.1.1.1/1.0.0.1, restarts unless-stopped, AppArmor=unconfined). NPM config is stored in `./config/npm/`.

---

## 🛡️ Security Notes

- **No VPN yet** — qBittorrent runs without a tunnel. Add Gluetun when ProtonVPN arrives. The compose has a placeholder comment for this.
- **AppArmor** — set to `unconfined` for *arr services to avoid filesystem permission issues.
- **NPM** — provides SSL termination via Nginx. Currently uses self-signed certs; Let's Encrypt ready.
- **NPM default admin** — still using `admin@example.com/changeme`? Needs verification — compose comment shows `anthonypiper1@gmail.com / nppass` but the default NPM creds were `admin@example.com/changeme`.
- **Vaultwarden** — migrated off CT 108 to CT 104 (Nextcloud).
- **CT specs** — was originally 2GB RAM / 10GB disk but was resized to 5.9Gi RAM / 99G disk to handle media operations.

---

## 🔮 Future Plans

| Feature | Status | Notes |
|---------|--------|-------|
| Gluetun + ProtonVPN | ⬜ Planned | Routes qBittorrent through WireGuard tunnel |
| Let's Encrypt SSL | ⬜ Planned | Via NPM's built-in certificate management |
| Public Jellyfin | ⬜ Planned | Via Cloudflare Tunnel or VPN only |
| Move to PVE2 | ⬜ Planned | Media stack will migrate to dedicated PVE2 node |
| Hardware transcoding | ⬜ Future | Requires GPU passthrough (Intel QSV possible on PVE2) |
| NPM cred rotation | ⬜ Needed | Default creds still in use |

---

## 🛠️ Troubleshooting

| Issue | Fix |
|-------|-----|
| *Arr can't see downloads | Check `/data` bind mounts — qBittorrent downloads to `/data/torrents/`, *arrs expect `/data` |
| Jellyfin playback fails | Check file permissions — all containers run as PUID 1000 |
| NPM proxy not working | Verify Pi-hole wildcard `*.pirate.lan → 10.2.7.109` is set |
| Port conflict | CT 108 has ports 80/443 — cannot coexist with another container hosting a reverse proxy on same ports |
| Disk full | `df -h` — CT 108 has 99G total, 7.3G used. Monitor `/data` usage as media grows |

---

## Related

- [`docker-compose.yml`](configs/pirate-docker-compose.yml) — Full compose file
- [`services-roadmap.md`](services-roadmap.md) — Planned service upgrades
- [`SECURITY-CHECKLIST.md`](SECURITY-CHECKLIST.md) — Expansion security protocol
- CT 104 Nextcloud docs — Vaultwarden now runs there
