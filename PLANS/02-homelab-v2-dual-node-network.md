# Piper Homelab v2 — Dual Node + Network Expansion

> **Status:** Draft / Proposed
> **Author:** Hermes Agent
> **Date:** 2026-05-23

---

## Overview

Phase 2 of the homelab: Add a second PVE node for the media stack, a dedicated PBS node for backups, overhaul the network to handle it all, and segment IoT/guest traffic. Current PVE1 keeps running the existing CTs untouched.

**End state:**
- **PVE1** (existing) — CTs 100–107, stays as-is
- **PVE2** (new HP ProDesk) — CT 108: Docker media stack (Radarr, Sonarr, Prowlarr, qBittorrent, Jellyfin) + 2×1TB RAID for media
- **PBS** (new HP ProDesk) — Proxmox Backup Server, 2×2TB ZFS mirror for backups
- **Network** — Managed switch + patch panel + rack shelf + WiFi AP for IoT & Guest VLANs

---

## Part 1: PVE2 — Media Stack Node

### Goal

A dedicated Proxmox node that hosts CT 108 with the Docker-based media automation stack. Separate from PVE1 so media downloads and transcoding don't compete with critical homelab services (Pi-hole, Wazuh, Grafana, etc.).

### Architecture

```
┌─ PVE2 (HP ProDesk) ─────────────────────────────────────┐
│                                                          │
│  ┌─ CT 108 (Alpine/Debian, 2GB RAM, 10GB root) ─────┐  │
│  │                                                    │  │
│  │  ┌── Gluetun (NordVPN tunnel) ───────────────┐    │  │
│  │  │  │                                          │    │  │
│  │  │  ├── qBittorrent ──── downloads ──────┐    │    │  │
│  │  │  ├── Prowlarr (indexers)              │    │    │  │
│  │  │  ├── Radarr (movies) ──── rename ─────┤    │    │  │
│  │  │  ├── Sonarr (shows)  ──── rename ─────┤    │    │  │
│  │  │  │                                    │    │    │  │
│  │  │  └── Jellyfin ────── streaming ───────┘    │    │  │
│  │  │         (local only, no VPN)                │    │  │
│  │  └─────────────────────────────────────────────┘    │  │
│  │                                                     │  │
│  │  Bind mount: /mnt/media (2×1TB RAID) ←────────────┘  │
│  │                                                     │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

### Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Media storage | **ZFS mirror (2×1TB)** | Redundancy, snapshots, easy expand later |
| Docker host | **LXC (CT 108)** | Isolated from hypervisor, portable, PBS-backable |
| VPN container | **Gluetun** | Most maintained Docker VPN gateway. Built-in kill switch. |
| VPN protocol | **WireGuard** | Faster than OpenVPN. NordVPN supports it natively. |
| Only qBit through VPN | **Gluetun network mode** | Jellyfin doesn't need VPN. Use `network_mode: service:gluetun` for *arrs + qBit only |
| Jellyfin access | **Tailscale + local LAN** | No reverse proxy, no public exposure |

### Hardware Specs

| Component | Spec |
|-----------|------|
| Machine | HP ProDesk 400/600 G3-G5 (i5-6500 or better) |
| RAM | 8GB+ (16GB ideal) |
| Boot drive | 256GB SSD (NVMe or SATA) |
| Media drives | 2×1TB SATA HDDs (ZFS mirror) |

### Implementation Plan

1. Obtain 2× matching HP ProDesk units
2. Install Proxmox VE on both (USB installer or PXE)
3. Configure networking on PVE2 (static IP in 10.2.7.0/24)
4. On PVE2: create CT 108 (Debian 12, nesting enabled, 2GB RAM, 10GB disk)
5. On PVE2: zpool create with 2×1TB drives: `zpool create media mirror /dev/sda /dev/sdb`
6. On PVE2: create ZFS dataset: `zfs create media/media`
7. On PVE2: bind mount CT 108 to the dataset via PVE config: `mp0: /media/media,/mnt/media`
8. Inside CT 108: install Docker + compose
9. Inside CT 108: deploy Gluetun with NordVPN WireGuard config
10. Inside CT 108: deploy qBittorrent with `network_mode: service:gluetun`
11. Inside CT 108: deploy Radarr, Sonarr, Prowlarr with VPN network mode
12. Inside CT 108: deploy Jellyfin with host network (no VPN)
13. Back up CT 108 to PBS

### Pitfalls

| Issue | Fix |
|-------|-----|
| Hard links break across filesystems | Ensure media downloads and media library are on SAME ZFS dataset, not separate mounts |
| qBittorrent initial password | Capture from `docker logs qbittorrent` on first start, or set via env |
| Gluetun kill switch blocks everything | Verify `FIREWALL_OUTBOUND_SUBNETS` includes Tailscale/local subnet so Jellyfin can still stream |
| ZFS on LXC bind mount needs permissions | Set `unprivileged: 1` in CT config and map user/group IDs in `/etc/pve/lxc/108.conf` |
| LXC nesting + Docker not working | Ensure CT has `features: nesting=1` in PVE config |

---

## Part 2: PBS — Backup Server Node

### Goal

A dedicated Proxmox Backup Server to handle daily/weekly backups of all CTs from both PVE1 and PVE2.

### Architecture

```
┌─ PBS (HP ProDesk) ─────────────────────────────┐
│                                                 │
│  ┌── ZFS mirror: 2×2TB ──────────────────────┐ │
│  │  datastore: /backup/homelab               │ │
│  │  dedup + compression: on                  │ │
│  │                                            │ │
│  │  Daily backups from:                      │ │
│  │  ├── PVE1: CTs 100-107                    │ │
│  │  └── PVE2: CT 108                         │ │
│  └────────────────────────────────────────────┘ │
│                                                 │
│  IP: 10.2.7.50 (or next available)              │
│  Web UI: https://10.2.7.50:8007                 │
└─────────────────────────────────────────────────┘
```

### Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Backup target | **Proxmox Backup Server** | Native PVE integration, incremental forever, dedup |
| Backup storage | **ZFS mirror (2×2TB)** | Redundancy + PBS dedup stretches effective capacity |
| Backup schedule | **Daily at 3am, keep 7 daily + 4 weekly** | Standard 3-2-1 approach |
| Prune & GC | **Weekly** | Prevents datastore bloat |

### Implementation Plan

1. Install Proxmox Backup Server on the second HP ProDesk
2. Configure static IP (10.2.7.50 or next available)
3. Create ZFS pool: `zpool create backups mirror /dev/sda /dev/sdb`
4. Create PBS datastore: `proxmox-backup-manager datastore create homelab /backup/homelab`
5. Add PBS as backup target on PVE1: Datacenter → Storage → Add → PBS
6. Add PBS as backup target on PVE2 (same)
7. Create backup jobs for all CTs (daily, keep 7)
8. Set up prune job: `proxmox-backup-manager prune-job create --datastore homelab --keep-daily 7 --keep-weekly 4`

### Pitfalls

| Issue | Fix |
|-------|-----|
| PVE can't reach PBS on setup | Ensure both on same subnet (10.2.7.0/24) and firewall allows port 8007 |
| PBS datastore runs out of space | Monitor with `pbs_datastore_usage` metric in Grafana |
| Restore from PBS not working | Test a restore DRILL immediately after setup: restore a small CT to a temp ID |

---

## Part 3: Network — Switch, Patch Panel, Rack Shelf

### Goal

Clean up the spaghetti, get the HP ProDesks rack-mounted, and give everything a proper home.

### Architecture

```
┌─ Wall jack ──→ Patch Panel ──→ Switch ──→ PVE1, PVE2, PBS, etc.
                    │                            │
                    │                            ├── OPNsense (router)
                    │                            ├── All CTs
                    │                            └── Future: AP
                    │
                    └── Shelf rack unit(s)
                         ├── PVE1 (existing)
                         ├── PVE2 (new ProDesk)
                         └── PBS (new ProDesk)
