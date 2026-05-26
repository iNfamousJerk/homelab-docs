# 02 - LXC Containers

## Overview
Real-world use: Think of each container as a dedicated computer for one job. Immich handles photos, Pi-hole handles DNS, etc. Containerization means if one breaks, the others keep running. LXC = lighter than a VM, nearly bare-metal speed.

## Container Inventory

| ID | Name | IP | RAM | Cores | Disk | Purpose | Boot Order |
|----|------|-----|-----|-------|------|---------|------------|
| PVE | pve | 10.2.7.64 | 31.2GB | 4 | ~670GB thin-lvm | Proxmox host | — |
| 107 | pihole | 10.2.7.2 | 4GB | 2 | 4GB | DNS + DHCP | 1 (up 30s) |
| 106 | grafana | 10.2.7.108 | 4GB | 2 | 30GB | Monitoring stack (10 Docker containers) | 2 (up 60s) |
| 102 | pialert | 10.2.7.77 | 512MB | 1 | 5GB | Network monitoring | 3 (up 20s) |
| 100 | hermesagent | 10.2.7.107 | 4GB | 2 | 20GB | AI assistant, Discord bot | 4 (up 20s) |
| 101 | immich | 10.2.7.44 | 6GB | 4 | 100GB | Photo backup | 5 (up 30s) |
| 103 | homarr | 10.2.7.105 | 2GB | 2 | 8GB | Dashboard | 6 (up 20s) |
| 105 | wazuh | 10.2.7.110 | 4GB | 4 | 24GB | Wazuh SIEM manager | 7 (up 30s) |
| 104 | nextcloud | 10.2.7.99 | 512MB | 1 | 30GB | Cloud storage | 8 (up 20s) |
| **108** | **pirate** | **10.2.7.109** | **2GB** | **2** | **10GB** | **Media stack (Docker) — temp on PVE1, ready for PVE2** | **9 (up 20s)** |

> ⚠️ **Boot order verified from live PVE config (2026-05-25):** Pi-hole boots first (DNS infra), then monitoring stack, then services. Containers with lower resources (PiAlert 512MB, Nextcloud 512MB) were downsized from their initial allocations — they run fine at these levels for current usage.

## Container Features
- Unprivileged (security)
- Nesting + keyctl enabled for Docker containers (CT 100, 101, 103, 105, 106, 107)
- Auto-start on boot with staggered startup sequence
- All storage on local-lvm (LVM thin pool, 670.7GB total on 1TB PNY SSD)
- No ZFS on PVE host — single SSD with LVM thin provisioning

## Maintenance

### Update all containers
```bash
# From PVE host, loop through all:
for ct in 100 101 102 103 104 105 106 107 108; do
  echo "=== Updating CT $ct ==="
  pct exec $ct -- apt update -qq 2>/dev/null
  pct exec $ct -- apt upgrade -y -qq 2>/dev/null
done
```

### Check disk usage on all containers
```bash
pct list | awk 'NR>1{printf "CT %-4s %-20s disk=%.1f/%.1fGB\n", $1, $2, $5, $6}'
```

### Backup a container
```bash
# To PBS (preferred):
vzdump <CT_ID> --storage pbs --mode snapshot --compress zstd

# To local storage:
vzdump <CT_ID> --compress zstd --mode stop
```

### Resize container disk
```bash
pct resize <CT_ID> rootfs <new_size>G
```

## SSH Access
- Hermes agent has SSH key deployed to all containers
- Password auth also available (see CREDENTIALS.md)

## Storage Layout

All 9 containers use `local-lvm` (LVM thin pool) on a single 1TB PNY CS900 SSD:

| Disk Image | Container | Size | Used |
|------------|-----------|------|------|
| vm-100-disk-0 | hermesagent (CT 100) | 20GB | 4.7GB |
| vm-101-disk-0 | immich (CT 101) | 100GB | 48.9GB |
| vm-102-disk-0 | pialert (CT 102) | 5GB | 1.4GB |
| vm-103-disk-0 | homarr (CT 103) | 8GB | 1.4GB |
| vm-104-disk-0 | nextcloud (CT 104) | 30GB | 2.5GB |
| vm-105-disk-0 | wazuh (CT 105) | 24GB | 16.1GB |
| vm-106-disk-0 | grafana (CT 106) | 30GB | 7.0GB |
| vm-107-disk-0 | pihole (CT 107) | 4GB | 1.2GB |
| vm-108-disk-0 | pirate (CT 108) | 10GB | 7.3GB |

**PBS backup storage:** 48.2GB used of 1.75TB available on `z230` (ZFS mirror, 2×2TB).

## Community Scripts
Many containers were initially set up using https://community-scripts.net — they provide one-command install scripts for popular self-hosted apps. See [00-pve-pbs-rebuild-blueprint.md](00-pve-pbs-rebuild-blueprint.md) for script-free rebuild instructions.

## Logs
Container logs on PVE host: `pct logs <CT_ID>`
Container syslog: `/var/log/syslog` inside each CT
