# 🏠 Homelab Documentation

**Owner:** Anthony Piper (iNfamousJerk)  
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
| **Hypervisor** | Proxmox VE 9.2.2 |
| **Hostname** | pve |
| **Management IP** | 10.2.7.64:8006 |
| **Uptime** | varies (last reboot: 2026-05-24) |

## 🔐 Credentials

**All SSH passwords and service logins are centralized in [`CREDENTIALS.md`](CREDENTIALS.md).**  
Individual service docs reference credentials there — no passwords are stored in the public docs.

## 📦 Container Inventory

See **[02-containers.md](02-containers.md)** for the full container list with specs (RAM, cores, disk), or the quick table below:

| ID | Name | IP | Purpose |
|----|------|-----|---------|
| 100 | hermesagent | 10.2.7.107 | AI assistant |
| 101 | Immich | 10.2.7.44 | Photo backup |
| 102 | PiAlert | 10.2.7.77 | Network monitoring |
| 103 | Homarr | 10.2.7.105 | Dashboard |
| 104 | Nextcloud | 10.2.7.99 | Cloud storage |
| 105 | Wazuh | 10.2.7.110 | SIEM manager |
| 106 | Grafana | 10.2.7.108 | Monitoring stack (8 Docker containers) |
| 107 | Pi-hole | 10.2.7.2 | DNS + DHCP |

---

## 🛠️ How to Reproduce (Step by Step)

Each doc includes **exact copy-paste commands** for rebuilding from scratch:

### Operational Playbooks
| Doc | What It Covers |
|-----|----------------|
| [00-pve-pbs-lifecycle-playbook.md](00-pve-pbs-lifecycle-playbook.md) | **1,614-line definitive runbook** — create/restore/maintain/troubleshoot every VM, LXC, and PBS operation |
| [00-pve-pbs-rebuild-blueprint.md](00-pve-pbs-rebuild-blueprint.md) | **Script-free rebuild** — migrate away from community helper scripts to raw upstream git, official APT, hand-written systemd |

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
| [13-wazuh.md](13-wazuh.md) | Wazuh SIEM: agent-based threat detection, file integrity, vuln scanning |
| [monitoring-stack.md](monitoring-stack.md) | **Updated Dec 2026** — 20 alert rules, blackbox exporter probes (17 targets), Discord alerting pipeline |

### Services
| Doc | What It Covers |
|-----|----------------|
| [04-pihole.md](04-pihole.md) | Pihole DNS + DHCP |
| [05-immich.md](05-immich.md) | Photo backup |
| [06-pialert.md](06-pialert.md) | Network monitoring |
| [07-homarr.md](07-homarr.md) | Dashboard |
| [homarr-setup.md](homarr-setup.md) | **Homarr dashboard setup guide** — all services, widgets, recommended layout |
| [08-nextcloud.md](08-nextcloud.md) | Cloud storage |
| [09-hermes-agent.md](09-hermes-agent.md) | AI assistant |
| [13-wazuh.md](13-wazuh.md) | Wazuh SIEM security monitoring (all 8 CTs) |

### Networking & Access
| Doc | What It Covers |
|-----|----------------|
| [12-tailscale.md](12-tailscale.md) | VPN access setup |
| [14-ups-monitoring.md](14-ups-monitoring.md) | UPS monitoring — NUT, Prometheus, Grafana, Discord alerts |
| [CREDENTIALS.md](CREDENTIALS.md) | Login reference (passwords redacted) |
| [DR-IR-PLAYBOOK.md](DR-IR-PLAYBOOK.md) | Disaster recovery & incident response — 5 scenarios |
| [SOC-UPGRADE-PLAN.md](SOC-UPGRADE-PLAN.md) | Enterprise SOC upgrade — 4-phase plan |

---

## 📁 Repo Structure

This repo covers the homelab infrastructure. The media automation stack has its own repo:

- **📺 [media-stack](https://github.com/iNfamousJerk/media-stack)** — Radarr, Sonarr, Prowlarr, qBittorrent, Jellyfin, Lidarr, Bazarr

```
homelab-docs/
├── README.md                        ← This file
├── 00-pve-pbs-lifecycle-playbook.md — 4-pillar PVE/PBS operational runbook (1,614 lines)
├── 00-pve-pbs-rebuild-blueprint.md  — Script-free rebuild from scratch (1,438 lines)
├── 01-proxmox.md                    — Server setup & API access
├── 02-containers.md                 — All LXC containers + Docker setup
├── 03-network.md                    — Subnet, OPNsense, IPs
├── 04-pihole.md                     — DNS + DHCP
├── 05-immich.md                     — Photo backup
├── 06-pialert.md                    — Network monitoring
├── 07-homarr.md                     — Dashboard
├── 08-nextcloud.md                  — Cloud storage
├── 09-hermes-agent.md               — AI assistant
├── 10-grafana.md                    — Full monitoring stack (Prometheus + Grafana + alerts)
├── 11-docker.md                     — Docker setup, compose, troubleshooting
├── 12-tailscale.md                  — VPN access
├── 13-wazuh.md                      — Wazuh SIEM security monitoring
├── 14-ups-monitoring.md             — UPS monitoring (NUT · Prometheus · Grafana · Discord alerts)
├── 15-pbs-setup.md                  — Proxmox Backup Server hardware & integration
├── CREDENTIALS.md                   — Login reference (passwords redacted)
├── DR-IR-PLAYBOOK.md                — Disaster recovery & incident response
├── SOC-UPGRADE-PLAN.md              — Enterprise SOC upgrade roadmap
├── services-roadmap.md              — Planned services & hardware pipeline
├── docs/                            — Setup guides (Homarr, monitoring-stack)
└── PLANS/                           — Hardware upgrade plans (pirate-pbs, homelab-v2)
```

## 🔮 Future Plans

- [ ] **🏴‍☠️ Project Pirate PBS** — Dell Optiplex 7050 MT second node (see [PLANS/pirate-pbs.md](PLANS/pirate-pbs.md))
- [ ] USB Blu-ray + auto-ripper for physical media
- [ ] 2.5Gb network upgrade
- [ ] VLAN segmentation
- [ ] More Grafana dashboards
