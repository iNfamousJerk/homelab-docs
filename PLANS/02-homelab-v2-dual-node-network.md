# Piper Homelab v2 — Dual Node + Network Expansion

> **Status:** Draft / Proposed
> **Author:** Hermes Agent
> **Date:** 2026-05-23

---

## Overview

Phase 2 of the homelab: Add a second PVE node for the media stack, a dedicated PBS node for backups, overhaul the network to handle it all, and segment IoT/guest traffic. Current PVE1 keeps running the existing CTs untouched.

**End state:**
- **PVE1** (existing) — CTs 100–107, stays as-is
- **PVE2** (new HP ProDesk) — CT 108: Docker media stack (Radarr, Sonarr, Prowlarr, qBittorrent, Jellyfin) + 2×2TB ZFS mirror for media (repurposed from PBS)
- **PBS** (existing Z230) — Proxmox Backup Server, 2×4TB ZFS mirror for backups (new drives)
- **Router** — Zimaboard 2 (replaces Optiplex OPNsense), 2.5Gb capable
- **Ripper** — Former OPNsense Optiplex → Automatic Ripping Machine + USB Blu-ray
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
│  │  Bind mount: /mnt/media (2×2TB ZFS mirror) ←──────────┘  │
│  │                                                     │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

### Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Media storage | **ZFS mirror (2×2TB)** | Redundancy, snapshots, enough space for 1080p library |
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
| Media drives | 2×2TB SATA HDDs (ZFS mirror — repurposed from PBS) | 2TB each gives room for 1080p + some 4K |

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
┌─ PBS (Z230) ────────────────────────────────────┐
│                                                 │
│  ┌── ZFS mirror: 2×4TB (new drives) ──────────┐ │
│  │  datastore: /backup/homelab               │ │
│  │  dedup + compression: on                  │ │
│  │                                            │ │
│  │  Includes:                                 │ │
│  │  ├── CT backups from PVE1 (CTs 100-107)   │ │
│  │  ├── CT 108 (media stack configs + apps)  │ │
│  │  └── Media library (movies, shows)        │ │
│  └────────────────────────────────────────────┘ │
│                                                 │
│  IP: 10.2.7.65                                  │
│  Web UI: https://10.2.7.65:8007                 │
└─────────────────────────────────────────────────┘
```

### Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Backup target | **Proxmox Backup Server** | Native PVE integration, incremental forever, dedup |
| Backup storage | **ZFS mirror (2×4TB new drives)** | Redundancy — enough for all CT backups + full media library |
| Backup scope | **CT backups + media library** | No longer excluding media — PBS has the capacity |
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
                    │                            ├── Zimaboard 2 (2.5Gb router)
                    │                            ├── Ripper (ex-OPNsense Optiplex)
                    │                            ├── All CTs
                    │                            └── WiFi AP
                    │
                    └── Shelf rack unit(s)
                         ├── Zimaboard 2 (tiny, on shelf)
                         ├── PVE1 (existing)
                         ├── PVE2 (new ProDesk)
                         ├── PBS (new ProDesk)
                         └── Ripper Optiplex (on shelf)
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
| mDNS not working across VLANs | Set up Avahi/reflector on Zimaboard 2 or a dedicated CT for AirPlay/Chromecast bridging |

---

## Part 5: DVD/Blu-ray Ripper — Getting Rips to Jellyfin

### Goal

Repurpose the former OPNsense Optiplex into an automatic disc ripper. When a disc is inserted, it rips it, converts it, and delivers the media file to Jellyfin's library on PVE2.

### Architecture

```
USB Blu-ray drive
       │
       ▼
┌─ Ripper (ex-OPNsense Optiplex) ──────────────┐
│                                                │
│  ARM (Automatic Ripping Machine)               │
│                                                │
│  Disc inserted ──→ udev trigger ──→ MakeMKV    │
│                                         │      │
│                                         ▼      │
│                                    HandBrake   │
│                                    (optional)   │
│                                         │      │
│                                         ▼      │
│                           ┌─────────────────┐  │
│                           │ Post-rip script  │  │
│                           │                  │  │
│                           │ rsync or cp to   │  │
│                           │ NFS mount/Jelly  │  │
│                           │ fin media folder  │  │
│                           └────────┬─────────┘  │
└────────────────────────────────────┼────────────┘
                                     │
                                     ▼
                   ┌─ PVE2: /mnt/media/Movies ─┐
                   │                           │
                   │   Jellyfin scans folder    │
                   │   New movie appears        │
                   └───────────────────────────┘
