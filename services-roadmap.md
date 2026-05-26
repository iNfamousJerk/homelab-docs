# Homelab Service Roadmap

> **Last updated:** May 25, 2026
>
> ✅ = Deployed | 🔄 = In Progress | ⬜ = Planned

---

## ✅ Deployed

| Service | Where | Purpose |
|---------|-------|---------|
| ✅ Proxmox VE | Host | Type-1 hypervisor |
| ✅ Proxmox Backup Server | PBS (Z230) | Immutable ZFS backups |
| ✅ OPNsense | Router | Firewall, VLAN, NUT client |
| ✅ Pi-hole | CT 107 | DNS + DHCP |
| ✅ Pi.Alert | CT 102 | Network device discovery |
| ✅ Wazuh SIEM | CT 105 | Security monitoring (10 agents) |
| ✅ Grafana + Prometheus | CT 106 | Metrics, dashboards, alerts |
| ✅ Uptime Kuma | CT 106 | Uptime monitoring (27 monitors) |
| ✅ Homarr | CT 103 | Service dashboard |
| ✅ Gitea | CT 106 | Self-hosted Git |
| ✅ Immich | CT 101 | Photo management |
| ✅ Nextcloud | CT 104 | Cloud storage |
| ✅ NUT UPS Monitoring | PVE + CT 106 + OPNsense | UPS status + graceful shutdown |
| ✅ Media Stack (Pirate) | CT 108 | qBittorrent, *arrs, Jellyfin, Jellyseerr |
| ✅ Vaultwarden | CT 108 | Self-hosted Bitwarden-compatible password manager |
| ✅ Nginx Proxy Manager | CT 108 | Subdomain routing, SSL termination for Pirate stack |
| ✅ Hermes Agent | CT 100 | AI assistant |
| ✅ Discord Alerting | — | Wazuh, Grafana, Uptime Kuma → Discord |

---

## ⬜ Recommended Next Services

### Priority 1 — Vaultwarden
Self-hosted Bitwarden-compatible password manager.

- ✅ **Deployed** on CT 108 as part of the pirate media stack
- Single Docker container, ~50MB RAM
- Bitwarden apps on phone/browser connect to your server
- Family sharing — Brittany, daughter each get their own vault
- Auto-fill on iPhone, MacBook, PC
- Access via NPM: `https://vaultwarden.pirate.lan`
- Next step: set up Bitwarden apps, import passwords, disable open signups

### Priority 2 — Nginx Proxy Manager
Web UI for the reverse proxy + automatic Let's Encrypt SSL certs.

- ✅ **Deployed** on CT 108 alongside the pirate stack
- Click-to-add proxy hosts instead of hand-editing nginx configs
- Auto-renewing SSL certificates via DNS-01 challenge
- Works with Pi-hole DNS wildcards
- Currently serves `*.pirate.lan` subdomains
- Next step: add Let's Encrypt SSL and expand to other CTs

### Priority 3 — Paperless-ngx
Document management: scan → OCR → auto-tag → searchable.

- Receipts, tax forms, insurance, manuals, medical records
- Full-text searchable PDFs with auto-generated tags/correspondent
- Mobile app for scanning with phone camera
- **Where:** Docker on CT 106 or dedicated CT with mounted storage
- **Why now:** Document accumulation never stops — better to start early than digitize 5 years of paper later.
- **Deps:** Persistent storage volume

### Priority 4 — Authentik (SSO)
Centralized single sign-on for all homelab services.

- Log in once → access Immich, Nextcloud, Jellyfin, Grafana, all arrs
- LDAP/SCIM provisioning for service-to-service auth
- MFA/TOTP/WebAuthn support
- **Where:** Docker on CT 106
- **Why later:** Most valuable after you have 5+ web services with separate logins — you're there now. But requires configuring each service to use OIDC/LDAP, which takes time per service.
- **Deps:** Each service must support OIDC/SAML/LDAP (most do)

### Priority 5 — ProtonVPN + Gluetun (Pirate Stack)
Route qBittorrent traffic through encrypted tunnel.

- Gluetun container wraps qBittorrent in WireGuard tunnel
- Built-in kill switch: VPN drops → traffic stops
- Already have a compose addon file ready — just need your ProtonVPN key
- **Where:** CT 108, Gluetun container
- **Deps:** ProtonVPN WireGuard config (you said you'll add it when you have it)

### Priority 6 — Minecraft Server
Family survival world for you, Brittany, and your daughter.

- Dedicated CT with 2-4 cores, 4GB RAM
- Easy to manage via Crafty Controller (web UI for server management)
- Bedrock + Java crossplay for whatever devices the family uses
- **Where:** Dedicated CT on PVE2 (future) or PVE1 if resources allow
- **Why later:** Fun family project once infrastructure is solid

### Priority 7 — Calibre-web
E-book management + OPDS feed.

- Organize, read, and download ebooks from browser
- Send books to Kindle or phone
- **Where:** Docker on CT 106
- **Why later:** Nice-to-have for reading on phone/tablet

### Priority 8 — Actual Budget
Self-hosted budgeting app.

- Sync across devices, offline-capable
- Track spending, set budgets, import bank CSVs
- **Where:** Docker on CT 106
- **Why later:** Good for household finances but lower infra impact

---

## Recommended Phasing

| Phase | What | Timeline |
|-------|------|----------|
| ✅ Media Stack | Vaultwarden + NPM + *arrs + Jellyfin | Deployed on CT 108 |
| 🔜 Now | Paperless-ngx + Authentik | Next |
| 🗓️ After VPN | Gluetun for Pirate | When ProtonVPN arrives |
| 🎯 Future | Minecraft + Calibre + Actual | After PVE2 is online |