```

### Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Switch | **TP-Link TL-SG1024D or Ubiquiti USW-Lite-16-PoE** | 24-port gigabit, quiet, rackmount. UBNT if future AP uses PoE. |
| Patch panel | **24-port Cat6 keystone patch panel** | Matches switch, clean cabling |
| Rack shelf | **2U vented shelf** | HP ProDesks sit on shelf (not rack ears) |
| Cables | **0.5ft-1ft Cat6 patch cables** | Short, clean, no spaghetti |

### Implementation Plan

1. Mount patch panel in rack (top)
2. Mount switch below patch panel
3. Install shelf below switch
4. Place PVE1, PVE2, PBS on shelf
5. Run permanent wall drops → patch panel rear
6. Patch panel front → switch with short cables
7. Server NICs → switch with short cables

### Pitfalls

| Issue | Fix |
|-------|-----|
| ProDesks are shallow — might need half-depth shelf | Measure depth of ProDesk before buying shelf. Most desktop racks are 450-600mm |
| Rack might be open frame or enclosed | Open frame is fine for ventilation. Enclosed needs fan consideration |
| Existing cables might not reach new switch | Plan cable routing. Use longer drops behind rack if needed |

---

## Part 4: WiFi — IoT & Guest AP

### Goal

Add a wireless access point with VLAN-separated SSIDs for IoT devices and guest users. Keeps untrusted devices off the main homelab subnet.

### Architecture

```
┌─ OPNsense ─────────────────────────────────────┐
│                                                 │
│  VLAN 10 (main): 10.2.7.0/24   ← PVE homelab   │
│  VLAN 20 (IoT):  10.2.20.0/24  ← Smart devices  │
│  VLAN 30 (guest): 10.2.30.0/24 ← Guest WiFi     │
│                                                 │
│  └── Trunk port → Switch → AP (tagged VLANs)    │
└─────────────────────────────────────────────────┘
```

### Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| AP hardware | **Ubiquiti U6 Lite** or **TP-Link EAP610** | Affordable, VLAN-supporting, reliable |
| AP management | **TP-Link Omada (standalone mode)** or **UniFi (controller)** | UBNT needs controller (can run in Docker on a CT). Omada works standalone. |
| IoT isolation | **Separate VLAN, no access to 10.2.7.0/24** | IoT devices can't reach homelab services |
| Guest isolation | **Separate VLAN, internet only** | No device-to-device or LAN access |

### Implementation Plan

1. On OPNsense: create VLAN interfaces (10, 20, 30) with DHCP scopes
2. On OPNsense: create firewall rules:
   - VLAN 10 → internet: allow
   - VLAN 20 → internet: allow, → VLAN 10: deny
   - VLAN 30 → internet: allow only, inter-VLAN: deny
3. On switch: configure trunk port to AP with tagged VLANs 10,20,30
4. Configure AP with 3 SSIDs: `Piper-Homelab`, `Piper-IoT`, `Piper-Guest`
5. Assign each SSID to its VLAN
6. Test: connect via guest SSID, verify you can't ping anything on 10.2.7.0/24

### Pitfalls

| Issue | Fix |
|-------|-----|
| Guest WiFi slow | Throttle via OPNsense traffic shaper |
| IoT devices can't reach internet | Check VLAN 20 has NAT rule and proper gateway on OPNsense |
| mDNS not working across VLANs | Set up Avahi/reflector on OPNsense or a dedicated CT for AirPlay/Chromecast bridging |

---

## Data Flow (End-to-End)

```
Torrent download                          Jellyfin stream
      │                                        ▲
      ▼                                        │
