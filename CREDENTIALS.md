# 🔐 Credentials Reference

> **⚠️ IMPORTANT:** Passwords are redacted in this public repo. Real passwords stored securely offline.

## Proxmox

| Field | Value |
|-------|-------|
| URL | https://10.2.7.64:8006 |
| Username | root@pam |
| Password | [REDACTED] |

## Containers

| CT | Container | User | Password Theme | SSH |
|----|-----------|------|---------------|-----|
| 100 | Hermes Agent | hermes | N/A (AI agent) | Key-based |
| 101 | Immich | root | [REDACTED - lowercase themed] | Key-based |
| 102 | PiAlert | root | [REDACTED - lowercase themed] | Key-based |
| 103 | Homarr | root | [REDACTED - lowercase themed] | Key-based |
| 104 | Nextcloud | root | [REDACTED - lowercase themed] | Key-based |
| 105 | Wazuh Manager | root | [REDACTED - lowercase themed] | Key-based (root@10.2.7.110) |
| 106 | Grafana | root | [REDACTED - lowercase themed] | Key-based |
| 107 | Pi-hole | root | [REDACTED - lowercase themed] | Key-based |
| 108 | Pirate Media Stack | root | [REDACTED - lowercase themed] | Key-based |

> **Note:** All passwords are themed phrases related to each container's or service's function. Stored securely in Hermes memory.

## Services

| Service | URL | Default Login |
|---------|-----|---------------|
| Grafana | http://10.2.7.108:3000 | admin / [REDACTED] |
| Prometheus | http://10.2.7.108:9090 | No auth |
| Homarr | http://10.2.7.105 | Local only |
| Pi-hole | http://10.2.7.2/admin | Local only |
| PiAlert | http://10.2.7.77 | Local only |
| Wazuh Dashboard | https://10.2.7.110:443 | admin / [REDACTED] |
| Wazuh Manager API | https://10.2.7.110:55000 | API key auth |
| PVE Host | https://10.2.7.64:8006 | root@pam / [REDACTED] |
| NPM Admin (CT 108) | http://10.2.7.109:81 | admin@example.com / [REDACTED] |
| Vaultwarden Admin | http://10.2.7.109 (via NPM) | `/admin` — [REDACTED token] |
| Jellyfin | http://10.2.7.109:8096 | First user = admin |

## Notes

- All services are on local network (10.2.7.0/24) only
- Remote access via Tailscale VPN (100.x.x.x Tailscale IPs assigned to each node)
- Wazuh agent enrollment on port 1515 (passwordless)
- Wazuh manager SSH: root@10.2.7.110
- Hermes Agent API: http://10.2.7.107:8642 (health endpoint at /health)
