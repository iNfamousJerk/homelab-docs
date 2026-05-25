# PVE & PBS Lifecycle Management Playbook

> **Definitive operational runbook** — Create, Restore, Maintain, and Support every VM, LXC, and service on Proxmox VE (PVE) and Proxmox Backup Server (PBS).
>
> **Environment:** Single-node PVE host (`pve`) + PBS (`z230`) on 10.2.7.0/24
> **PVE Version:** 9.1.1 | **PBS Version:** 3.x (Debian 13 Trixie)

---

## Table of Contents

- [PILLAR 1: INITIAL CREATION & DEPLOYMENT STRATEGY](#pillar-1-initial-creation--deployment-strategy)
  - [1.1 LXC Container Provisioning](#11-lxc-container-provisioning)
  - [1.2 QEMU VM Provisioning](#12-qemu-vm-provisioning)
  - [1.3 Storage Topologies & Profiles](#13-storage-topologies--profiles)
  - [1.4 Networking & VLAN Architecture](#14-networking--vlan-architecture)
  - [1.5 Baseline Templates for Common Workloads](#15-baseline-templates-for-common-workloads)
- [PILLAR 2: ENTERPRISE DISASTER RECOVERY & RESTORATION](#pillar-2-enterprise-disaster-recovery--restoration)
  - [2.1 PBS Integration & Encryption](#21-pbs-integration--encryption)
  - [2.2 Restoration Workflows](#22-restoration-workflows)
  - [2.3 Emergency Recovery Scenarios](#23-emergency-recovery-scenarios)
- [PILLAR 3: SYSTEM MAINTENANCE & AUTOMATION](#pillar-3-system-maintenance--automation)
  - [3.1 Patch Management](#31-patch-management)
  - [3.2 Storage Optimization](#32-storage-optimization)
  - [3.3 Metrics & Monitoring](#33-metrics--monitoring)
- [PILLAR 4: LIFECYCLE SUPPORT & TROUBLESHOOTING](#pillar-4-lifecycle-support--troubleshooting)
  - [4.1 Triage Diagnostics](#41-triage-diagnostics)
  - [4.2 Common Failure Modes & Fixes](#42-common-failure-modes--fixes)
  - [4.3 Log Analytics](#43-log-analytics)
- [Quick-Reference Command Index](#quick-reference-command-index)

---

## PILLAR 1: INITIAL CREATION & DEPLOYMENT STRATEGY

### 1.1 LXC Container Provisioning

#### Bare Minimum (for reference)

```bash
pct create <CT_ID> local:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst \
  --hostname <name> \
  --storage local-lvm \
  --rootfs local-lvm:4 \
  --cores 1 \
  --memory 512 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --unprivileged 1 \
  --ostype debian
```

#### Production Template (with all recommended flags)

```bash
pct create <CT_ID> local:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst \
  --hostname <name> \
  --storage <storage_pool> \            # e.g., local-zfs, local-lvm, pbs-nfs
  --rootfs <storage_pool>:<size> \      # e.g., local-zfs:16
  --cores <N> \
  --memory <MB> \
  --swap 512 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp,firewall=1 \
  --onboot 1 \
  --startup order=<N>,up=<delay> \
  --features keyctl=1,nesting=1,fuse=1 \
  --unprivileged 1 \
  --ostype debian \
  --tags <comma-separated-tags> \
  --nameserver 10.2.7.2 \
  --searchdomain local.lan
```

#### ⚠️ Critical Flags Explained

| Flag | Value | Why |
|------|-------|-----|
| `--features keyctl=1` | Required for Docker in unprivileged LXC | Enables kernel keyring — Docker overlay2 needs this |
| `--features nesting=1` | Required for Docker in LXC | Allows container to create its own namespaces |
| `--features fuse=1` | Only if using FUSE mounts | SSHFS, rclone mount, mergerFS inside container |
| `--unprivileged 1` | Default and preferred | Better security — UID mapping prevents host escape |
| `--onboot 1` | Auto-start after host reboot | Essential for infrastructure containers |
| `--startup order=N,up=M` | Staggered boot sequence | DNS containers first (order=1), then services (order=2+) |
| `--nameserver 10.2.7.2` | Point to Pi-hole directly | PVE-managed resolv.conf defaults to host IP which breaks DNS |
| `--swap 512` | Minimal swap | Prevents OOM for small containers without wasting disk |

#### Post-Creation Hardening

```bash
# 1. Fix DNS immediately (PVE writes wrong nameserver)
pct exec <CT_ID> -- bash -c 'echo "nameserver 10.2.7.2" > /etc/resolv.conf'

# 2. Set root password (API --password flag often doesn't work)
pct enter <CT_ID> passwd

# 3. Deploy SSH key
pct exec <CT_ID> -- bash -c 'mkdir -p ~/.ssh && chmod 700 ~/.ssh && echo "<PUBKEY>" >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys'

# 4. Remove stale apt sources (Proxmox templates include enterprise repos)
pct exec <CT_ID> -- rm -f /etc/apt/sources.list.d/pve-enterprise.list

# 5. Update immediately
pct exec <CT_ID> -- bash -c 'apt update && apt upgrade -y'

# 6. Install common utilities
pct exec <CT_ID> -- apt install -y curl wget gnupg sudo
```

#### Resizing Existing Containers

```bash
# Grow the rootfs (instant, no reboot needed for LXC)
pct resize <CT_ID> rootfs <new_total_size>G

# Add a mount point (if you passed through a whole disk via hardware passthrough)
pct set <CT_ID> --mp0 /dev/disk/by-id/<ata-...>,mp=/data

# Add disk from storage pool
pct set <CT_ID> --mp0 <storage>:<size>,mp=/data

# Change CPU/RAM (hot-pluggable for containers)
pct set <CT_ID> --cores 4 --memory 4096
```

### 1.2 QEMU VM Provisioning

#### Minimal QEMU VM

```bash
qm create <VM_ID> \
  --name <name> \
  --memory 2048 \
  --cores 2 \
  --net0 virtio,bridge=vmbr0,firewall=1 \
  --scsihw virtio-scsi-single \
  --boot order=scsi0 \
  --ostype l26 \
  --agent 1 \
  --onboot 1
```

#### VM with Full Performance Optimization

```bash
qm create <VM_ID> \
  --name <name> \
  --machine q35 \
  --cpu host \
  --sockets 1 \
  --cores <N> \
  --memory <MB> \
  --balloon 0 \
  --scsihw virtio-scsi-single \
  --net0 virtio,bridge=vmbr0,model=virtio,firewall=1 \
  --boot order=scsi0 \
  --ostype l26 \
  --agent 1 \
  --onboot 1 \
  --startup order=<N>,up=<delay> \
  --kvm 1 \
  --numa 1 \
  --vga qxl \
  --tablet 1

# Then attach storage
qm set <VM_ID> --scsi0 <storage>:<size>,iothread=1,discard=on,ssd=1

# Attach ISO for OS installation
qm set <VM_ID> --ide2 local:iso/<installer>.iso,media=cdrom

# Set boot order
qm set <VM_ID> --boot order=ide2;scsi0
```

#### ⚠️ QEMU Performance Flags Explained

| Flag | Value | Why |
|------|-------|-----|
| `--machine q35` | Modern chipset | Better PCIe topology, NVMe support, UEFI compatibility |
| `--cpu host` | Pass through host CPU features | Exposes AES-NI, AVX2, etc. for database/ML workloads |
| `--balloon 0` | Disable ballooning | Balloon driver can cause guest memory fragmentation. Only enable if overcommitting aggressively. Use `1` for overprovisioned hosts. |
| `--scsihw virtio-scsi-single` | One vCPU queue per disk | Critical for IOPS-heavy workloads. Avoid `virtio-scsi` (single queue) or `lsi` (emulated, slow). |
| `iothread=1` | Dedicated I/O thread per disk | Prevents disk I/O from blocking vCPU threads |
| `discard=on` | TRIM pass-through | Frees thin-provisioned space on the host when guest deletes files |
| `ssd=1` | Inform guest it's on SSD | Enables guest-side SSD optimizations (TRIM, noop scheduler) |
| `--agent 1` | QEMU Guest Agent | Enables graceful shutdown, IP reporting, filesystem freeze for snapshots |
| `--numa 1` | NUMA awareness | Essential for VMs with >8 vCPUs or >32GB RAM |

#### Importing Existing Disk Images

```bash
# VirtIO disk images
qm disk import <VM_ID> /path/to/disk.qcow2 <storage> --format qcow2

# Raw images
qm disk import <VM_ID> /path/to/disk.raw <storage> --format raw

# VMDK from VMware
qemu-img convert -f vmdk disk.vmdk -O qcow2 disk.qcow2
qm disk import <VM_ID> disk.qcow2 <storage> --format qcow2
```

### 1.3 Storage Topologies & Profiles

#### Storage Profile Decision Matrix

| Profile | Use Case | Pros | Cons | Best For |
|---------|----------|------|------|----------|
| **local-lvm** | Thin LVM | Fast, easy resize, snapshots | No checksumming, no compression | Stateless VMs, temp containers |
| **local-zfs** | ZFS native | Checksums, compression (lz4/zstd), snapshots, clones | RAM hungry (ARC), complex tuning | Production containers, databases |
| **NFS/CIFS** | NAS-backed | Shared storage, central mgmt | Network bottleneck, no snapshots | Media storage, file shares |
| **PBS** | Backup target | Dedup, encryption, verification | Not for live workloads | Only backups |
| **Directory** | Simple | No overhead, any filesystem | No snapshots, no thin provisioning | ISOs, templates, CT root images |

#### ZFS Pool Creation for Different Workloads

```bash
# Performance pool (mirror — good for VM/CT storage)
zpool create -o ashift=12 \
  -O compression=lz4 \
  -O atime=off \
  -O recordsize=128k \
  -O xattr=sa \
  -O mountpoint=/<pool_name> \
  <pool_name> mirror /dev/disk/by-id/<ata-...> /dev/disk/by-id/<ata-...>

# Backup pool (mirror — large recordsize for sequential writes)
zpool create -o ashift=12 \
  -O compression=zstd-3 \
  -O atime=off \
  -O recordsize=1M \
  -O xattr=sa \
  -O mountpoint=/<pool_name> \
  <pool_name> mirror /dev/disk/by-id/<ata-...> /dev/disk/by-id/<ata-...>

# Capacity pool (RAIDZ1 — 3+ disks, no redundancy for single-disk failure on large disks)
# Not recommended for homelab. Use mirrors instead.
```

#### Adding Disks to an Existing ZFS Pool

```bash
# Attach a disk to a mirror (grows a 2-way to a 3-way mirror)
zpool attach <pool_name> /dev/disk/by-id/<existing-disk> /dev/disk/by-id/<new-disk>

# Add a second mirror vdev to the pool (stripe across 2 mirrors — +capacity, +redundancy)
zpool add <pool_name> mirror /dev/disk/by-id/<disk1> /dev/disk/by-id/<disk2>

# Replace a failed disk
zpool replace <pool_name> /dev/disk/by-id/<failed-disk> /dev/disk/by-id/<new-disk>
```

#### LVM Thin Provisioning (local-lvm alternative)

```bash
# Create thin pool
pvcreate /dev/sdX
vgcreate vg-pve /dev/sdX
lvcreate -L <total_size>G -T vg-pve/thinpool

# Register in Proxmox
# Web UI → Datacenter → Storage → Add → LVM-Thin
# Or manually in /etc/pve/storage.cfg
```

### 1.4 Networking & VLAN Architecture

#### Bridge Configuration

PVE creates `vmbr0` during installation, bridged to the physical NIC. Additional bridges enable VLAN-aware networking:

```bash
# /etc/network/interfaces on PVE host
auto lo
iface lo inet loopback

auto eno1
iface eno1 inet manual

# Main bridge (management VLAN)
auto vmbr0
iface vmbr0 inet static
    address 10.2.7.64/24
    gateway 10.2.7.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094                    # Allow all VLANs through

# OPTIONAL: OPNsense WAN bridge (passthrough physical NIC to VM)
auto vmbr1
iface vmbr1 inet manual
    bridge-ports eno2                    # Second physical NIC connected to modem
    bridge-stp off
    bridge-fd 0
```

#### VLAN Tagging per Container/VM

```bash
# LXC: tag VLAN 10 on eth0
pct set <CT_ID> --net0 name=eth0,bridge=vmbr0,tag=10,ip=dhcp,firewall=1

# QEMU: tag VLAN 20 on virtio NIC
qm set <VM_ID> --net0 virtio,bridge=vmbr0,tag=20,firewall=1

# VLAN 1 (default) = management traffic
# VLAN 10 = infrastructure (DNS, monitoring, management)
# VLAN 20 = media (Jellyfin, Sonarr, Radarr — firewall restricted to port 8096)
# VLAN 30 = guest/IoT (internet only, no LAN access)
```

#### OPNsense Integration

OPNsense sits on `vmbr1` (WAN) and `vmbr0` (LAN) as a VM. The Proxmox host is a LAN client:

```bash
# OPNsense VM creation
qm create 100 \
  --name opnsense \
  --memory 2048 \
  --cores 2 \
  --cpu host \
  --machine q35 \
  --net0 virtio,bridge=vmbr0,firewall=0 \
  --net1 virtio,bridge=vmbr1,firewall=0 \
  --scsihw virtio-scsi-single \
  --boot order=scsi0 \
  --ostype l26 \
  --agent 1 \
  --onboot 1 \
  --startup order=1,up=10

qm set 100 --scsi0 local-zfs:16,iothread=1,discard=on,ssd=1
qm set 100 --ide2 local:iso/OPNsense-<ver>-dvd.iso,media=cdrom
```

#### Static IP vs DHCP

```bash
# Static IP (preferred for infrastructure):
pct set <CT_ID> --net0 name=eth0,bridge=vmbr0,ip=10.2.7.X/24,gw=10.2.7.1

# DHCP (OPNsense is the DHCP server for VLANs):
pct set <CT_ID> --net0 name=eth0,bridge=vmbr0,tag=<VLAN>,ip=dhcp
```

### 1.5 Baseline Templates for Common Workloads

#### Media Automation Stack (CT 108 — proposed)

```bash
CT_ID=108
SIZE=32               # GB
CORES=4
MEMORY=4096

pct create $CT_ID local:vztmpl/ubuntu-24.04-standard_24.04-2_amd64.tar.zst \
  --hostname media-stack \
  --storage local-zfs \
  --rootfs local-zfs:${SIZE} \
  --cores $CORES \
  --memory $MEMORY \
  --swap 1024 \
  --net0 name=eth0,bridge=vmbr0,tag=20,ip=10.2.7.108/24,gw=10.2.7.1 \
  --onboot 1 \
  --startup order=2,up=60 \
  --features keyctl=1,nesting=1 \
  --unprivileged 1 \
  --ostype ubuntu \
  --tags media,sabnzbd,sonarr,radarr,prowlarr,qbit \
  --nameserver 10.2.7.2 \
  --searchdomain local.lan
```

**Post-provision for Docker-based media stack:**

```bash
pct exec $CT_ID -- bash -c '
  # Fix DNS
  echo "nameserver 10.2.7.2" > /etc/resolv.conf

  # Install Docker
  curl -fsSL https://get.docker.com | sh
  usermod -aG docker root

  # Compose plugin
  apt-get install -y docker-compose-plugin

  # Create stack directory
  mkdir -p /opt/media-stack
'
```

#### AI / Local LLM Container (CT 100 — Hermes Agent)

```bash
CT_ID=100
SIZE=20
CORES=2
MEMORY=2048
# For Ollama: need 8GB+ RAM and nesting for Docker-in-LXC
# For inference-only (Hermes Agent): 2GB is sufficient

pct create $CT_ID local:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst \
  --hostname hermes \
  --storage local-zfs \
  --rootfs local-zfs:${SIZE} \
  --cores $CORES \
  --memory $MEMORY \
  --swap 512 \
  --net0 name=eth0,bridge=vmbr0,ip=10.2.7.107/24,gw=10.2.7.1 \
  --onboot 1 \
  --startup order=2,up=30 \
  --features keyctl=1,nesting=1,fuse=1 \
  --unprivileged 1 \
  --ostype debian \
  --tags ai,agent,automation \
  --nameserver 10.2.7.2 \
  --searchdomain local.lan
```

#### Core DNS / Pi-hole (CT 107 — must be first boot)

```bash
CT_ID=107
SIZE=8
CORES=1
MEMORY=512

pct create $CT_ID local:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst \
  --hostname pihole \
  --storage local-zfs \
  --rootfs local-zfs:${SIZE} \
  --cores $CORES \
  --memory $MEMORY \
  --swap 256 \
  --net0 name=eth0,bridge=vmbr0,ip=10.2.7.2/24,gw=10.2.7.1 \
  --onboot 1 \
  --startup order=1,up=10 \
  --unprivileged 1 \
  --ostype debian \
  --tags dns,dhcp,pihole \
  --nameserver 1.1.1.1 \
  --searchdomain local.lan
```

**Note:** Pi-hole needs upstream DNS during install. Set `--nameserver 1.1.1.1` during creation, then switch to `127.0.0.1` after Pi-hole is configured.

#### Docker Monitoring Stack (CT 106 — Grafana, Prometheus, etc.)

```bash
CT_ID=106
SIZE=32
CORES=4
MEMORY=6144

pct create $CT_ID local:vz.../ubuntu-24.04-standard_24.04-2_amd64.tar.zst \
  --hostname docker-monitoring \
  --storage local-zfs \
  --rootfs local-zfs:${SIZE} \
  --cores $CORES \
  --memory $MEMORY \
  --swap 1024 \
  --net0 name=eth0,bridge=vmbr0,ip=10.2.7.108/24,gw=10.2.7.1 \
  --onboot 1 \
  --startup order=2,up=30 \
  --features keyctl=1,nesting=1 \
  --unprivileged 1 \
  --ostype ubuntu \
  --tags monitoring,docker,grafana,prometheus \
  --nameserver 10.2.7.2 \
  --searchdomain local.lan
```

---

## PILLAR 2: ENTERPRISE DISASTER RECOVERY & RESTORATION

### 2.1 PBS Integration & Encryption

#### PBS User Management

```bash
# Create a dedicated backup user (instead of using root@pam)
proxmox-backup-manager user create backup-user@pbs \
  --password '<strong-password>' \
  --comment "Automated backup user for PVE"

# Create API token for the user (for unattended backups)
proxmox-backup-manager user create-token backup-user@pbs backup-token

# Set ACL — allow backup/restore on the datastore
proxmox-backup-manager acl update /datastore/main/ \
  --auth-id backup-user@pbs \
  --perm Datastore.Backup,Datastore.Restore

# Set retention
proxmox-backup-manager datastore set main \
  --keep-daily 7 \
  --keep-weekly 4 \
  --keep-monthly 3 \
  --keep-yearly 0
```

#### Encryption Domain Setup

```bash
# Create encryption key
openssl enc -aes-256-cbc -k '<encryption-passphrase>' -P -md sha256 | head -1
# Output: salt=... key=HEXKEY

# Or use PBS's key manager
proxmox-backup-manager key create master.key \
  --cipher aes-256-gcm \
  --kdf none

# Register on PVE side (Web UI or via API)
# Datacenter → Storage → Add → Proxmox Backup Server
# Check "Encryption" → Upload or paste key

# Verify encryption is active
proxmox-backup-manager datastore list-backups main | grep encrypted
```

**⚠️ Encryption pitfall:** If you lose the encryption key/passphrase, your backups are **permanently unrecoverable**. Store a paper copy and an off-site copy.

#### Registering PBS on PVE

```bash
# Via Web UI:
# Datacenter → Storage → Add → Proxmox Backup Server
#   ID: pbs
#   Server: 10.2.7.65
#   Username: backup-user@pbs
#   Password: <token-or-password>
#   Datastore: main
#   Fingerprint: <from PBS dashboard>
#   (Optional) Encryption Key: <paste key>
```

**To get the PBS fingerprint:**

```bash
# From PBS host:
proxmox-backup-manager cert info | grep Fingerprint
# Or from CLI:
openssl x509 -in /etc/proxmox-backup/proxy.pem -fingerprint -sha256 -noout
```

#### Automating Backups via CLI (instead of Web UI)

```bash
# Create a backup job for a specific container
vzdump 106 \
  --storage pbs \
  --mode snapshot \
  --compress zstd \
  --remove 0 \
  --notes "CT 106 - Monitoring stack weekly backup" \
  --all 0

# Create a backup that runs on schedule
# Web UI: Datacenter → Backup → Add
# Or use PBS's built-in sync jobs:
proxmox-backup-manager sync-job create \
  --owner root@pam \
  --store main \
  --remote <remote-pbs> \
  --remote-store main \
  --rate 100 \
  --schedule "daily"

# Or on PVE, use cron:
# /etc/cron.d/pve-backups
# 0 3 * * * root vzdump --all --mode snapshot --compress zstd --storage pbs --exclude-path /mnt/
```

### 2.2 Restoration Workflows

#### Full Container Restore from PBS

```bash
# List available backups
pbs backup list main
# Or via vzdump index
vzdump --list --storage pbs

# Restore entire CT
pct restore <CT_ID> <backup-volume> \
  --storage local-zfs \
  --hostname <name> \
  --unprivileged 0 \
  --ignore-unpack-errors 0

# Alternative: restore to a NEW container ID (avoids conflict with existing)
pct restore <NEW_CT_ID> <backup-volume> --storage local-zfs

# Start it
pct start <CT_ID>
```

#### Full VM Restore from PBS

```bash
# List VM backups
pbs backup list main host/pve

# Restore
qmrestore <backup-file> <VM_ID> --storage local-zfs

# Alternative with live mapping
qmrestore pbs:backup/<snapshot> <VM_ID> \
  --storage local-zfs \
  --skip-template
```

#### Single-File Extraction from LXC Backups

This is critical — extracting a single file (e.g., a broken config) without restoring the whole container:

```bash
# Method 1: Mount PBS backup via FUSE
# Install pbs-client tools
apt install proxmox-backup-client

# Mount the backup
proxmox-backup-client mount \
  --keyfile /path/to/encryption-key \
  main:ct/<CT_ID>/<backup-id> \
  /mnt/restore

# Navigate through the filesystem
ls /mnt/restorerm -full-path/
cp /mnt/restore/path/to/config.file /tmp/
cp /mnt/restore/var/lib/docker/volumes/.../_data/config.yaml /tmp/

# Unmount when done
proxmox-backup-client unmount /mnt/restore

# ---
# Method 2: Extract from local vzdump archive
# Find the backup file:
ls /var/lib/vz/dump/ | grep <CT_ID>

# Extract specific file
tar -tzf vzdump-lxc-<CT_ID>-*.tar.zst | grep config.yaml
tar -xzf vzdump-lxc-<CT_ID>-*.tar.zst \
  -C /tmp/restore \
  ./etc/config.yaml

# ---
# Method 3: PBS web UI (for non-technical users)
# Web UI → Datastore → Backup → Click backup → "File-Level Restore"
# Downloads a pxar archive you can browse
```

#### Restoring a Single Docker Volume from PBS

```bash
# 1. Mount the CT backup
proxmox-backup-client mount main:ct/<CT_ID>/<backup-id> /mnt/restore

# 2. Find the Docker volume data
# Docker volumes in LXC are at /var/lib/docker/volumes/<name>/_data/
ls /mnt/restore/var/lib/docker/volumes/

# 3. Copy the specific volume
cp -a /mnt/restore/var/lib/docker/volumes/grafana_data/_data/ /tmp/restored-grafana/

# 4. Stop the container, restore
systemctl stop docker  # or docker-compose down
cp -a /tmp/restored-grafana/* /var/lib/docker/volumes/grafana_data/_data/

# 5. Start services again
systemctl start docker  # or docker-compose up -d

# 6. Clean up
proxmox-backup-client unmount /mnt/restore
```

### 2.3 Emergency Recovery Scenarios

#### Scenario A: PVE Boot Drive Failure

**Situation:** The OS SSD in the PVE host dies. All containers/VMs are stored on separate disks (ZFS pool or LVM VG) that are still healthy.

```bash
# Step 1: Reinstall Proxmox on a new SSD
# Boot from Proxmox ISO, install to new disk

# Step 2: Import the existing ZFS pool
zpool import -f -R /mnt <pool_name>
# If pool isn't auto-detected:
zpool import -D                     # Show destroyed/offline pools
zpool import -R /mnt <pool_name>    # Import manually

# Step 3: Re-register existing VM/CT configs
# If /etc/pve/ is on the ZFS pool (ZFS-based install):
pve-container restore <CT_ID>
qm rescan

# If configs are lost, re-attach disk images manually:
qm disk import <VM_ID> <disk-path> <storage>   # For QEMU
# For LXC, you need the rootfs tarball — usually lost with OS disk.
# This is why PBS backups are critical.

# Step 4: Scan for orphaned disks
pvesm scan <storage_name>
```

**⚠️ Critical:** Without PBS backups, an OS disk failure loses:
- All LXC containers (they have no independent disk images — their rootfs is on the OS disk unless stored on separate storage)
- VM config files (but VM disk images on separate storage survive)
- PVE host configuration

**Prevention:** Store VMs/LXCs on ZFS- or LVM-based storage separate from the OS disk.

#### Scenario B: Importing Orphaned Disk Images

When you have raw/qcow2 disk images on a healthy ZFS pool but no VM configs:

```bash
# Step 1: Find all disk images
find /<zpool> -name "*.qcow2" -o -name "*.raw" -o -name "vm-*-disk-*"
ls /dev/<vg>/ | grep vm-

# Step 2: Create a new VM with matching specs
qm create <VM_ID> --name <name> --memory <MB> --cores <N> --scsihw virtio-scsi-single

# Step 3: Import the disk
qm disk import <VM_ID> /path/to/vm-<OLD_VMID>-disk-0.qcow2 local-zfs --format qcow2

# Step 4: Attach and set boot
qm set <VM_ID> --scsi0 local-zfs:vm-<VM_ID>-disk-0,iothread=1
qm set <VM_ID> --boot order=scsi0

# Step 5: For LXC — extract rootfs from backup or recreate from scratch
# LXC rootfs is a subvolume/dataset, not a standalone file.
# Without PBS backup, you must rebuild the container.
```

#### Scenario C: Corrupted LXC Root Filesystem

```bash
# Step 1: Stop the container
pct stop <CT_ID>

# Step 2: Run fsck on the raw image
# Find the image path:
pct config <CT_ID> | grep rootfs
# Example: rootfs: local-zfs:subvol-<CT_ID>-disk-0

# For ZFS subvolume (resides in pool):
zfs get all <pool>/subvol-<CT_ID>-disk-0
zfs scrub <pool>

# For raw image file:
fsck.ext4 -y /var/lib/vz/images/<CT_ID>/vm-<CT_ID>-disk-0.raw

# Step 3: Start and verify
pct start <CT_ID>
pct enter <CT_ID> -- bash -c 'mount -o remount,rw / && df -h'
```

#### Scenario D: Failed PVE Upgrade

```bash
# If PVE upgrade corrupts pveproxy or web UI:
systemctl status pveproxy
journalctl -u pveproxy -n 50 --no-pager

# Fix common issues:
# 1. Remove enterprise repo
rm -f /etc/apt/sources.list.d/pve-enterprise.list

# 2. Reinstall pveproxy
apt install --reinstall pveproxy

# 3. Fix database corruption
pvecm updatecerts --force
systemctl restart pveproxy pvedaemon

# 4. Last resort: restore from PBS bare-metal backup
# Boot from Proxmox ISO → "Restore from Proxmox Backup Server"
```

#### Scenario E: UPS-Triggered Shutdown Recovery

When power returns and all containers boot simultaneously:

```bash
# Check what started:
pct list
# Verify DNS is up first:
pct exec 107 -- pihole status 2>/dev/null || echo "Pi-hole still down"
# Check Docker stack:
pct exec 106 -- docker ps

# If containers failed due to race condition (NTP sync, DNS, dependencies):
pct stop 106               # Stop monitoring stack
sleep 30                    # Wait for Pi-hole + NTP to stabilize
pct start 106              # Start monitoring stack cleanly
docker compose -f /opt/monitoring/docker-compose.yml up -d  # From inside CT 106

# Force staggered reboots:
for ct in 102 103 104 100 106; do
  pct start $ct
  sleep 20
done
```

---

## PILLAR 3: SYSTEM MAINTENANCE & AUTOMATION

### 3.1 Patch Management

#### PVE Host Patch Schedule

```bash
# 1. Disable enterprise repos (they require a subscription)
rm -f /etc/apt/sources.list.d/pve-enterprise.list

# 2. Add no-subscription repo
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-no-subscription.list

# 3. Test update
apt update
apt list --upgradable

# 4. Apply updates
apt dist-upgrade -y

# 5. Check if reboot is needed
[ -f /var/run/reboot-required ] && echo "REBOOT REQUIRED" || echo "No reboot needed"

# 6. Reboot if kernel updated
systemctl reboot
```

#### Automated Cran with Cron

```bash
# /etc/cron.d/pve-maintenance — PVE host updates
# Run every other day at 02:00
0 2 */2 * * root \
  bash -c '\
    apt update -qq 2>&1 | logger -t pve-update && \
    apt dist-upgrade -y -qq 2>&1 | logger -t pve-upgrade && \
    [ -f /var/run/reboot-required ] && \
    echo "PVE $(cat /etc/debian_version): Kernel update requires reboot" | \
    mail -s "PVE Reboot Required" root \
  '

# Alternative: notify Discord via webhook
0 2 */2 * * root \
  bash -c '\
    LOG=/tmp/pve-update-$(date +%Y%m%d).log; \
    apt update -qq 2>&1 | tee $LOG; \
    apt dist-upgrade -y -qq 2>&1 | tee -a $LOG; \
    if [ -f /var/run/reboot-required ]; then \
      curl -s -H "Content-Type: application/json" \
        -d "{\"content\":\"🔄 **PVE Reboot Required** — Kernel updated. Schedule maintenance window.\"}" \
        <DISCORD_WEBHOOK_URL> > /dev/null; \
    fi \
  '
```

#### Container Patch Management (via PVE host)

```bash
# Update ALL containers in sequence
for ct in $(pct list | awk 'NR>1{print $1}'); do
  echo "=== Updating CT $ct ($(pct config $ct | grep hostname | cut -d: -f2)) ==="
  pct exec $ct -- bash -c '
    echo "nameserver 10.2.7.2" > /etc/resolv.conf
    apt update -qq 2>/dev/null && apt upgrade -y -qq 2>/dev/null
  ' 2>&1 | tail -3
  echo ""
done
```

**Note:** The `pct exec` approach will abort on the first container that doesn't have the required commands. For more robust updates, see the `wazuh-bulk-deployment` pattern — use SSH key-based access to containers individually.

#### Container Update with Error Isolation

```bash
for ct in $(pct list | awk 'NR>1{print $1}'); do
  (
    name=$(pct config $ct 2>/dev/null | grep hostname | awk '{print $2}')
    pct exec $ct -- bash -c '
      echo "nameserver 10.2.7.2" > /etc/resolv.conf
      apt-get update -qq && apt-get upgrade -y -qq
    ' 2>&1 | tail -1
    echo "CT $ct ($name): done"
  ) || echo "CT $ct: FAILED — skipping"
done
```

#### NUT UPS Package Updates

The NUT (Network UPS Tools) on PVE host uses a custom `local.conf` to bypass the Debian repos. See [`nut-ups-pve-setup.md` in the proxmox skill](https://github.com/iNfamousJerk/homelab-docs) for the exact config.

```bash
# Check if 'disabled' repos are blocking nut updates:
apt-cache policy nut
# If version is wrong, check /etc/apt/sources.list.d/ for disabled entries

# NUT daemon restart after update
systemctl restart nut-server nut-monitor
upsc cyberpower@localhost  # Verify connectivity
```

### 3.2 Storage Optimization

#### ZFS Pool Scrubs

```bash
# Check current scrub status
zpool status -v <pool>
# Sample output:
#   scan: scrub repaired 0B in 00:03:15 with 0 errors on Sun May 24 02:15:00 2026

# Manual scrub
zpool scrub <pool>

# Wait for completion and verify
zpool status <pool> | grep -E "scan:|errors:"

# Schedule automatic scrubs (built into Proxmox)
# /etc/cron.d/zfsutils-linux — distributed with package
# Default: 1st Sunday of month at 0:00 for pools less than 50TB
# Check:
grep -r "zpool.*scrub" /etc/cron* /etc/anacrontab

# Force weekly scrub for homelab:
echo "0 3 * * 0 root zpool scrub backup" > /etc/cron.d/zfs-scrub-backup
```

**What scrubs detect:** Checksum mismatches (bit rot), read errors, device timeouts, pending sector reallocations. A scrub that finds no errors = filesystem is bit-perfect.

**What scrubs don't detect:** Disk mechanical failure (clicking, vibration), controller issues, cable faults. Those show as device timeouts in `zpool status`.

#### SMART Monitoring

```bash
# Install
apt install smartmontools

# Enable for all disks
smartctl --scan
# Add to /etc/smartd.conf:
# /dev/sda -a -o on -S on -s (S/../.././02|L/../../7/03) -m root

# Quick health check
for disk in /dev/sd[a-z]; do
  echo "=== $disk ==="
  smartctl -H $disk | grep "SMART overall-health"
  smartctl -A $disk | grep -E "Reallocated_Sector|Pending_Sector|Uncorrectable|UDMA_CRC"
done
```

#### Continuous TRIM / fstrim

```bash
# On PVE host (trims all mounted ZFS/LVM pools):
fstrim -av

# Schedule weekly trim
systemctl enable fstrim.timer
systemctl start fstrim.timer

# Check timer status
systemctl status fstrim.timer
# Defaults to: Mon 00:00, weekly

# For ZFS pools specifically:
zpool trim <pool>

# Check trim progress:
zpool status -t <pool>

# For Docker volumes on thin-provisioned storage, run inside containers:
pct exec <CT_ID> -- fstrim -av

# Or from PVE host for all containers:
for ct in $(pct list | awk 'NR>1{print $1}'); do
  pct exec $ct -- fstrim -v / 2>/dev/null || true
done
```

#### ZFS ARC Tuning

```bash
# Check current ARC usage
arc_summary
# Or from /proc:
cat /proc/spl/kstat/zfs/arcstats | grep -E "size|max|c_min|c_max" | head -6

# Check RAM consumed by ARC
echo "ARC: $(grep -oP 'size\s+\K\d+' /proc/spl/kstat/zfs/arcstats) bytes"
echo "RAM: $(free -b | awk '/Mem:/{print $2}') bytes"

# Limit ARC to 25% of RAM (prevents ARC from starving other processes)
# /etc/modprobe.d/zfs.conf:
options zfs zfs_arc_max=$(( $(awk '/MemTotal/{print $2}' /proc/meminfo) * 1024 * 25 / 100 ))

# Or set temporarily (until reboot):
echo $(( $(awk '/MemTotal/{print $2}' /proc/meminfo) * 1024 * 25 / 100 )) > /sys/module/zfs/parameters/zfs_arc_max

# Tuning guidelines:
# - <8GB RAM: set zfs_arc_max to 512MB-1GB
# - 8-16GB RAM: set zfs_arc_max to 1-2GB
# - 16-32GB RAM: default adaptive is fine (capped at 50% of RAM)
# - >32GB RAM: default adaptive, no action needed
```

#### Disk Usage Alerts

```bash
# Quick check
df -h | grep -v tmpfs | grep -v overlay | grep -v loop

# Check ZFS pool capacity (warning at 80%, critical at 95%)
zpool list

# Percent full
zpool get capacity <pool>
# NAME    PROPERTY   VALUE  SOURCE
# backup  capacity   43%    -

# Set up a simple threshold check
THRESHOLD=80
zpool list -H -o capacity,pool | while read cap pool; do
  cap=${cap%\%}
  if [ "$cap" -gt "$THRESHOLD" ]; then
    echo "WARNING: Pool $pool at ${cap}% capacity!"
  fi
done
```

### 3.3 Metrics & Monitoring

#### Proxmox Metric Server Setup

Proxmox has a built-in metric server that can push host data to external monitoring:

```bash
# Web UI: Datacenter → Metric Server → Add
# Or CLI:
pvesh create /cluster/metrics/server \
  --type influxdb \
  --server 10.2.7.108 \
  --port 8086 \
  --protocol http \
  --organization pve \
  --bucket proxmox \
  --token <influxdb-token>

# Graphite format (for Prometheus via carbon-relay or directly)
pvesh create /cluster/metrics/server \
  --type graphite \
  --server 10.2.7.108 \
  --port 2003
```

#### Prometheus Integration

```bash
# Install pve-exporter on PVE host (or in a Docker container on CT 106)
# Docker image: prompve/prometheus-proxmox-exporter

docker run -d \
  --name pve-exporter \
  --restart always \
  -p 9221:9221 \
  -e PVE_USER="root@pam" \
  -e PVE_PASSWORD="<password>" \
  -e PVE_VERIFY_SSL="false" \
  prompve/prometheus-proxmox-exporter

# In prometheus.yml on CT 106:
# scrape_configs:
#   - job_name: 'pve'
#     static_configs:
#       - targets: ['10.2.7.64:9221']
```

#### Alternative: node_exporter on PVE Host

Simpler than pve-exporter — exposes standard host metrics (CPU, RAM, disk, network):

```bash
# Install node_exporter on PVE host
# Option 1: Docker on CT 106 (probes PVE via SSH)
# Option 2: Direct install on PVE host
wget https://github.com/prometheus/node_exporter/releases/latest/download/node_exporter-*.linux-amd64.tar.gz
tar xzf node_exporter-*.linux-amd64.tar.gz
cp node_exporter-*/node_exporter /usr/local/bin/

# Create systemd service
cat > /etc/systemd/system/node_exporter.service << 'EOF'
[Unit]
Description=Prometheus Node Exporter
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/node_exporter \
  --collector.filesystem.mount-points-exclude="^/(dev|proc|sys|run)($$|/)" \
  --collector.netclass.ignored-devices="^(veth|docker|br-|tun|tap).*"
Restart=always

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now node_exporter
```

#### PBS Monitoring

```bash
# PBS exposes Prometheus metrics at port 8080 by default
curl -s http://10.2.7.65:8080/metrics | head -20

# In prometheus.yml:
#   - job_name: 'pbs'
#     static_configs:
#       - targets: ['10.2.7.65:8080']

# Check backup status from PVE cluster
pvesh get /cluster/backup
pvesh get /nodes/pve/tasks --typefilter backup
```

#### Prometheus Health Check (One-Liner)

```bash
# Quick health check — all targets UP?
curl -s http://10.2.7.108:9090/api/v1/targets | \
  python3 -c "import json,sys; d=json.load(sys.stdin); \
  [print(f'{t[\"labels\"][\"job\"]:20s} → {t[\"health\"]}') for t in d['data']['activeTargets']]"
```

---

## PILLAR 4: LIFECYCLE SUPPORT & TROUBLESHOOTING

### 4.1 Triage Diagnostics

#### Quick Health Scan (PVE Host)

```bash
# Run this first for any issue:
echo "=== PVE SYSTEM OVERVIEW ==="
echo "--- Load ---";              uptime
echo "--- Memory ---";            free -h
echo "--- ZFS Pools ---";         zpool list
echo "--- ZFS Errors ---";        zpool status -x
echo "--- Disk ---";              df -h | grep -v tmpfs | grep -v overlay
echo "--- Containers ---";        pct list
echo "--- VMs ---";               qm list
echo "--- PVE Services ---";      systemctl status pveproxy pvedaemon pve-ha-lrm pve-cluster corosync | grep -E "●|Active:"
echo "--- Pending Tasks ---";     pvesh get /cluster/tasks --running 1 | head -10
```

#### Container Health Scan

```bash
echo "=== LXC HEALTH ==="
echo "--- All Containers ---"
pct list | awk 'NR>1{printf "CT %-4s %-20s %-10s %s/%s CPU\n", $1, $2, $3, $5, $6}'

echo ""
echo "--- Stopped Containers ---"
for ct in $(pct list | awk '$3=="stopped"{print $1}'); do
  echo "⚠️  CT $ct ($(pct config $ct | grep hostname | awk '{print $2}')) is STOPPED"
done

echo ""
echo "--- High Memory ---"
for ct in $(pct list | awk 'NR>1{print $1}'); do
  mem_used=$(pct exec $ct -- free -m 2>/dev/null | awk '/Mem:/{print $3}')
  mem_total=$(pct config $ct | grep memory | awk '{print $2}')
  if [ -n "$mem_used" ] && [ "$mem_used" -gt "$((mem_total * 80 / 100))" ]; then
    echo "🔴 CT $ct: memory ${mem_used}MB / ${mem_total}MB (>80%)"
  fi
done

echo ""
echo "--- High Disk ---"
for ct in $(pct list | awk 'NR>1{print $1}'); do
  disk_pct=$(pct exec $ct -- df / 2>/dev/null | awk 'NR>1{print $5}' | tr -d '%')
  if [ -n "$disk_pct" ] && [ "$disk_pct" -gt 80 ]; then
    echo "🔴 CT $ct: disk ${disk_pct}% full"
  fi
done
```

#### PBS Health Check

```bash
echo "=== PBS HEALTH ==="
echo "--- Datastores ---"
proxmox-backup-manager datastore list

echo "--- Verification Status ---"
proxmox-backup-manager verification list

echo "--- Recent Backups ---"
proxmox-backup-manager backup list 2>/dev/null | head -10

echo "--- Disk Usage ---"
df -h /backup/

echo "--- Sync Jobs ---"
proxmox-backup-manager sync-job list

echo "---Last 10 log lines ---"
journalctl -u proxmox-backup -n 10 --no-pager | tail -10
```

### 4.2 Common Failure Modes & Fixes

#### ZFS Pool Degradation

```bash
# Symptom: pct/qm operations hang, I/O errors in dmesg
zpool status -x
# Expected: "all pools are healthy"
# Actual:  "pool 'backup' is DEGRADED"
#         "One or more devices has experienced an unrecoverable error"

# Step 1: Identify failed device
zpool status -v <pool>
# Look for: FAULTED, DEGRADED, OFFLINE, or errors:
#   NAME                     STATE     READ WRITE CKSUM
#   backup                   DEGRADED     0     0     0
#     mirror-0               DEGRADED     0     0     0
#       sda                  ONLINE       0     0     0
#       sdb                  FAULTED      5    20     0

# Step 2: Check SMART on the failing drive
smartctl -H /dev/sdb
smartctl -A /dev/sdb | grep -E "Reallocated|Pending|Uncorrectable|UDMA_CRC|Current_Pending"

# Step 3: If disk is physically failing, replace it
zpool offline <pool> /dev/sdb
# Physically replace the disk
zpool replace <pool> /dev/sdb /dev/sd<new>
zpool scrub <pool>

# Step 4: If CHECKSUM errors on the healthy mirror member (bit rot on the *other* disk):
# This means the data is actually corrupt — restore from PBS backup
zpool clear <pool> /dev/sda    # Clear error counters
zpool scrub <pool>
# If errors return, restore the affected CT/VM from backup

# Step 5: If disk is healthy but ZFS marked it (transient error):
zpool clear <pool> /dev/sdb
zpool scrub <pool>
```

#### Stuck/Locked LXC Container

```bash
# Symptom: "can't lock file '/var/lock/qemu-server/lock-<CT_ID>.conf'"
# or: pct commands hang indefinitely

# Step 1: Check if process is genuinely stuck
pct status <CT_ID> --verbose

# Step 2: Force unlock
pct unlock <CT_ID>

# Step 3: If that fails, manually remove the lock file
ls -la /var/lock/qemu-server/lock-<CT_ID>.conf
rm -f /var/lock/qemu-server/lock-<CT_ID>.conf

# Step 4: If container itself is hung ("status: running" but not responding):
pct stop <CT_ID>
# If stop hangs, force:
pct stop <CT_ID> --skiplock

# Step 5: Kill any stuck processes
kill -9 $(pgrep -f "pct.*<CT_ID>") 2>/dev/null
kill -9 $(pgrep -f "qm.*<CT_ID>") 2>/dev/null

# Step 6: Start fresh
pct start <CT_ID>

# Step 7: Clean up leftover mount points
mount | grep "<CT_ID>" && umount /var/lib/lxc/<CT_ID>/rootfs
```

#### Corosync/Quorum Loss (Single Node)

```bash
# Symptom: Web UI shows "HA cluster not ready" or "No quorum"
pvecm status
# Expected: Quorum: 1 (single-node cluster always has quorum)
# Actual:  "Quorate: NO" or "Expected votes: 2"

# Step 1: Check expected votes
pvecm expected 1
# Forces single-node cluster to have quorum even if corosync thinks it's multi-node

# Step 2: If that doesn't stick, reinit corosync
pvecm expected 1
killall corosync
sleep 2
systemctl start corosync

# Step 3: Verify
pvecm status | grep Quorate

# Step 4: If all else fails, force quorum at the corosync level
# /etc/corosync/corosync.conf — set quorum.two_node: yes for 2-node clusters
# For single-node: quorum.device is not needed. Just set expected votes.
corosync-cfgtool -s   # Show ring status
```

#### PVE Web UI Not Loading

```bash
systemctl status pveproxy
journalctl -u pveproxy -n 30 --no-pager

# Common fixes:
systemctl restart pveproxy pvedaemon
pvecm updatecerts --force
systemctl restart pveproxy

# If SSL cert is expired:
rm /etc/pve/local/pve-ssl.*
pvecm updatecerts --force
systemctl restart pveproxy

# If port 8006 is already in use:
ss -tulpn | grep 8006
killall pveproxy && systemctl start pveproxy
```

#### Container Network Issues

```bash
# Symptom: Can't reach network from inside a container

# Step 1: Check if container has an IP
pct config <CT_ID> | grep net0

# Step 2: From inside:
pct exec <CT_ID> -- ip addr show eth0
pct exec <CT_ID> -- ip route show default
pct exec <CT_ID> -- ping -c 2 10.2.7.1    # Test gateway
pct exec <CT_ID> -- ping -c 2 10.2.7.2    # Test DNS

# Step 3: If no IP, set it:
pct set <CT_ID> --net0 name=eth0,bridge=vmbr0,ip=10.2.7.X/24,gw=10.2.7.1

# Step 4: If DNS fails:
pct exec <CT_ID> -- echo "nameserver 10.2.7.2" > /etc/resolv.conf
pct exec <CT_ID> -- dig google.com @10.2.7.2

# Step 5: If firewalld or iptables is blocking:
pct exec <CT_ID> -- iptables -L -n | head -20
pct exec <CT_ID> -- iptables -F    # Flush rules (temporary)
```

#### PBS Not Accepting Backups

```bash
# Symptom: Backup fails with "500 connection error" or "authentication failed"

# Step 1: Test PBS connectivity
curl -sk https://10.2.7.65:8007/api2/json/version
# Should return JSON with version info

# Step 2: Check PBS service
ssh root@10.2.7.65
systemctl status proxmox-backup
journalctl -u proxmox-backup-proxy -n 20 --no-pager

# Step 3: Check datastore disk space
df -h /backup/datastore/
zpool list backup
# PBS refuses new backups if datastore is >95% full

# Step 4: Verify credentials
proxmox-backup-manager user list
# Test PVE → PBS auth (from PVE):
pvesh get /storage/pbs/status

# Step 5: If fingerprint changed (cert reissued):
pvesh set /storage/pbs --fingerprint <NEW_FINGERPRINT>

# Step 6: Check backup verification queue isn't stuck
proxmox-backup-manager verification list
# If stuck, cancel and restart:
proxmox-backup-manager verification abort

# Step 7: Restart PBS services
systemctl restart proxmox-backup-proxy proxmox-backup
```

#### Docker-in-LXC Specific Issues

See the [`proxmox` skill](devops/proxmox/SKILL.md) for the full AppArmor fix and Docker daemon recovery.

```bash
# Symptom: "docker: Error response from daemon: AppArmor enabled on system"
# Fix: Add security_opt to each service in docker-compose.yml:
#   security_opt:
#     - apparmor=unconfined

# Symptom: Docker daemon won't restart after config change
# Fix:
rm -f /etc/docker/daemon.json
kill -9 $(cat /var/run/docker.pid) 2>/dev/null
rm -f /var/run/docker.pid
systemctl start docker

# Symptom: "overlay2: mount failed: operation not permitted"
# Fix: Enable nesting=1 on the container
pct set <CT_ID> --features nesting=1
# Then reboot container
pct reboot <CT_ID>

# Symptom: Docker pull fails with DNS error
# Fix: Nameserver in container must be external (10.2.7.2), not PVE host
pct exec <CT_ID> -- echo "nameserver 10.2.7.2" > /etc/resolv.conf
```

#### SSH Key Deployment Failures

```bash
# When SSH key-based auth stops working:
# Step 1: Check permissions on the container
ssh -vvv root@<CT_IP> 2>&1 | grep -i "permission denied\|authenticate\|publickey"

# Step 2: Verify authorized_keys exists and has correct permissions
pct exec <CT_ID> -- ls -la ~/.ssh/
pct exec <CT_ID> -- cat ~/.ssh/authorized_keys
# Must be: 600 permissions, owned by root:root

# Step 3: Fix permissions
pct exec <CT_ID> -- bash -c '
  chmod 700 ~/.ssh
  chmod 600 ~/.ssh/authorized_keys
  chown -R root:root ~/.ssh
'

# Step 4: Check sshd config
pct exec <CT_ID> -- grep -E "PubkeyAuthentication|AuthorizedKeysFile|PasswordAuthentication" /etc/ssh/sshd_config

# Step 5: Restart SSH
pct exec <CT_ID> -- systemctl restart ssh

# Step 6: Fallback — use pct enter to re-deploy key
pct enter <CT_ID>
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo "<PUBKEY>" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
exit
```

### 4.3 Log Analytics

#### Critical Log File Map

| Log Path | What It Contains | When to Check |
|----------|-----------------|---------------|
| `/var/log/pve/tasks/` | Active/Historical PVE task queue entries | Backup failures, resize errors, long-running tasks |
| `/var/log/pve/tasks/index` | SQLite DB of ALL tasks (by date) | Audit trail of admin actions |
| `/var/log/syslog` | Kernel, system, and PVE daemon logs | Almost everything — container start failures, OOM, ZFS events |
| `/var/log/pveproxy/access.log` | Web UI access log | Auth failures, unusual IPs hitting port 8006 |
| `/var/log/pve-firewall.log` | PVE host firewall rules (if enabled) | Connection rejections, port scans |
| `/var/log/proxmox-backup/proxy.log` | PBS proxy — web UI auth + requests | PBS login failures, API issues |
| `/var/log/proxmox-backup/data/` | Per-datastore backup task logs | Failed backups, CRC errors, verification issues |
| `/var/log/proxmox-backup/tasks/` | All PBS task queue entries | Async job status (verification, GC, sync) |
| `/var/log/corosync/corosync.log` | Cluster communication logs | Quorum issues, ring errors (multi-node only) |
| `journalctl -u pveproxy` | PVE proxy daemon journal | SSL/cert errors, HTTP 500s on web UI |
| `journalctl -u corosync` | Corosync ring journal | Cluster reconnection attempts |
| `journalctl -u zfs-zed` | ZFS Event Daemon | Disk errors, pool state changes, predictive failures |

#### Quick Log Commands

```bash
# Last 20 PVE tasks (most recent first)
ls -lt /var/log/pve/tasks/ | head -20

# PVE tasks for a specific container
grep "<CT_ID>" /var/log/pve/tasks/active 2>/dev/null

# Check ZFS events for disk errors
journalctl -u zfs-zed -n 20 --no-pager

# Monitor live container start
tail -f /var/log/pve/tasks/active &
pct start <CT_ID>

# Get last 50 lines of syslog, filtering for relevant errors
grep -E "error|fail|oom-kill|out of memory|I/O error" /var/log/syslog | tail -50

# Check for OOM kills
grep -i "oom" /var/log/syslog

# Check for filesystem errors
grep -E "EXT4-fs error|XFS.*error|btrfs.*error" /var/log/syslog

# Check for disk I/O errors in kernel ring buffer
dmesg | grep -E "I/O error|ata.*error|sd.*err|buffer I/O" | tail -20

# Check PBS task log for the last backup run
ls -lt /var/log/proxmox-backup/tasks/ | head -5
cat /var/log/proxmox-backup/tasks/$(ls -t /var/log/proxmox-backup/tasks/ | head -1)

# Disk health log
smartctl -l error /dev/sda  # View disk error log
```

#### Setting Up Log Rotation for PBS

```bash
# Default rotation is aggressive. Check current config:
cat /etc/logrotate.d/proxmox-backup

# Typical configuration:
# /var/log/proxmox-backup/*.log {
#     daily
#     rotate 30
#     compress
#     delaycompress
#     missingok
#     notifempty
#     create 640 backup backup
#     sharedscripts
#     postrotate
#         systemctl kill -s HUP proxmox-backup-proxy 2>/dev/null || true
#     endscript
# }

# For PVE tasks (SQLite DB doesn't rotate — it's a finite rolling buffer)
# PVE auto-cleans tasks older than 30 days by default
pvesh get /cluster/tasks --vmid <CT_ID>
```

#### Using PVE API to Search Tasks

```bash
# List tasks for a specific VMID/CTID (last 20)
pvesh get /cluster/tasks --vmid <CT_ID> --limit 20

# Show running tasks
pvesh get /cluster/tasks --running 1

# Search for failed backups
pvesh get /cluster/tasks --typefilter backup --statusfilter error
```

---

## Quick-Reference Command Index

| Task | Command |
|------|---------|
| Create LXC | `pct create <CT_ID> <template> --hostname <n> --storage <s>:<size> [--flags]` |
| Create VM | `qm create <VM_ID> --name <n> --memory <M> --cores <N> [--flags]` |
| Resize LXC disk | `pct resize <CT_ID> rootfs <new_size>G` |
| Resize VM disk | `qm resize <VM_ID> scsi0 <new_size>G` |
| Start container | `pct start <CT_ID>` |
| Stop container | `pct stop <CT_ID>` |
| Enter container shell | `pct enter <CT_ID>` |
| List all containers | `pct list` |
| List all VMs | `qm list` |
| Snapshot | `pct snapshot <CT_ID> <name> [--description ""]` |
| Rollback | `pct rollback <CT_ID> <snapshot>` |
| Unlock stuck CT | `pct unlock <CT_ID>` |
| Force unlock | `rm -f /var/lock/qemu-server/lock-<CT_ID>.conf` |
| PVE health check | `pvesh get /cluster/resources?type=vm` |
| List PBS backups | `proxmox-backup-manager backup list` |
| Restore CT from PBS | `pct restore <CT_ID> <backup-volume> --storage <pool>` |
| Mount PBS backup | `proxmox-backup-client mount main:ct/<ID>/<backup> /mnt/restore` |
| ZFS pool check | `zpool status -x` |
| ZFS scrub | `zpool scrub <pool>` |
| ZFS disk replace | `zpool replace <pool> <old> <new>` |
| Fix quorum (single node) | `pvecm expected 1` |
| PVE log check | `journalctl -u pveproxy -n 30 --no-pager` |
| Container log | `pct logs <CT_ID> \| tail -50` |
| Update PVE | `apt update && apt dist-upgrade -y` |
| Update all containers | `for ct in $(pct list \| awk 'NR>1{print $1}'); do pct exec $ct -- apt update -qq && apt upgrade -y -qq; done` |
| TRIM all | `fstrim -av` |
| Verify PBS connectivity | `curl -sk https://10.2.7.65:8007/api2/json/version` |
| List PVE tasks for CT | `pvesh get /cluster/tasks --vmid <CT_ID>` |
| Staggered boot seq | `pct set <CT_ID> --onboot 1 --startup order=N,up=M` |
| Remove enterprise repo | `rm -f /etc/apt/sources.list.d/pve-enterprise.list` |
| Enable no-sub repo | `echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list` |
| Rename container | `pct set <CT_ID> --hostname <new_name>` |
| Move CT to diff storage | `pct migrate <CT_ID> <target-storage>` |
| Convert VM to template | `qm template <VM_ID>` |
| Create linked clone | `qm clone <TEMPLATE_ID> <NEW_VM_ID> --name <name>` |
| Check ZFS ARC size | `cat /proc/spl/kstat/zfs/arcstats \| grep -E "size\|c_max"` |
| List LXC IPs (all) | `for ct in $(pct list \| awk 'NR>1{print $1}'); do echo -n "CT $ct: "; pct config $ct \| grep net0 \| grep -oP 'ip=\K[0-9./]+'; done` |

---

> **Document version:** 1.0 — Created 2026-05-25
> **Previous references:** `01-proxmox.md` (basic), `15-pbs-setup.md` (hardware config)
> **Related playbooks:** `DR-IR-PLAYBOOK.md` (emergency scenarios), `SOC-UPGRADE-PLAN.md` (phase progression)