```

**Key insight:** The ripper and Jellyfin are separate physical machines. The ripper writes to Jellyfin's storage over the network. There are two clean ways to do this.

### Option A: NFS Export from PVE2 (Recommended)

**How it works:**
1. PVE2 exports its media ZFS pool via NFS
2. Ripper mounts the NFS share at `/mnt/jellyfin-media/Movies`
3. ARM writes finished rips directly to that share
4. Jellyfin sees the file immediately (no manual transfer needed)

**Setup:**
```
# On PVE2 — export the media dataset via NFS
apt install nfs-kernel-server
zfs set sharenfs="rw=@10.2.7.0/24,no_subtree_check" media/media

# On Ripper — mount it permanently
echo "10.2.7.x:/media/media /mnt/jellyfin-media nfs defaults 0 0" >> /etc/fstab
mount -a
```

**Pros:** Real-time, automated, no double-storage. Rip writes directly to Jellyfin.
**Cons:** NFS adds a network dependency. If PVE2 is down, rips fail.

### Option B: Rsync Post-Rip Script

**How it works:**
1. Ripper rips to local storage first (e.g., `/mnt/ripper-staging/`)
2. ARM calls a post-rip script that `rsync`s the file to PVE2's media folder
3. Script verifies checksum before deleting local copy

```bash
#!/bin/bash
# /usr/local/bin/arm-postrip.sh — runs after ARM finishes
DEST="10.2.7.x:/mnt/media/Movies"
rsync -av --remove-source-files "$1" "$DEST/"
```

**Pros:** Works even if PVE2 is temporarily offline (rip stays local). No NFS config needed.
**Cons:** Needs enough local storage for staging. Extra step.

### Key Decisions — Ripper

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Ripping software | **ARM (Automatic Ripping Machine)** | Detects disc, auto-rips, runs post-rip actions. Proven. |
| Disc drive | **USB Blu-ray (LG WH16NS60 or similar)** | Reads both DVDs and Blu-rays. UHD-friendly. |
| Video codec | **MakeMKV (raw) + optional HandBrake (compressed)** | MakeMKV dumps perfect copy. HandBrake shrinks for storage. |
| Delivery method | **NFS mount (Option A)** | Real-time, no staging storage needed, simpler automation |
| Ripper OS | **Ubuntu Server / Debian** | ARM packages are Linux-native. Lightweight. |

### Implementation Plan

1. Wipe the former OPNsense Optiplex, install Debian/Ubuntu Server
2. Install ARM: `apt install arm` or via docker
3. Configure ARM to watch the Blu-ray drive for disc insertion
4. (For NFS) Export `/mnt/media/Movies` from PVE2, mount on ripper
5. Configure ARM post-rip to write to PVE2's Movies folder
6. Test: insert a DVD, verify movie appears in Jellyfin

### Pitfalls

| Issue | Fix |
|-------|-----|
| ARM doesn't detect disc insertion | Add udev rule: `SUBSYSTEM=="block", KERNEL=="sr[0-9]*", ACTION=="change", RUN+="/usr/bin/arm"` |
| MakeMKV needs libaacs keys | Copy `KEYDB.cfg` to `/home/arm/.MakeMKV/` for encrypted Blu-rays |
| NFS permission issues | Ensure ARM user ID matches jellyfin user ID on PVE2, or use `no_root_squash` on NFS export |
| Large rip takes hours | ARM can send notification on completion. Or just check Jellyfin next day. |
| Blu-ray menu structure complex | ARM's MakeMKV integration handles most discs. Problem discs: use MakeMKV GUI manually. |

---

```
Torrent download / Disc rip                      Jellyfin stream
      │        │                                        ▲
      ▼        ▼                                        │
┌─ qBittorrent ─┐    ┌─────────────┐    ┌─ Ripper ───────┐
│  (VPN tunnel) │    │  Radarr/    │    │  ARM + MakeMKV │
│  downloads/   │    │  Sonarr    │    │  USB Blu-ray   │
│  incomplete/  │    │  organize  │    │       │        │
└───────┬───────┘    └─────┬───────┘    └───────┼────────┘
        │                  │                    │ (NFS mount)
        ▼                  ▼                    ▼
┌──────────────────────────────────────────────────────┐
│              /mnt/media/ (2×1TB ZFS mirror)          │
│         Movies/  TV Shows/  Music/  etc.            │
└────────────────────┬─────────────────────────────────┘
                     │
                     ▼
              ┌─ Jellyfin ──────┐
              │  (local LAN)    │
              │  reads all media│
              │  no VPN needed  │
              └─────────────────┘
                           │
                           ▼
                    ┌─ PBS Backup ───────────┐
                    │  Nightly (CT 108 +     │
                    │  media library)         │
                    │  ALL backed up to       │
                    │  2×4TB ZFS mirror      │
                    └────────────────────────┘
                  (Full data protection)
