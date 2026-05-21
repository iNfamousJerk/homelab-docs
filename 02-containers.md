# 02 - LXC Containers

All containers run on Proxmox as LXC (Linux Containers). Most were deployed via [community-scripts](https://community-scripts.org).

## Container List

| ID | Hostname | IP | RAM | Cores | Disk | Purpose |
|----|----------|-----|-----|-------|------|---------|
| 100 | pihole | 10.2.7.2 | 2 GB | 2 | 8 GB | DNS + DHCP |
| 101 | immich | — | — | — | — | Photo backup |
| 102 | pialert | 10.2.7.103 | — | — | — | Network monitoring |
| 103 | homarr | 10.2.7.105 | 2 GB | 2 | 8 GB | Dashboard |
| 104 | nextcloud | — | — | — | — | Cloud storage |
| 105 | hermesagent | 10.2.7.107 | — | — | — | AI assistant |
| 106 | grafana | 10.2.7.108 | — | — | — | Monitoring |

## Container Features

- **Unprivileged:** Most containers are unprivileged for security
- **Nesting enabled:** containers 103+ have `nesting=1,keyctl=1` features
- **Boot order:** All set to `onboot=1` (auto-start)
- **OS:** Debian-based (Ubuntu 24.04 for grafana)

## Container 106 (Grafana) - Manual Setup

Created manually via Proxmox API from `ubuntu-24.04-standard_24.04-2_amd64.tar.zst` template:

```bash
# Created with:
net0: name=eth0,bridge=vmbr0,gw=10.2.7.1,ip=10.2.7.108/24
ostype: ubuntu
```

SSH access configured with key-based auth.
