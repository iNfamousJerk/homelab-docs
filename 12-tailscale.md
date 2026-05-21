# 12 - Tailscale

Tailscale provides secure VPN access to the homelab from anywhere.

## Setup

- **Installed on:** Pi-hole container (100)
- **Install method:** Add-on (Pi-hole supported)
- **Network:** Tailscale mesh network

## Why Tailscale (Not WireGuard)

- Zero config — no port forwarding needed
- Built-in NAT traversal
- Works through firewalls
- Device management via Tailscale admin console
- Free for personal use

## Accessing Homelab Remotely

1. Install Tailscale on your device
2. Authenticate to the same Tailscale network
3. Access homelab services via their Tailscale IPs or MagicDNS names

## Resources

- [Tailscale Admin Console](https://login.tailscale.com)
- Device name: pihole / homelab
