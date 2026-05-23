# 04 - Pi-hole

## Overview

Real-world use: Pi-hole is a network-wide ad blocker. It sits between every device and the internet, and when a device asks "where is ads.example.com?", Pi-hole says "nowhere, that domain doesn't exist". It also handles DHCP (assigning IPs to everything on the network). Think of it as the traffic cop + ad blocker for your whole house.

## Access

- Web UI: http://10.2.7.2/admin
- SSH: root@10.2.7.2 (password: PiHoleIsKing)
- DNS port: 53

## User Manual

- How to check query stats: Web UI → Query Log or Dashboard
- How to whitelist/blacklist a domain: Web UI → Domain Management
- How to check DHCP leases: Web UI → Settings → DHCP
- How to temporarily disable blocking: Web UI → Disable for X seconds

## Maintenance

### Update Pi-hole
```bash
pct enter 107
pihole -up
```

### Update container OS
```bash
pct enter 107
apt update && apt upgrade -y
```

### Restart DNS service
```bash
pct enter 107 -- pihole restartdns
```

### Check disk space (8GB total, Pi-hole is lightweight)
```bash
df -h
```

## Tailscale

- Tailscale is installed on this container as an exit node
- Allows remote access to the entire homelab without exposing ports
- Check status: `tailscale status`

## Troubleshooting

- DNS not resolving? Check if Pi-hole is running: `pct enter 107 -- pihole status`
- Devices can't get IPs? Check DHCP is enabled in Web UI → Settings → DHCP
- resolv.conf issue on new containers: point to 10.2.7.2 for DNS

## Logs

- Query log: /var/log/pihole.log
- Web UI also has live query log viewer