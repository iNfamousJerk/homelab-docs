# Proxmox Backup Server Setup

## Hardware
- **Host:** HP Z230
- **IP:** 10.2.7.65 (PBS web UI on port 8007)
- **Root password:** `********` (stored in password manager)
- **OS:** Debian 13 Trixie

### Storage
- **256GB SK hynix SSD** (`/dev/sdc`) — OS disk (LVM: pbs-root 213.5G + swap 8G)
- **2× 2TB ZFS mirror pool** (`backup`) — backup storage
  - `sda` — ST2000NM0033 (Seagate Constellation ES.3, 2TB)
  - `sdb` — ST2000DM008 (Seagate BarraCuda, 2TB)
  - Pool: `backup` — ashift=12, compression=lz4, atime=off, recordsize=1M
  - Mount: `/backup`
  - Available: ~1.76 TiB

### PBS Datastore
- **Name:** `main`
- **Path:** `/backup/datastore`
- **Retention:** 7 daily, 4 weekly, 3 monthly
- **Verification:** on new backups

## Integration with PVE

### Storage Target
- **Name:** `pbs` (type: PBS)
- **Server:** 10.2.7.65:8007
- **Datastore:** main
- **Username:** root@pam
- **Fingerprint:** B1:8D:A7:DE:3B:94:39:1B:6F:99:CF:24:74:87:A7:B1:46:C1:AA:1B:0A:BF:9E:22:0A:E0:B9:E6:8C:BA:06:C4

### Backup Schedule
To set up automatic backups for CT 106 (monitoring stack):
1. PVE Web UI → Datacenter → Backup
2. Add → select `pbs` storage
3. Schedule: daily at 03:00
4. Selection: include CT 106
5. Mode: snapshot
6. Compression: zstd
7. Keep: 7 daily, 4 weekly, 3 monthly