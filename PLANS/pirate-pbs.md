# 🏴‍☠️ Project Pirate PBS

## Mission
Build a dedicated second node running Proxmox Backup Server for off-host backups of the entire homelab.

## Why "Pirate PBS"
- **Pirate** = off the main ship (PVE host), sailing independently
- **PBS** = Proxmox Backup Server
- If the main PVE host sinks, Pirate PBS has all the loot (backups) safe

---

## 🎯 Goals

1. **Off-host backups** — containers and PVE host backed up to a separate machine
2. **ZFS protection** — bit rot detection, compression, snapshots
3. **3TB usable storage** — 1TB for daily backups, 2TB for long-term archives
4. **Hands-on build** — learn ZFS, PBS config, hardware assembly

---

## 🛠️ Hardware

| Part | Item | Cost | Status |
|------|------|------|--------|
| **Base machine** | Dell Optiplex 7050 MT (i7-7700, 16GB, 512GB SSD) | ~$200 | 🔍 Sourced |
| **Drive adapters** | 2x 3.5"→5.25" adapter trays | ~$16 | 📋 To buy |
| **Blu-ray drive** | External USB Blu-ray (for later ARM project) | ~$50 | 📋 Future |
| **HDD 1** | 1TB SATA (already owned) | $0 | ✅ Have it |
| **HDD 2** | 1TB SATA (already owned) | $0 | ✅ Have it |
| **HDD 3** | 2TB SATA (already owned) | $0 | ✅ Have it |
| **HDD 4** | 2TB SATA (already owned) | $0 | ✅ Have it |
| **NVMe** | 512GB NVMe/SSD (comes with Optiplex) | $0 | ✅ Included |
| **Total** | | **~$216** | |

---

## 💾 Drive Layout

```
Dell Optiplex 7050 MT
┌──────────────────────────────────────────┐
│                                            │
│  5.25" Bay 1: [2TB HDD in adapter]  ← ZFS mirror pair
│  5.25" Bay 2: [2TB HDD in adapter]  ← "backup-archive" pool
│                                            │
│  HDD Cage 1: [1TB HDD]               ← ZFS mirror pair
│  HDD Cage 2: [1TB HDD]               ← "backup-fast" pool
│                                            │
│  M.2 Slot: [512GB SSD]               ← Debian + PBS OS
│                                            │
│  SATA Ports: 4 used (all 4 HDDs)          │
│  Power: Stock PSU (enough for 4 HDDs)     │
└──────────────────────────────────────────┘
```

### Why Two ZFS Pools?

| Pool | Drives | Usable | Purpose |
|------|--------|--------|---------|
| `backup-fast` | 2×1TB mirror | **1TB** | Daily CT backups (7-day rotation) |
| `backup-archive` | 2×2TB mirror | **2TB** | Long-term retention, media backups |

Two pools means:
- Daily backups don't compete with archive storage for space
- Different retention policies per pool
- If one pool fails, the other still has your data

---

## 📦 Software Stack

```
Debian 12 (minimal install)
└── Proxmox Backup Server (installed via .deb repo)
    ├── backup-fast (ZFS pool)  →  Daily CT snapshots
    └── backup-archive (ZFS)    →  Long-term storage
```

---

## 🔗 Network

Pirate PBS sits on the same subnet as everything else:

```
Hostname:  pbs (or pirate-pbs)
IP:        10.2.7.xxx (next available)
Subnet:    10.2.7.0/24
Gateway:   10.2.7.1 (OPNsense)
DNS:       10.2.7.2 (Pi-hole)
Access:    Web UI + SSH (key-based auth from Hermes)
```

---

## 🧭 Build Steps (to be followed when hardware arrives)

```
Phase 1: Hardware Setup
  □ Unbox Optiplex, verify it boots
  □ Install 2×1TB HDDs in the drive cage
  □ Install 2×2TB HDDs in 5.25" bays with adapter trays
  □ Close case, plug in power + Ethernet

Phase 2: Install PBS
  □ Download Proxmox Backup Server ISO
  □ Flash to USB (dd or Rufus)
  □ Boot from USB, install to the 512GB SSD
  □ Configure: hostname=pbs, IP=10.2.7.xxx

Phase 3: Configure Storage
  □ Create ZFS pool: zpool create backup-fast mirror /dev/sda /dev/sdb
  □ Create ZFS pool: zpool create backup-archive mirror /dev/sdc /dev/sdd
  □ Enable compression: zfs set compression=zstd backup-fast
  □ Create PBS datastores via web UI or CLI

Phase 4: Connect PVE → PBS
  □ In PVE web UI: Datacenter → Storage → Add → PBS
  □ Point to Pirate PBS IP and datastores
  □ Set backup schedules (daily at 3 AM)

Phase 5: Verify & Document
  □ Run a manual backup of one CT
  □ Restore to a test location to verify integrity
  □ Add to homelab-docs / homelab-architecture
  □ Celebrate ☠️
```

---

## 📊 Backup Schedule (Planned)

| What | Schedule | Target | Retention |
|------|----------|--------|-----------|
| All CTs (100-107) | Daily @ 3 AM | `backup-fast` | Keep last 7 |
| PVE host config | Daily @ 3 AM | `backup-fast` | Keep last 7 |
| Weekly full snapshot | Every Sunday | `backup-archive` | Keep last 4 |
| Monthly archive | 1st of month | `backup-archive` | Keep last 3 |

---

## ☠️ The Pirate Code (Rules)

1. **The PBS never runs on the PVE host** — that's not a backup, that's a copy
2. **Test a restore at least once** — a backup you can't restore isn't a backup
3. **Keep backups boring** — PBS just sits there and does its job, no tinkering
4. **Document everything** — when the PVE host dies, the docs should get you back up
5. **Monitor PBS disk usage** — add to Grafana alerts so you know before it fills up

---

## 🔮 Future Expansion

After Pirate PBS is running, the Optiplex has room to grow:
- Run Hermes Jr. as a VM/container on it (16GB RAM + i7 = plenty of headroom)
- Add the USB Blu-ray for ARM media ripping
- Use as a general lab machine for experiments