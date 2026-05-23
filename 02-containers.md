# 02 - LXC Containers

## Overview
Real-world use: Think of each container as a dedicated computer for one job. Immich handles photos, Pi-hole handles DNS, etc. Containerization means if one breaks, the others keep running. LXC = lighter than a VM, nearly bare-metal speed.

## Container Inventory

| ID | Name | IP | RAM | Cores | Disk | Purpose |
|----|------|-----|-----|-------|------|---------|
| PVE | pve | 10.2.7.64 | 32GB | 4 | ~94GB | Proxmox host |
| 100 | hermesagent | 10.2.7.107 | 4GB | 2 | 16GB | AI assistant, Discord bot |
| 101 | immich | 10.2.7.44 | 4GB | 2 | 30GB | Photo backup |
| 102 | pialert | 10.2.7.77 | 2GB | 2 | 5GB | Network monitoring |
| 103 | homarr | 10.2.7.105 | 2GB | 2 | 8GB | Dashboard |
| 104 | nextcloud | 10.2.7.99 | 4GB | 2 | 30GB | Cloud storage |
| 105 | wazuh | 10.2.7.110 | 4GB | 4 | 20GB | Wazuh SIEM manager |
| 106 | grafana | 10.2.7.108 | 4GB | 2 | 20GB | Monitoring stack |
| 107 | pihole | 10.2.7.2 | 2GB | 2 | 8GB | DNS/DHCP |

## Container Features
- Unprivileged (security)
- Nesting enabled for Docker containers
- Auto-start on boot

## Maintenance

### Update all containers
```bash
# From PVE host, loop through all:
for ct in 100 101 102 103 104 105 106 107; do
  echo "=== Updating CT $ct ==="
  pct exec $ct -- apt update -qq 2>/dev/null
  pct exec $ct -- apt upgrade -y -qq 2>/dev/null
done
```

### Check disk usage on all containers
```bash
pct list | while read line; do echo "$line"; done
# Or check individually:
pct exec <CT_ID> -- df -h
```

### Backup a container
```bash
pct stop <CT_ID>
vzdump <CT_ID> --compress zstd --mode stop
pct start <CT_ID>
```

### Resize container disk
```bash
pct resize <CT_ID> rootfs <new_size>G
# Then SSH in and expand filesystem
```

## SSH Access
- Hermes agent has SSH key deployed to all containers
- Password auth also available (see CREDENTIALS.md)
- Default root password for most CTs: [REDACTED — themed phrases, see CREDENTIALS.md]

## Community Scripts
Many containers were initially set up using https://community-scripts.net - they provide one-command install scripts for popular self-hosted apps.

## Logs
Container logs on PVE host: `pct logs <CT_ID>`
Container syslog: `/var/log/syslog` inside each CT