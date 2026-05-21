# 01 - Proxmox Server

## Hardware

- **CPU:** Intel i5-7500 @ 3.40GHz (4 cores)
- **RAM:** 31.2 GB total
- **Storage:** ~94 GB on local-lvm (LVM thin pool)
- **OS:** Proxmox Virtual Environment 9.1.1

## Networking

- **Hostname:** pve
- **Web UI:** https://10.2.7.64:8006
- **Subnet:** 10.2.7.0/24
- **Gateway:** 10.2.7.1 (OPNsense)

## API Access

The Proxmox API is accessible at:

```
POST https://10.2.7.64:8006/api2/json/access/ticket
```

Authentication payload:
```json
{
  "username": "root@pam",
  "password": "<password>"
}
```

Returns a ticket (cookie) and CSRFPreventionToken for subsequent requests.

### Useful API Commands

```bash
# List all containers
curl -k -b "PVEAuthCookie=$TICKET" \
  https://10.2.7.64:8006/api2/json/nodes/pve/lxc

# Get container config
curl -k -b "PVEAuthCookie=$TICKET" \
  https://10.2.7.64:8006/api2/json/nodes/pve/lxc/103/config

# Get node status
curl -k -b "PVEAuthCookie=$TICKET" \
  https://10.2.7.64:8006/api2/json/nodes/pve/status
```

## Resource Usage

| Resource | Usage |
|----------|-------|
| CPU | ~3% idle |
| RAM | ~19% |
| Disk | ~14% |
| Swap | ~4% |
