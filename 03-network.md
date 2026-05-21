# 03 - Network Configuration

## Subnet

- **Network:** 10.2.7.0/24
- **Gateway:** 10.2.7.1 (OPNsense)
- **DHCP:** Pi-hole (container 100, 10.2.7.2)
- **DNS:** Pi-hole (ad-blocking)

## IP Assignments

| Device | IP | Notes |
|--------|-----|-------|
| Gateway/OPNsense | 10.2.7.1 | Router/firewall |
| Pi-hole | 10.2.7.2 | DNS + DHCP |
| Proxmox | 10.2.7.64 | Web UI + API |
| PiAlert | 10.2.7.103 | Network monitoring |
| Homarr | 10.2.7.105 | Dashboard |
| Hermes Agent | 10.2.7.107 | AI assistant |
| Grafana | 10.2.7.108 | Monitoring dashboards |
| Immich | DHCP | Photo backup |
| Nextcloud | DHCP | Cloud storage |

## Remote Access

- **VPN:** Tailscale (ad-on on Pi-hole container)
- **No WireGuard** — Tailscale handles all remote access needs

## Bridge

All containers connect through Proxmox bridge `vmbr0` on the same /24 subnet.