```

**Backup strategy:** Full data protection — CT 108 configs, Docker volumes, apps, AND the media library are all backed up nightly to PBS via the 2×4TB ZFS mirror. The ZFS mirror on PVE2 (2×2TB) protects against single disk failure on the live storage. PBS gives you a point-in-time recovery for everything including movies and shows.

---

## Hardware Shopping List

| Item | Qty | Rough Cost | Notes |
|------|-----|------------|-------|
| **Zimaboard 2** (2.5Gb router) | 1 | ~$200 | Replaces OPNsense Optiplex. 2.5Gb NICs. |
| HP ProDesk 400/600 G3-G5 (i5, 8GB+) | 1 | ~$100-150 | PVE2 only. PBS already on Z230. Check eBay/local surplus. |
| USB Blu-ray drive (LG WH16NS60) | 1 | ~$60-80 | For ripper. Reads DVD + Blu-ray + UHD. |
| **4TB CMR NAS HDD** (for PBS) | **2** | **~$80-120 each** | **New — replace the 2×2TB in PBS. WD Red Plus or Seagate IronWolf.** |
| 256GB SSD (boot drives) | 1 | ~$20-30 | PVE2 boot. NVMe if slot available |
| 2U vented rack shelf | 1 | ~$20-30 | Amazon basics or similar |
| 24-port patch panel | 1 | ~$20-30 | Cat6 keystone type |
| Managed switch (16-24 port) | 1 | ~$50-200 | Non-PoE ~$50 (TL-SG1024D). PoE ~$200 (UBNT USW-Lite-16-PoE) |
| Cat6 patch cables (0.5ft-1ft) | ~24-pack | ~$15-20 | Amazon basics |
| WiFi AP (U6 Lite / EAP610) | 1 | ~$60-100 | PoE-capable for clean install |
| **Estimated total** | | **~$700-1,050** | Biggest cost: 2×4TB NAS drives (~$200) |

---

## Migration Path

### Phase 1: Zimaboard 2 Router Swap (1-2 hours)
1. Install OPNsense on Zimaboard 2 (boot from eMMC or SSD)
2. Migrate config from Optiplex OPNsense → Zimaboard (export/import XML)
3. Swap cables: WAN from Optiplex → Zimaboard, LAN from Optiplex → Zimaboard
4. Verify all services still reachable
5. Wipe the Optiplex — it's now the ripper

### Phase 2: Network Refresh (1 day)
1. Mount patch panel + switch + shelf in rack
2. Wire everything up
3. Cable manage
4. Verify all existing services still work

### Phase 2: PBS Upgrade (1-2 hours)
1. **Purchase 2×4TB NAS drives** (WD Red Plus or Seagate IronWolf)
2. Install drives in Z230 (PBS machine)
3. Create new ZFS pool: `zpool create backup mirror /dev/sda /dev/sdb`
4. Recreate PBS datastore: `proxmox-backup-manager datastore create main /backup/datastore`
5. Re-link PVE1 to PBS (re-auth with new fingerprint)
6. Restore backups from old 2×2TB pool (optional, if data existed)
7. Set up backup jobs for all CTs + media mount
8. **Restore drill:** restore a small CT from PBS to verify

### Phase 3: PVE2 + Media Stack (2-3 hours)
1. Install PVE on the HP ProDesk
2. Move the **2×2TB drives from PBS** into PVE2
3. Create ZFS media pool: `zpool create media mirror /dev/sda /dev/sdb`
4. Create CT 108 with bind mount to /mnt/media
5. Deploy Docker + compose
6. Configure Gluetun VPN
7. Deploy *arrs + Jellyfin
8. Link PVE2 to PBS
9. Back up CT 108 to PBS

### Phase 5: Ripper Setup (1-2 hours)
1. Install Debian/Ubuntu on former OPNsense Optiplex
2. Install ARM + MakeMKV + HandBrake
3. Export NFS from PVE2 /mnt/media
4. Mount NFS on ripper
5. Test: rip a disc, verify in Jellyfin

### Phase 6: WiFi Segmentation (1-2 hours)
1. Set up VLANs on Zimaboard 2 (OPNsense)
2. Configure switch trunk port
3. Mount AP and configure SSIDs
4. Test isolation

---

## Open Questions

1. **Switch with PoE or without?** If you get a PoE switch, you can power the AP without a separate injector. UBNT USW-Lite-16-PoE is ~$200. Non-PoE TP-Link is ~$50.
2. **Media library on PVE2 accessible from PVE1 services?** If Jellyfin lives on PVE2, Homarr on PVE1 can still link to it — just use the IP. No NFS mount needed.
3. **Back up media or not?** I say no — *arrs + quality settings = re-downloadable. The ZFS mirror handles disk failure. Saves PBS space for what actually matters (CT configs).
4. **WiFi controller:** Ubiquiti needs a software controller. Can run in Docker on CT 108. TP-Link Omada works standalone. Preference?