┌─ qBittorrent ─┐    ┌─ Radarr/Sonarr ─┐    ┌─ Jellyfin ──────┐
│  (VPN tunnel) │───→│  rename +        │───→│  (local LAN)    │
│  downloads/   │    │  organize        │    │  reads from     │
│  incomplete/  │    │  /mnt/media/     │    │  /mnt/media/    │
└───────────────┘    └─────────────────┘    └─────────────────┘
                           │
                           ▼
                    ┌─ PBS Backup ─────┐
                    │  nightly: CT 108 │
                    │  (configs + apps)│
                    │  Media NOT backed│
                    └─────────────────┘
                  (2×2TB ZFS mirror)
```

**Backup strategy:** CT 108 configs + Docker volumes backed up daily to PBS. Media on 2×1TB RAID is NOT backed up to PBS (too large) — it's maintained by the *arrs (re-downloadable) and the ZFS mirror protects against disk failure.

---

## Hardware Shopping List

| Item | Qty | Rough Cost | Notes |
|------|-----|------------|-------|
| HP ProDesk 400/600 G3-G5 (i5, 8GB+) | 2 | ~$100-150 each | Check eBay/local surplus |
| 1TB SATA HDD (for PVE2 media) | 2 | ~$25-40 each | NAS-rated preferred (WD Red, Seagate IronWolf) |
| 2TB SATA HDD (for PBS) | 2 | ~$40-60 each | For backups |
| 256GB SSD (boot drives) | 2 | ~$20-30 each | PVE2 + PBS boot. NVMe if slot available |
| 2U vented rack shelf | 1 | ~$20-30 | Amazon basics or similar |
| 24-port patch panel | 1 | ~$20-30 | Cat6 keystone type |
| Managed switch (16-24 port) | 1 | ~$50-150 | TP-Link TL-SG1024D (~$50) or UBNT USW-Lite-16-PoE (~$200) |
| Cat6 patch cables (0.5ft-1ft) | ~24-pack | ~$15-20 | Amazon basics |
| WiFi AP (U6 Lite / EAP610) | 1 | ~$60-100 | PoE-capable for clean install |
| **Estimated total** | | **~$450-800** | Depends on switch choice & ProDesk deals |

---

## Migration Path

### Phase 1: Network Refresh (1 day)
1. Mount patch panel + switch + shelf in rack
2. Wire everything up
3. Cable manage
4. Verify all existing services still work

### Phase 2: PBS Setup (1-2 hours)
1. Install PBS on fresh HP ProDesk
2. Configure ZFS + datastore
3. Link PVE1 to PBS
4. Set up backup jobs
5. **Restore drill:** restore a small CT from PBS to verify

### Phase 3: PVE2 + Media Stack (2-3 hours)
1. Install PVE on second HP ProDesk
2. Create ZFS media pool
3. Create CT 108 with bind mount
4. Deploy Docker + compose
5. Configure Gluetun VPN
6. Deploy *arrs + Jellyfin
7. Link PVE2 to PBS
8. Back up CT 108

### Phase 4: WiFi Segmentation (1-2 hours)
1. Set up VLANs on OPNsense
2. Configure switch trunk port
3. Mount AP and configure SSIDs
4. Test isolation

---

## Open Questions

1. **Switch with PoE or without?** If you get a PoE switch, you can power the AP without a separate injector. UBNT USW-Lite-16-PoE is ~$200. Non-PoE TP-Link is ~$50.
2. **Media library on PVE2 accessible from PVE1 services?** If Jellyfin lives on PVE2, Homarr on PVE1 can still link to it — just use the IP. No NFS mount needed.
3. **Back up media or not?** I say no — *arrs + quality settings = re-downloadable. The ZFS mirror handles disk failure. Saves PBS space for what actually matters (CT configs).
4. **WiFi controller:** Ubiquiti needs a software controller. Can run in Docker on CT 108. TP-Link Omada works standalone. Preference?