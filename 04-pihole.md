# 04 - Pi-hole

DNS sinkhole + DHCP server for the homelab.

## Access

- **IP:** 10.2.7.2
- **Web UI:** http://10.2.7.2/admin
- **Port:** 80 (admin), 53 (DNS)

## Role

- **DHCP Server:** Assigns IPs to devices on the 10.2.7.0/24 subnet
- **DNS Server:** Ad-blocking via blocklists
- **Tailscale Exit Node:** Tailscale add-on installed on this container

## Configuration

- Upstream DNS: Cloudflare (1.1.1.1) or Google (8.8.8.8)
- DHCP range: managed by Pi-hole
- Blocklists: Default + additional privacy lists

## Stats

Track queries blocked, top domains, and client activity via the admin web UI.
