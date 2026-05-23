# 03 - Network Configuration

## Overview

Real-world use: This is the plumbing that connects everything. OPNsense is the firewall/router (your front door), Pi-hole is the DNS server (the phone book + ad blocker), and everything talks over the 10.2.7.0/24 subnet.

## Network Map

```
ISP ─── OPNsense (.1) ─── vmbr0 (Proxmox bridge) ─── All containers (.2-.110)
           │                     │
           │               Pi-hole (.2) - DNS/DHCP
      WAN: 192.168.0.171    
```

## Key Details

- **Subnet:** 10.2.7.0/24
- **Gateway:** 10.2.7.1 (OPNsense)
- **DNS:** 10.2.7.2 (Pi-hole, ad-blocking)
- **DHCP:** Pi-hole assigns IPs
- **WAN:** 192.168.0.171/24 (DHCP from upstream router)

## OPNsense

- **Web UI:** https://10.2.7.1 (port 443)
- **SSH:** root@10.2.7.1 (port 22, offers menu — option 8 for shell)
- **OS:** OPNsense 26.1.6_2 on FreeBSD 14.3

### Maintenance

- Update via Web UI: System → Firmware → Updates → Check for updates
- Or SSH menu option 12
- Always reboot after major updates

## Pi-hole (DNS/DHCP)

- **Web UI:** http://10.2.7.2/admin
- Handles all DNS queries with ad-blocking
- DHCP assigns static leases for containers, dynamic for clients

## IP Assignments

| Device | IP | Notes |
|--------|-----|-------|
| OPNsense | 10.2.7.1 | Gateway, firewall |
| Pi-hole | 10.2.7.2 | DNS, DHCP |
| PVE Host | 10.2.7.64 | Proxmox |
| Immich | 10.2.7.44 | Photo backup |
| PiAlert | 10.2.7.77 | Network monitoring |
| Nextcloud | 10.2.7.99 | Cloud storage |
| Homarr | 10.2.7.105 | Dashboard |
| Hermes Agent | 10.2.7.107 | AI assistant |
| Grafana | 10.2.7.108 | Monitoring |
| Wazuh | 10.2.7.110 | SIEM manager |

## Remote Access

- Tailscale installed on Pi-hole container (.2)
- No port forwarding exposed — all remote access through Tailscale

## Troubleshooting

- **Container can't reach internet?** Check `/etc/resolv.conf` inside — should point to `10.2.7.2`
- **Known issue:** fresh PVE containers get `/etc/resolv.conf` pointing to `10.2.7.209` (old DNS), fix: `echo 'nameserver 10.2.7.2' > /etc/resolv.conf`
- **Can't SSH?** Check OPNsense firewall rules allowing LAN traffic