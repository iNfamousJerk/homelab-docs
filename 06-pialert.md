# 06 - PiAlert

PiAlert monitors the local network for new devices and sends notifications when unknown devices appear.

## Access

- **URL:** http://10.2.7.103
- **Port:** 80 (default)

## What It Does

- Scans the local subnet (10.2.7.0/24) periodically
- Tracks MAC addresses and hostnames
- Alerts on new/unrecognized devices
- Shows device history and online/offline status

## Configuration

- Scans the entire /24 subnet
- Reports for: 10.2.7.64 (Proxmox), 10.2.7.107 (Hermes Agent), 10.2.7.108 (Grafana)
- All containers visible on vmbr0 bridge
