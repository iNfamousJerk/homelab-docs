# 12 - Tailscale

## Overview

Real-world use: Tailscale creates a secure, private network between your devices that works from anywhere. Instead of exposing ports to the internet (risky) or setting up a traditional VPN (complex), you install Tailscale on each device and they magically find each other. It's like having your home network accessible from your phone at the coffee shop — securely, with zero config.

## Setup

- Installed on: Pi-hole container (CT 107, 10.2.7.2)
- Acts as: network gateway for remoting into the homelab
- Admin console: https://login.tailscale.com

## Why Tailscale (Not WireGuard)

- Zero config — no port forwarding needed on OPNsense
- Built-in NAT traversal (works through firewalls, CGNAT, hotel Wi-Fi)
- Device management via admin console
- Free for personal use (up to 3 users, 100 devices)
- Uses WireGuard protocol under the hood

## Accessing Homelab Remotely

1. Install Tailscale on your device (phone, laptop, etc.)
2. Sign in to the same Tailscale network (same account)
3. Access services via Tailscale IPs:
   - Wazuh Dashboard: http://100.x.x.x:443 (replace with actual Tailscale IP)
   - Grafana: http://100.x.x.x:3000
   - Homarr: http://100.x.x.x:7575
   - Any other CT IP via Tailscale
4. Or use MagicDNS names if enabled

## Maintenance

- Tailscale auto-updates on most platforms
- Check status: `tailscale status` (inside CT 107)
- Re-authenticate if device expires: `tailscale up`

## Troubleshooting

- Can't connect? Make sure the Pi-hole container is running (CT 107 is the exit node)
- Device not showing in admin console? Re-authenticate with `tailscale up`
- Speed issues? Tailscale uses relay servers when direct connection isn't possible (less common on mobile networks)