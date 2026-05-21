# 🏠 Homelab Documentation

**Owner:** Anthony Pipher (iNfamousJerk)  
**Started:** May 2026  
**Goal:** Hands-on networking experience → NOC / Jr. Network Admin role

## 📋 Overview

This homelab runs on a **Proxmox** hypervisor hosting multiple LXC containers serving various self-hosted services. All systems are on a local subnet managed by OPNsense with Tailscale for remote access.

## 🖥️ Server Specs

| Component | Spec |
|-----------|------|
| **CPU** | Intel i5-7500 (4 cores) |
| **RAM** | 31.2 GB |
| **Disk** | 94 GB (local-lvm) |
| **Hypervisor** | Proxmox VE 9.1.1 |
| **Hostname** | pve |
| **Management IP** | 10.2.7.64:8006 |
| **Uptime** | ~12 days |

## 📦 Container Inventory

| ID | Name | IP | Purpose | Status |
|----|------|----|---------|--------|
| 100 | Pi-hole | 10.2.7.2 | DNS + DHCP | ✅ Online |
| 101 | Immich | — | Photo backup | ✅ Online |
| 102 | PiAlert | 10.2.7.103 | Network monitoring | ✅ Online |
| 103 | Homarr | 10.2.7.105 | Dashboard | ✅ Online |
| 104 | Nextcloud | — | Cloud storage | ✅ Online |
| 105 | Hermes Agent | 10.2.7.107 | AI assistant | ✅ Online |
| 106 | Grafana | 10.2.7.108 | Monitoring dashboards | ✅ Online |

---

## 🛠️ How to Reproduce (Step by Step)

Each doc includes **exact copy-paste commands** for rebuilding from scratch:

### Infrastructure
| Doc | What It Covers |
|-----|----------------|
| [01-proxmox.md](01-proxmox.md) | Creating LXC containers from templates, managing via API |
| [02-containers.md](02-containers.md) | Container configs, Docker install inside LXC |
| [03-network.md](03-network.md) | Subnet, OPNsense, IP assignments |

### Monitoring (flagship project)
| Doc | What It Covers |
|-----|----------------|
| [10-grafana.md](10-grafana.md) | **Full monitoring stack** — Prometheus, Grafana, Alertmanager, cAdvisor, Pi-hole exporter, Discord alerts |
| [11-docker.md](11-docker.md) | Docker install, compose commands, AppArmor fix |

### Services
| Doc | What It Covers |
|-----|----------------|
| [04-pihole.md](04-pihole.md) | Pihole DNS + DHCP |
| [05-immich.md](05-immich.md) | Photo backup |
| [06-pialert.md](06-pialert.md) | Network monitoring |
| [07-homarr.md](07-homarr.md) | Dashboard |
| [08-nextcloud.md](08-nextcloud.md) | Cloud storage |
| [09-hermes-agent.md](09-hermes-agent.md) | AI assistant |

### Networking & Access
| Doc | What It Covers |
|-----|----------------|
| [12-tailscale.md](12-tailscale.md) | VPN access setup |
| [CREDENTIALS.md](CREDENTIALS.md) | Login reference (passwords redacted) |

---

## 📁 Repo Structure

```
homelab-docs/
├── README.md                   ← This file
├── 01-proxmox.md               — Server setup & API access
├── 02-containers.md            — All LXC containers + Docker setup
├── 03-network.md               — Subnet, OPNsense, IPs
├── 04-pihole.md                — DNS + DHCP
├── 05-immich.md                — Photo backup
├── 06-pialert.md               — Network monitoring
├── 07-homarr.md                — Dashboard
├── 08-nextcloud.md             — Cloud storage
├── 09-hermes-agent.md          — AI assistant
├── 10-grafana.md               — Full monitoring stack (Prometheus + Grafana + alerts)
├── 11-docker.md                — Docker setup, compose, troubleshooting
├── 12-tailscale.md             — VPN access
└── CREDENTIALS.md              — Login reference (passwords redacted)
```

## 🔮 Future Plans

- [ ] Voice-activated assistant (Raspberry Pi + mic/speaker)
- [ ] Animated Hermes display screen
- [ ] Portainer for Docker management
- [ ] Nginx Proxy Manager
- [ ] More Grafana dashboards
- [ ] Automated backups
- [ ] VLAN segmentation (IoT / management / trusted)
