# 01 - Proxmox VE

## Overview
Proxmox is the hypervisor that runs ALL containers. Think of it as the landlord of the apartment building — every container lives inside it. Real-world use: running multiple isolated services on one physical machine, ability to snapshot/backup/restore any container, hot-resize resources.

This PVE host uses **LVM thin provisioning** (`local-lvm`) on a single 1TB PNY CS900 SSD — no ZFS on this host. Backups are offloaded to a separate **Proxmox Backup Server** (see [15-pbs-setup.md](15-pbs-setup.md)).

## Quick Reference
- Web UI: https://10.2.7.64:8006
- SSH: root@10.2.7.64
- API: https://10.2.7.64:8006/api2/json
- Hostname: pve
- OS: PVE 9.2.2 (Debian 12 Bookworm base)
- Storage: local-lvm (LVM thin pool, 670.7GB), local (ISOs/templates, 28GB), pbs (backup target)

## User Manual
- How to access web UI (browser → https://10.2.7.64:8006)
- How to check container list (`pct list` from SSH or API)
- How to SSH into a container (`pct enter <CT_ID>`)
- How to view logs (`pct logs <CT_ID>`)
- How to check resource usage (Web UI → Node → Summary)

## Container Boot Order

Staggered startup to avoid resource contention after AC power restoration. Order verified from live PVE config (2026-05-25):

| Order | CT | Name | Up Delay | Purpose |
|-------|----|------|----------|---------|
| 1 | 107 | pihole | 30s | DNS + DHCP — must be first |
| 2 | 106 | grafana | 60s | Docker monitoring stack |
| 3 | 102 | pialert | 20s | Network monitoring |
| 4 | 100 | hermesagent | 20s | AI assistant |
| 5 | 101 | immich | 30s | Photo backup |
| 6 | 103 | homarr | 20s | Dashboard |
| 7 | 105 | wazuh | 30s | SIEM manager |
| 8 | 104 | nextcloud | 20s | Cloud storage |

```bash
# Check current boot order for all containers:
for ct in $(pct list | awk 'NR>1{print $1}'); do
  echo -n "CT $ct: "; pct config $ct | grep -E "onboot|startup" | tr '\n' ' '; echo
done
```

## Maintenance
### Update Proxmox itself
```bash
ssh root@10.2.7.64
apt update && apt dist-upgrade -y
```
Then reboot if kernel was updated. Check with: `[ -f /var/run/reboot-required ] && echo "REBOOT NEEDED"`

### Resize a container disk
```bash
pct resize <CT_ID> rootfs <new_size>G
# No filesystem resize needed for LXC — container sees new space immediately
```

### Snapshot before risky changes
```bash
pct snapshot <CT_ID> before-update --description "Before XYZ change"
# Rollback: pct rollback <CT_ID> before-update
```

### Backup container to PBS
```bash
vzdump <CT_ID> --storage pbs --mode snapshot --compress zstd
```

## Storage

### PVE Storage Pools

| Pool | Type | Total | Used | Content |
|------|------|-------|------|---------|
| local | Directory | ~28GB | ~61GB* | vztmpl, iso, snippets, backup |
| local-lvm | LVM-Thin | 670.7GB | 123.1GB | images, rootdir (all CTs) |
| pbs | PBS (remote) | 1.75TB | 48.2GB | backup |

*\*The "local" pool's used>total is misleading — it includes shared template files that are cached.*

### Disk Mapping

All 8 containers live on `local-lvm` (LVM thin pool). No ZFS on this host.

| Disk Image | CT | Size | Used | Format |
|------------|----|------|------|--------|
| vm-100-disk-0 | hermesagent | 20GB | 4.7GB | raw |
| vm-101-disk-0 | immich | 100GB | 48.9GB | raw |
| vm-102-disk-0 | pialert | 5GB | 1.4GB | raw |
| vm-103-disk-0 | homarr | 8GB | 1.4GB | raw |
| vm-104-disk-0 | nextcloud | 30GB | 2.5GB | raw |
| vm-105-disk-0 | wazuh | 24GB | 16.1GB | raw |
| vm-106-disk-0 | grafana | 30GB | 7.0GB | raw |
| vm-107-disk-0 | pihole | 4GB | 1.2GB | raw |

## PVE ↔ PBS Integration
- **Storage target:** `pbs` (type: PBS)
- **Server:** 10.2.7.65:8007
- **Datastore:** main
- **Username:** backup-user@pbs
- **Encryption:** not currently enabled
- **Fingerprint:** B1:8D:A7:DE:3B:94:39:1B:6F:99:CF:24:74:87:A7:B1:46:C1:AA:1B:0A:BF:9E:22:0A:E0:B9:E6:8C:BA:06:C4

## Helper Scripts
For PVE API authentication from external tools (e.g., Hermes Agent monitoring cron jobs), use the Proxmox API directly:

```bash
TICKET=$(curl -sk -d 'username=root%40pam&password=<password>' \
  https://10.2.7.64:8006/api2/json/access/ticket | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['data']['ticket'])")
```

## Logs & Monitoring
- System logs: `/var/log/syslog`, `/var/log/pve/tasks/`
- Container logs: `pct logs <CT_ID>`
- PVE version: 9.2.2
- Prometheus node_exporter: 10.2.7.64:9100 (on PVE host, scraped by Prometheus on CT 106)

## Troubleshooting
- Can't reach Web UI? Check: `systemctl status pveproxy`
- Container won't start? `pct start <CT_ID> --debug`
- Disk full? Check: `df -h` on PVE host, then `pct resize` if needed
- Locked container? `pct unlock <CT_ID>`
