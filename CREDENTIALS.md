# 🔐 Credentials Reference

> **⚠️ IMPORTANT:** Passwords are redacted in this public repo. Real passwords stored securely offline.

## Proxmox

| Field | Value |
|-------|-------|
| URL | https://10.2.7.64:8006 |
| Username | root@pam |
| Password | [REDACTED] |

## Containers

| Container | User | Password | SSH |
|-----------|------|----------|-----|
| 106 - Grafana (Ubuntu) | root | [REDACTED] | Key-based auth |

## Services

| Service | URL | Default Login |
|---------|-----|---------------|
| Grafana | http://10.2.7.108:3000 | admin / admin |
| Prometheus | http://10.2.7.108:9090 | No auth |
| Homarr | http://10.2.7.105 | Local only |
| Pi-hole | http://10.2.7.2/admin | Local only |
| PiAlert | http://10.2.7.103 | Local only |

## Notes

- All services are on local network (10.2.7.0/24) only
- Remote access via Tailscale VPN
- Default credentials should be changed for production use
- Hermes Agent API: http://10.2.7.107:8642 (health endpoint at /health)
