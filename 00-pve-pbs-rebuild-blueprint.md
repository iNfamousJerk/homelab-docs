# PVE/PBS Rebuild Blueprint — No Helper Scripts

> **Zero-dependency rebuild guide.** Every service in this homelab was originally deployed via [community-scripts.net](https://community-scripts.net) (formerly tteck's Proxmox Helper Scripts). This document maps out how to rebuild everything from scratch using only raw upstream git repos, official APT repos, official Docker images, and hand-written systemd units. No third-party installers, no wrappers, no magic.
>
> **Target audience:** This is the SRE who just suffered a complete host failure and needs to rebuild without internet access to helper scripts — or who simply wants a version-pinned, script-free, audit-ready infrastructure.

---

## Table of Contents

- [Inventory: What the Scripts Touched](#inventory-what-the-scripts-touched)
- [Phase 1: Config Extraction Before Rebuild](#phase-1-config-extraction-before-rebuild)
- [Phase 2: Bare-Metal Rebuild from Scratch](#phase-2-bare-metal-rebuild-from-scratch)
  - [2.1 PVE Host: Base Install](#21-pve-host-base-install)
  - [2.2 PBS: Base Install](#22-pbs-base-install)
  - [2.3 Storage & Networking: Raw CLI](#23-storage--networking-raw-cli)
  - [2.4 PVE ↔ PBS Pairing (No Web UI)](#24-pve--pbs-pairing-no-web-ui)
  - [2.5 DNS: Pi-hole (CT 107)](#25-dns-pi-hole-ct-107)
  - [2.6 Agent: Hermes (CT 100)](#26-agent-hermes-ct-100)
  - [2.7 Monitoring: Docker Stack (CT 106)](#27-monitoring-docker-stack-ct-106)
  - [2.8 Wazuh SIEM (CT 105)](#28-wazuh-siem-ct-105)
  - [2.9 SSO & Dashboard: Homarr (CT 103)](#29-sso--dashboard-homarr-ct-103)
  - [2.10 Photos: Immich (CT 101)](#210-photos-immich-ct-101)
  - [2.11 Storage: Nextcloud (CT 104)](#211-storage-nextcloud-ct-104)
  - [2.12 Network Monitor: PiAlert (CT 102)](#212-network-monitor-pialert-ct-102)
  - [2.13 Local LLM: Ollama (CT 108)](#213-local-llm-ollama-ct-108)

---

## Inventory: What the Scripts Touched

| Service | CT | Script Used | Alternative (Raw) |
|---------|----|-------------|-------------------|
| **Pi-hole** | 107 | community-scripts Pi-hole installer (git clone) | Official [Pi-hole installer](https://github.com/pi-hole/pi-hole/#one-step-automated-install) — note: still a script, but it's the **upstream vendor's** script from GitHub. Or manual tarball install. |
| **PiAlert** | 102 | community-scripts PiAlert installer (puts app in `/opt/pialert`) | Official [PiAlert GitHub](https://github.com/leiweibau/Pi.Alert) — manual git clone + systemd unit |
| **Nextcloud** | 104 | community-scripts Nextcloud installer (Apache, `/var/www/nextcloud`) | Official Nextcloud tarball from [nextcloud.com/install](https://nextcloud.com/install) or official Docker image |
| **Homarr** | 103 | community-scripts Homarr installer | Official [Homarr Docker image](https://github.com/ajnart/homarr) — write compose file manually |
| **Immich** | 101 | community-scripts Immich installer | Official [Immich GitHub](https://github.com/immich-app/immich) — `git clone` + `.env` + `docker compose up -d` |
| **Hermes Agent** | 100 | container created from script template | `pct create` with flags from the lifecycle playbook |
| **Ollama** | 108 | community-scripts Ollama installer (Ubuntu) | Official [Ollama install script](https://ollama.com/install.sh) (vendor-signed) or manual tarball from [GitHub releases](https://github.com/ollama/ollama/releases) |
| **monitoring stack** | 106 | **None** (hand-written docker-compose.yml) | Already raw — no migration needed |
| **Wazuh** | 105 | **None** (official APT repo) | Already raw — no migration needed |
| **PBS** | z230 | **None** (official APT repo) | Already raw — no migration needed |
| **node_exporter** | 106 host | **None** (manual systemd unit) | Already raw — no migration needed |

---

## Phase 1: Config Extraction Before Rebuild

Before blowing anything away, extract every piece of durable configuration from the script-installed containers. These files are what you cannot regenerate from memory.

### 1.1 Container Checkout Checklist

```bash
# Create a staging directory
mkdir -p ~/rebuild-extract/{pihole,pialert,nextcloud,homarr,immich,ollama}

# ========== Pi-hole (CT 107) ==========
ssh root@10.2.7.2
# Extract EVERYTHING durable:
tar czf /tmp/pihole-config.tar.gz \
  /etc/pihole/setupVars.conf \
  /etc/pihole/custom.list \
  /etc/pihole/custom-adlists.list \
  /etc/dnsmasq.d/*.conf \
  /etc/pihole/FTL.conf \
  /etc/pihole/adlists.list \
  /etc/pihole/local.list \
  /etc/pihole/migrate-v5.conf \
  /var/log/pihole/*.log \
  /etc/lighttpd/*.conf \
  2>/dev/null
scp root@10.2.7.2:/tmp/pihole-config.tar.gz ~/rebuild-extract/pihole/

# Also extract Pi-hole v6 DB for DHCP leases:
ssh root@10.2.7.2 "sqlite3 /etc/pihole/gravity.db '.dump' > /tmp/gravity-dump.sql && sqlite3 /etc/pihole/FTL.db '.dump' > /tmp/ftl-dump.sql"
scp root@10.2.7.2:/tmp/gravity-dump.sql ~/rebuild-extract/pihole/
scp root@10.2.7.2:/tmp/ftl-dump.sql ~/rebuild-extract/pihole/

# ========== PiAlert (CT 102) ==========
ssh root@10.2.7.77
tar czf /tmp/pialert-config.tar.gz \
  /opt/pialert/config/pialert.conf \
  /opt/pialert/config/configuration.conf \
  /opt/pialert/back/pialert.db \
  /opt/pialert/frontend/server.js \
  /opt/pialert/config/version.txt \
  /etc/cron.d/pialert*
scp root@10.2.7.77:/tmp/pialert-config.tar.gz ~/rebuild-extract/pialert/

# ========== Nextcloud (CT 104) ==========
ssh root@10.2.7.99
# Nextcloud config is the critical file — contains DB creds, trusted domains, etc.
cat /var/www/nextcloud/config/config.php
# Extract config + apps config
tar czf /tmp/nextcloud-config.tar.gz \
  /var/www/nextcloud/config/config.php \
  /var/www/nextcloud/config/CAN_INSTALL \
  /var/www/nextcloud/.htaccess \
  /var/www/nextcloud/.user.ini \
  /etc/apache2/sites-available/*.conf \
  /var/www/nextcloud/config/apps.config.php 2>/dev/null
scp root@10.2.7.99:/tmp/nextcloud-config.tar.gz ~/rebuild-extract/nextcloud/

# DB creds (not in config if using socket auth)
ssh root@10.2.7.99 "cat /var/www/nextcloud/config/config.php | grep -E 'dbhost|dbname|dbuser|dbpassword|dbtableprefix'"
# Note: Nextcloud config contains DB password in plaintext. This is normal.

# ========== Homarr (CT 103) ==========
ssh root@10.2.7.105
# Homarr stores config in a SQLite DB or JSON files
find / -path "*/homarr*" -name "*.db" -o -path "*/homarr*" -name "*.json" -o -path "*/homarr*" -name "*.yaml" 2>/dev/null | grep -v proc | grep -v sys
# Typical paths: /var/lib/docker/volumes/homarr_* or /opt/homarr/
# Docker containers: homarr-db, homarr-app
docker ps --format "{{.Names}}" | grep -i homar
docker inspect $(docker ps -q --filter "name=homar") --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{"\n"}}{{end}}'
# Extract SQLite DB (this is the dashboard config)
docker exec homarr-db sqlite3 /path/to/database.sqlite '.dump' > /tmp/homarr-dump.sql 2>/dev/null
# Or copy the DB file directly:
cp $(docker volume inspect homarr_data --format '{{.Mountpoint}}')/database.sqlite /tmp/
scp root@10.2.7.105:/tmp/homarr-dump.sql ~/rebuild-extract/homarr/

# ========== Immich (CT 101) ==========
ssh root@10.2.7.44
cat /opt/immich/.env        # Contains DB password, API key, etc.
cat /opt/immich/docker-compose.yml  # Or hcloud.yml
tar czf /tmp/immich-config.tar.gz /opt/immich/
scp root@10.2.7.44:/tmp/immich-config.tar.gz ~/rebuild-extract/immich/

# ========== Ollama (CT 108) ==========
ssh root@10.2.7.167
# Models are the durable asset — without them, every model must be re-pulled from scratch
ollama list
tar czf /tmp/ollama-config.tar.gz \
  /etc/systemd/system/ollama.service \
  /etc/ollama/config.json 2>/dev/null
scp root@10.2.7.167:/tmp/ollama-config.tar.gz ~/rebuild-extract/ollama/

# Also extract Hermes agent config from CT 100:
ssh root@10.2.7.107
tar czf /tmp/hermes-config.tar.gz \
  /home/hermes/.hermes/config.yaml \
  /home/hermes/.hermes/.env \
  /home/hermes/.ssh/ \
  /home/hermes/.hermes/gateway/ 2>/dev/null
scp root@10.2.7.107:/tmp/hermes-config.tar.gz ~/rebuild-extract/
```

### 1.2 Critical Config Registry

These are the values that **cannot be regenerated** from a fresh install. Store them in a password manager and a paper envelope:

| Value Type | Where It Lives | What to Save |
|------------|---------------|--------------|
| **Pi-hole DHCP reservation map** | `gravity.db` → `dhcp` table | Static leases (MAC→IP→hostname) |
| **Pi-hole adlist URLs** | `adlists.list` | Curated blocklist URLs (Stevens-Google, OISD, etc.) |
| **Pi-hole allow/deny list** | `custom.list`, gravity DB | Any whitelisted domains |
| **Nextcloud DB credentials** | `config.php` → `dbpassword` | MariaDB/PostgreSQL root password + nextcloud user password |
| **Nextcloud trusted_domains** | `config.php` → `trusted_domains` | Allowed proxy/domain entries |
| **Immich DB password** | `/opt/immich/.env` → `DB_PASSWORD` | PostgreSQL password |
| **Immich API key** | `/opt/immich/.env` → `IMMICH_API_KEY` | If generated |
| **Wazuh admin password** | Post-install output or last reset | Dashboard login |
| **Grafana admin password** | `docker-compose.yml` env var | `GF_SECURITY_ADMIN_PASSWORD` |
| **Prometheus pve-exporter password** | `docker-compose.yml` or `prometheus.yml` | PVE root password for exporter |
| **Discord webhook URL** | Wazuh integration config + docker-compose | For alerts |
| **Ollama model list** | `ollama list` | Model names + sizes for re-pull |
| **Hermes config + keys** | `~/.hermes/config.yaml` | Discord bot token, API keys, provider config |
| **Homarr layout** | SQLite DB `database.sqlite` | Dashboard grid, service links, categories |
| **PBS encryption key** | PBS admin or paper copy | `master.key` file or passphrase string |

---

## Phase 2: Bare-Metal Rebuild from Scratch

### 2.1 PVE Host: Base Install

#### Step 1: Install from ISO

Download the [Proxmox VE ISO](https://www.proxmox.com/en/downloads) — Official Proxmox site only. **Do not use any third-party modified ISOs.**

```bash
# 1. Write ISO to USB
dd if=proxmox-ve_*.iso of=/dev/sdX bs=4M status=progress conv=fsync

# 2. Boot from USB on target hardware

# 3. During install:
#    - Filesystem: ZFS (RAID1 if 2+ disks, RAID10 if 4+)
#    - Hostname: pve
#    - IP: 10.2.7.64/24
#    - Gateway: 10.2.7.1
#    - DNS: 10.2.7.2 (Pi-hole) or 1.1.1.1 temporarily
#    - Password: <strong-password>
#    - Email: root@localhost
```

#### Step 2: Post-Install — Switch to No-Subscription Repo

```bash
# Remove the enterprise repo (requires paid subscription)
rm -f /etc/apt/sources.list.d/pve-enterprise.list

# Add no-subscription repo
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-no-subscription.list

# Add Ceph repo (if using Ceph — not needed for this homelab)
# echo "deb http://download.proxmox.com/debian/ceph-quincy bookworm no-subscription" \
#   > /etc/apt/sources.list.d/ceph.list

# Fix Debian sources to include bookworm-updates and bookworm-security
cat > /etc/apt/sources.list << 'EOF'
deb http://deb.debian.org/debian bookworm main contrib non-free non-free-firmware
deb http://deb.debian.org/debian bookworm-updates main contrib non-free non-free-firmware
deb http://security.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware
EOF

# Update
apt update && apt dist-upgrade -y

# Install useful packages
apt install -y \
  curl wget gnupg sudo vim htop \
  smartmontools \
  btop \
  unzip \
  git \
  pve-nvidia-kmod  # only if NVIDIA GPU passthrough planned
```

#### Step 3: Configure SSH Hardening (Manual)

```bash
# Keep it simple — key-only auth, disable root password login
sed -i 's/^#PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
sed -i 's/^#PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
systemctl restart sshd

# Deploy your SSH public key
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo "<your-ed25519-public-key>" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

#### Step 4: Configure Network Bridge (Raw `/etc/network/interfaces`)

```bash
cat > /etc/network/interfaces << 'EOF'
# Network configuration — NO helper scripts, raw Debian networking

auto lo
iface lo inet loopback

auto eno1
iface eno1 inet manual

auto vmbr0
iface vmbr0 inet static
    address 10.2.7.64/24
    gateway 10.2.7.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094

# If a second NIC exists for OPNsense WAN passthrough:
# auto eno2
# iface eno2 inet manual
#
# auto vmbr1
# iface vmbr1 inet manual
#     bridge-ports eno2
#     bridge-stp off
#     bridge-fd 0
EOF

# Apply
systemctl restart networking
```

#### Step 5: Verify PVE is Ready

```bash
# Web UI should be accessible at https://10.2.7.64:8006
systemctl status pveproxy pvedaemon pve-cluster corosync

# Verify no enterprise repo warnings
pveversion -v | head -5

# Check subscription status (will show "no valid subscription" — expected)
pvesubscription get

# Suppress the subscription nag in Web UI (optional):
sed -i "s/.*data.status.*{.*}/if (false) {/" /usr/share/perl5/PVE/API2/Subscription.pm
# NOTE: This modifies a PVE file directly — will be overwritten on package updates.
```

---

### 2.2 PBS: Base Install

#### Step 1: OS Install

Use **Debian 12 Bookworm** (official ISO, not Proxmox's modified Debian):

```bash
# 1. Download from https://www.debian.org/download
# 2. Install: ZFS filesystem, hostname: z230, IP: 10.2.7.65/24
# 3. During install: set root password to strong value
# 4. Do NOT install a desktop environment — just SSH server + standard utilities
```

#### Step 2: Install PBS from Official Repo

```bash
# Add PBS repository
echo "deb http://download.proxmox.com/debian/pbs bookworm pbs-no-subscription" \
  > /etc/apt/sources.list.d/pbs.list

# Add Proxmox GPG key
wget -qO- https://enterprise.proxmox.com/debian/proxmox-release-bookworm.gpg \
  | gpg --dearmor -o /etc/apt/trusted.gpg.d/proxmox-release-bookworm.gpg

# Update and install
apt update && apt dist-upgrade -y
apt install proxmox-backup-server -y

# Configure daemon
systemctl enable --now proxmox-backup.service
systemctl enable --now proxmox-backup-proxy.service
```

#### Step 3: Create Datastore

```bash
# Assuming a ZFS mirror pool called "backup" already exists (see 2.3)
mkdir -p /backup/datastore

proxmox-backup-manager datastore create main /backup/datastore \
  --keep-daily 7 \
  --keep-weekly 4 \
  --keep-monthly 3

# Create dedicated backup user
proxmox-backup-manager user create backup-user@pbs \
  --password '<strong-password>' \
  --comment "PVE backup user"

# Grant restore permissions (not just backup)
proxmox-backup-manager acl update /datastore/main/ \
  --auth-id backup-user@pbs \
  --perm Datastore.Backup,Datastore.Restore

# Enable HTTPS-only
proxmox-backup-manager cert info
```

#### Step 4: Configure Backups Schedule (No-Script, Raw CRON)

```bash
# Direct vzdump from PVE host — no helper script needed
cat > /etc/cron.d/pbs-backups << 'CRON'
# PVE Backup to PBS — daily at 3 AM, snapshot mode, zstd compression
# All containers except those we explicitly exclude
0 3 * * * root /usr/sbin/vzdump --all 1 --mode snapshot --compress zstd --storage pbs --exclude 100 2>&1 | logger -t pbs-backup
CRON

# PBS garbage collection (free space from deleted backups) — weekly
cat > /etc/cron.d/pbs-gc << 'CRON'
# PBS garbage collection — every Sunday at 4 AM
0 4 * * 0 root /usr/bin/proxmox-backup-manager garbage-collection start --datastore main 2>&1 | logger -t pbs-gc
CRON

# PBS datastore verification — verify backup integrity monthly
cat > /etc/cron.d/pbs-verify << 'CRON'
# Verify all backups on the 1st of every month
0 5 1 * * root /usr/bin/proxmox-backup-manager verify-job start --datastore main 2>&1 | logger -t pbs-verify
CRON
```

---

### 2.3 Storage & Networking: Raw CLI

#### ZFS Pool: Performance (CT/VM Storage)

```bash
# Identify disks
ls -la /dev/disk/by-id/ | grep -E "ata-|wwn-" | grep -v part

# Create mirror pool for CT/VM storage
# ashift=12 is optimal for 4K-sector drives (most modern HDDs/SSDs)
# recordsize=128k is optimal for VM/CT mixed workloads
# lz4 compression is near-zero CPU overhead
zpool create -o ashift=12 \
  -O compression=lz4 \
  -O atime=off \
  -O recordsize=128k \
  -O mountpoint=/storage-vm \
  storage-vm mirror \
  /dev/disk/by-id/<ata-DISK1> \
  /dev/disk/by-id/<ata-DISK2>
```

#### ZFS Pool: Backup (PBS Target)

```bash
# Backup pool — large recordsize for sequential backup writes
# zstd-3 compression is stronger than lz4, worthwhile for backup data
zpool create -o ashift=12 \
  -O compression=zstd-3 \
  -O atime=off \
  -O recordsize=1M \
  -O mountpoint=/backup \
  backup mirror \
  /dev/disk/by-id/<ata-DISK1> \
  /dev/disk/by-id/<ata-DISK2>
```

#### Register Storage on PVE

```bash
# Manually add ZFS pool to PVE
pvesm add zfspool local-zfs \
  --pool storage-vm \
  --content rootdir,images \
  --sparse 1
```

#### VLAN-Aware Network (if using OPNsense with VLANs)

```bash
# /etc/network/interfaces — PVE host side
# The vmbr0 bridge is already VLAN-aware (bridge-vlan-aware yes)
# Containers/VMs specify their VLAN tag via pct/qm config:
pct set <CT_ID> --net0 name=eth0,bridge=vmbr0,tag=20,ip=dhcp

# VLAN naming convention:
#   tag=1     → Management / default (infra, monitoring, dns)
#   tag=10    → Infrastructure (PVE management, PBS)
#   tag=20    → Media (Jellyfin, *arr)
#   tag=30    → Guest/IoT (internet only)
```

---

### 2.4 PVE ↔ PBS Pairing (No Web UI)

This is the **critical CLI pathway** for connecting PVE to PBS without ever touching the web UI on either side.

#### Step 1: Get PBS Fingerprint

```bash
# From the PBS host:
proxmox-backup-manager cert info | grep Fingerprint
# Output: Fingerprint (SHA-256): B1:8D:A7:DE:...:06:C4
```

#### Step 2: Register PBS on PVE (CLI)

```bash
# From PVE host:
pvesm add pbs pbs \
  --server 10.2.7.65 \
  --datastore main \
  --username backup-user@pbs \
  --password '<pbs-user-password>' \
  --fingerprint 'B1:8D:A7:DE:...:06:C4'

# If using encryption (recommended for off-site backups):
# Create encryption key file on PVE
openssl rand -hex 32 > /root/pbs-encryption.key
chmod 400 /root/pbs-encryption.key

# Add the encryption config
pvesm set pbs --encryption-key /root/pbs-encryption.key

# Verify it's working
pvesm list pbs
```

#### Step 3: Test Backup from CLI

```bash
# Test backup of a single CT (CT 107 - Pi-hole, small and critical)
vzdump 107 \
  --storage pbs \
  --mode snapshot \
  --compress zstd \
  --notes "Test backup after rebuild" \
  --remove 0

# Check task status
pvesh get /cluster/tasks --typefilter backup --limit 5
```

---

### 2.5 DNS: Pi-hole (CT 107)

**Previously installed by:** community-scripts Pi-hole installer  
**Rebuild method:** Official [Pi-hole GitHub](https://github.com/pi-hole/pi-hole) installer script OR manual tarball

#### LXC Container

```bash
CT_ID=107
pct create $CT_ID local:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst \
  --hostname pihole \
  --storage local-zfs \
  --rootfs local-zfs:8 \
  --cores 1 \
  --memory 512 \
  --swap 256 \
  --net0 name=eth0,bridge=vmbr0,ip=10.2.7.2/24,gw=10.2.7.1 \
  --onboot 1 \
  --startup order=1,up=10 \
  --unprivileged 1 \
  --ostype debian \
  --tags dns,dhcp,pihole \
  --nameserver 1.1.1.1 \
  --searchdomain local.lan

pct start $CT_ID
```

#### Install Pi-hole (From GitHub, No Wrapper)

**The official Pi-hole installer is a script** — it's the vendor's own script from `https://github.com/pi-hole/pi-hole`, not a third-party wrapper. This is the accepted upstream installation method. The alternative is manual tarball extraction:

**Option A — Vendor script (recommended, still upstream):**

```bash
pct exec $CT_ID -- bash -c '
  # Official Pi-hole install from GitHub
  git clone --depth 1 https://github.com/pi-hole/pi-hole.git /tmp/pi-hole
  cd /tmp/pi-hole
  bash automated_install/install.sh \
    --unattended \
    --disable-cloudflared \
    --disable-stubby \
    --disable-doctor
    
  # Set web password
  pihole setpassword '"'piholeisking'"'
'
```

**Option B — Manual binary install (fully script-free):**

```bash
pct exec $CT_ID -- bash -c '
  # Install dependencies manually
  apt-get install -y \
    lighttpd \
    php-cgi \
    php-sqlite3 \
    sqlite3 \
    dnsmasq \
    curl \
    git

  # Clone Pi-hole core, web, FTL from GitHub
  git clone --depth 1 https://github.com/pi-hole/pi-hole.git /etc/.pihole
  git clone --depth 1 https://github.com/pi-hole/AdminLTE.git /var/www/html/admin
  git clone --depth 1 https://github.com/pi-hole/FTL.git /tmp/ftl

  # Install FTL binary manually from GitHub releases
  curl -sL https://github.com/pi-hole/FTL/releases/latest/download/pihole-FTL-linux-x86_64 \
    -o /usr/bin/pihole-FTL
  chmod 755 /usr/bin/pihole-FTL

  # Create pihole script
  ln -sf /etc/.pihole/advanced/Scripts/pihole /usr/local/bin/pihole

  # Configure lighttpd for PHP
  lighttpd-enable-mod fastcgi fastcgi-php 2>/dev/null || true

  # Run gravity (download adlists + build databases)
  pihole -g
'
```

#### Post-Install: Restore Config

```bash
# Restore config from extraction backup
pct push $CT_ID ~/rebuild-extract/pihole/pihole-config.tar.gz /tmp/
pct exec $CT_ID -- tar xzf /tmp/pihole-config.tar.gz -C /

# Restore gravity DB (adlists, DHCP leases, allow/deny lists)
pct push $CT_ID ~/rebuild-extract/pihole/gravity-dump.sql /tmp/
pct exec $CT_ID -- sqlite3 /etc/pihole/gravity.db < /tmp/gravity-dump.sql

# Restart services
pct exec $CT_ID -- systemctl restart pihole-FTL lighttpd

# Verify
pct exec $CT_ID -- pihole status
```

#### Raw Systemd Units (No Wrappers)

```bash
# FTL service — check what was installed
pct exec $CT_ID -- cat /etc/systemd/system/pihole-FTL.service
# Expected: starts /usr/bin/pihole-FTL as pihole user

# If missing, create it:
pct exec $CT_ID -- bash -c '
cat > /etc/systemd/system/pihole-FTL.service << EOF
[Unit]
Description=Pi-hole FTL (DNS + DHCP)
After=network.target

[Service]
Type=simple
User=pihole
Group=pihole
ExecStart=/usr/bin/pihole-FTL
ExecReload=/usr/bin/pihole-FTL -reload
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now pihole-FTL
'
```

---

### 2.6 Agent: Hermes (CT 100)

**Previously installed by:** Community script container template  
**Rebuild method:** Manual `pct create` + upstream Hermes Agent install from [GitHub](https://github.com/nousresearch/hermes-agent)

#### LXC Container

```bash
CT_ID=100
pct create $CT_ID local:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst \
  --hostname hermes \
  --storage local-zfs \
  --rootfs local-zfs:20 \
  --cores 2 \
  --memory 4096 \
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

pct start $CT_ID
```

#### Install Hermes Agent (From GitHub, No Wrappers)

```bash
pct exec $CT_ID -- bash -c '
  # Install system dependencies
  apt update && apt upgrade -y
  apt install -y curl sudo git python3 python3-pip python3-venv openssh-server

  # Create user
  useradd -m -s /bin/bash hermes
  usermod -aG sudo hermes

  # Install Hermes Agent
  su - hermes -c "
    curl -fsSL https://raw.githubusercontent.com/nousresearch/hermes-agent/main/install.sh | bash
  "

  # Add to path
  echo '"'"'export PATH=\"$PATH:$HOME/.hermes/bin\"'"'"' >> /home/hermes/.bashrc
'
```

#### Config Restore

```bash
# Restore previously extracted config
pct push $CT_ID ~/rebuild-extract/hermes-config.tar.gz /home/hermes/
pct exec $CT_ID -- tar xzf /home/hermes/config.tar.gz -C /home/hermes/
pct exec $CT_ID -- chown -R hermes:hermes /home/hermes/.hermes /home/hermes/.ssh
```

---

### 2.7 Monitoring: Docker Stack (CT 106)

**Already manually built** — no script migration needed. The existing `docker-compose.yml`, `prometheus.yml`, `alertmanager.yml`, `blackbox.yml`, `nut_exporter.py`, and `webhook/receiver.py` are all hand-written files. Just:

- Recreate the LXC container
- Install Docker from the official repo
- Copy and reinstall the compose stack

#### LXC Container

```bash
CT_ID=106
pct create $CT_ID local:vztmpl/ubuntu-24.04-standard_24.04-2_amd64.tar.zst \
  --hostname grafana \
  --storage local-zfs \
  --rootfs local-zfs:32 \
  --cores 4 \
  --memory 6144 \
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

pct start $CT_ID
```

#### Install Docker (From Official Repo, No Convenience Script)

```bash
pct exec $CT_ID -- bash -c '
  # Remove any old Docker installations
  for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
    apt-get remove -y $pkg 2>/dev/null || true
  done

  # Install pre-reqs
  apt-get update
  apt-get install -y ca-certificates curl gnupg

  # Add Docker official GPG key (from download.docker.com, NOT get.docker.com convenience script)
  install -m 0755 -d /etc/apt/keyrings
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
    -o /etc/apt/keyrings/docker.asc
  chmod a+r /etc/apt/keyrings/docker.asc

  # Add Docker APT repo
  echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
    https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
    | tee /etc/apt/sources.list.d/docker.list > /dev/null

  apt-get update

  # Install Docker engine + compose plugin (official packages, no convenience script)
  apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

  # Add root to docker group
  usermod -aG docker root
'
```

#### Deploy the Monitoring Stack

```bash
# Create directory structure on CT 106
pct exec $CT_ID -- mkdir -p /opt/monitoring/{webhook,dashboards,provisioning/{datasources,dashboards}}

# Copy compose file and configs from backup
pct push $CT_ID ~/rebuild-extract/ct106-docker-compose.yml /opt/monitoring/docker-compose.yml
pct push $CT_ID ~/rebuild-extract/ct106-prometheus.yml /opt/monitoring/prometheus.yml
# ... push all config files from extraction ...

# Start the stack
pct exec $CT_ID -- bash -c '
  cd /opt/monitoring
  docker compose pull
  docker compose up -d
  docker compose ps
'
```

#### Install node_exporter (Raw Binary + Systemd)

```bash
pct exec $CT_ID -- bash -c '
  VERSION=$(curl -sL https://api.github.com/repos/prometheus/node_exporter/releases/latest \
    | grep tag_name | cut -d'"'"'"'"'"'" -f4 | sed "s/^v//")
  
  # Download from official GitHub releases
  curl -sL https://github.com/prometheus/node_exporter/releases/download/v${VERSION}/node_exporter-${VERSION}.linux-amd64.tar.gz \
    -o /tmp/node_exporter.tar.gz
  
  tar xzf /tmp/node_exporter.tar.gz -C /tmp/
  cp /tmp/node_exporter-*/node_exporter /usr/local/bin/
  rm -rf /tmp/node_exporter-*

  # Create systemd unit (raw, no script)
  cat > /etc/systemd/system/node_exporter.service << EOF
[Unit]
Description=Prometheus Node Exporter
Documentation=https://prometheus.io/docs/guides/node-exporter/
After=network.target

[Service]
Type=simple
User=nobody
Group=nogroup
ExecStart=/usr/local/bin/node_exporter \
  --collector.filesystem.mount-points-exclude="^/(dev|proc|sys|run)($$|/)" \
  --collector.netclass.ignored-devices="^(veth|docker|br-|tun|tap).*" \
  --web.listen-address=:9100
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

  systemctl daemon-reload
  systemctl enable --now node_exporter
'
```

#### NUT Exporter (Custom Python Script, Raw)

The NUT exporter is a hand-written Python script — no migration needed. But document the systemd approach (alternative to Docker):

```bash
pct exec $CT_ID -- bash -c '
  cat > /opt/monitoring/nut_exporter.py << NUTPY
#!/usr/bin/env python3
\"\"\"NUT Prometheus Exporter — raw Python, no framework\"\"\"
import http.server, socket, os, json

NUT_HOST = os.environ.get("NUT_HOST", "10.2.7.64")
NUT_PORT = int(os.environ.get("NUT_PORT", 3493))
NUT_PASS = os.environ.get("NUT_PASSWORD", "upsmonitor")
LISTEN_PORT = int(os.environ.get("LISTEN_PORT", 9999))

METRICS = [
    ("ups.status",     "nut_ups_status",     "1=OL,2=OB,3=OB+LB,4=OL+LB", "gauge"),
    ("battery.charge", "nut_battery_charge",  "percent",                    "gauge"),
    ("battery.runtime","nut_battery_runtime", "seconds",                    "gauge"),
    ("ups.load",       "nut_ups_load",        "percent",                    "gauge"),
    ("input.voltage",  "nut_input_voltage",   "volts",                      "gauge"),
    ("output.voltage", "nut_output_voltage",  "volts",                      "gauge"),
]

def query_nut(var):
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.settimeout(3)
    s.connect((NUT_HOST, NUT_PORT))
    data = s.recv(1024)
    s.send(f"USERNAME monitor {NUT_PASS}\\n".encode())
    data = s.recv(1024)
    s.send(f"GET VAR cyberpower {var}\\n".encode())
    data = s.recv(1024).decode().strip()
    s.close()
    return data.split(" = ")[-1].strip('"')

class Handler(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header("Content-Type", "text/plain")
        self.end_headers()
        lines = []
        for nut_var, prom_name, help_text, _ in METRICS:
            try:
                val = query_nut(nut_var)
                lines.append(f"# HELP {prom_name} {help_text}")
                lines.append(f"{prom_name} {val}")
            except Exception as e:
                lines.append(f"# {prom_name}_error {e}".replace("\\n", " "))
        self.wfile.write("\\n".join(lines).encode())

if __name__ == "__main__":
    server = http.server.HTTPServer(("0.0.0.0", LISTEN_PORT), Handler)
    server.serve_forever()
NUTPY
  chmod +x /opt/monitoring/nut_exporter.py
'
```

---

### 2.8 Wazuh SIEM (CT 105)

**Already manually installed** from official APT repo. No migration needed — just the standard install from the Wazuh docs.

```bash
CT_ID=105
SIZE=30  # Wazuh logs grow fast
CORES=4
MEMORY=4096

pct create $CT_ID local:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst \
  --hostname wazuh \
  --storage local-zfs \
  --rootfs local-zfs:${SIZE} \
  --cores $CORES \
  --memory $MEMORY \
  --swap 1024 \
  --net0 name=eth0,bridge=vmbr0,ip=10.2.7.110/24,gw=10.2.7.1 \
  --onboot 1 \
  --startup order=3,up=60 \
  --unprivileged 1 \
  --ostype debian \
  --tags siem,soc,wazuh \
  --nameserver 10.2.7.2 \
  --searchdomain local.lan

pct start $CT_ID
```

#### Install from Official APT Repo

```bash
pct exec $CT_ID -- bash -c '
  # Wazuh APT repo — no third-party scripts
  curl -s https://packages.wazuh.com/4.x/apt/GPG-KEY-WAZUH \
    | gpg --no-tty --yes --dearmor -o /usr/share/keyrings/wazuh.gpg

  echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" \
    > /etc/apt/sources.list.d/wazuh.list

  apt-get update
  apt-get install -y wazuh-manager

  systemctl enable --now wazuh-manager
'
```

#### Install Wazuh Dashboard + Indexer

```bash
pct exec $CT_ID -- bash -c '
  # Install all-in-one (uses Wazuh\'s built-in configuration tool)
  apt-get install -y wazuh-indexer wazuh-dashboard

  # Configure indexer networking
  cat > /etc/wazuh-indexer/opensearch.yml << EOF
network.host:
  - "127.0.0.1"
  - "10.2.7.110"
EOF

  # Initialize cluster (no helper script — raw Wazuh tool)
  bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh \
    -u admin -p "W4zzuhP0wer."

  systemctl restart wazuh-indexer
  systemctl restart wazuh-dashboard
  systemctl restart wazuh-manager
'
```

#### Discord Alert Integration (Raw Python Script)

See the Wazuh skill for the exact integration script. No helper needed:

```bash
pct exec $CT_ID -- bash -c '
  # Create custom Discord integration (raw Python, no framework)
  cat > /var/ossec/integrations/custom-discord.py << PYEOF
#!/usr/bin/env python3
"""Wazuh → Discord Alert Integration — raw Python, no pip dependencies"""
import json, sys, urllib.request

HOOK_URL = "https://discord.com/api/webhooks/..."

def main():
    if len(sys.argv) < 2:
        sys.exit(1)
    with open(sys.argv[1]) as f:
        alert = json.load(f)

    level = alert.get("rule", {}).get("level", 0)
    color = {3: 0xFFFF00, 7: 0xFFA500, 10: 0xFF0000, 14: 0x800000}.get(level, 0x808080)

    embed = {
        "embeds": [{
            "title": alert.get("rule", {}).get("description", "Wazuh Alert"),
            "color": color,
            "fields": [
                {"name": "Agent", "value": alert.get("agent", {}).get("name", "N/A"), "inline": True},
                {"name": "Rule ID", "value": str(alert.get("rule", {}).get("id", "N/A")), "inline": True},
                {"name": "Level", "value": str(level), "inline": True},
                {"name": "Description", "value": alert.get("full_log", "N/A")[:1024]},
            ],
            "timestamp": alert.get("timestamp", "")
        }]
    }
    req = urllib.request.Request(HOOK_URL,
        data=json.dumps(embed).encode(),
        headers={"Content-Type": "application/json"})
    urllib.request.urlopen(req)

if __name__ == "__main__":
    main()
PYEOF
  chmod 755 /var/ossec/integrations/custom-discord.py
  chown root:wazuh /var/ossec/integrations/custom-discord.py
'
```

---

### 2.9 SSO & Dashboard: Homarr (CT 103)

**Previously installed by:** community-scripts Homarr installer  
**Rebuild method:** Official [Homarr Docker image](https://github.com/ajnart/homarr)

```bash
CT_ID=103
pct create $CT_ID local:vztmpl/ubuntu-24.04-standard_24.04-2_amd64.tar.zst \
  --hostname homarr \
  --storage local-zfs \
  --rootfs local-zfs:8 \
  --cores 2 \
  --memory 2048 \
  --swap 512 \
  --net0 name=eth0,bridge=vmbr0,ip=10.2.7.105/24,gw=10.2.7.1 \
  --onboot 1 \
  --startup order=3,up=30 \
  --features keyctl=1,nesting=1 \
  --unprivileged 1 \
  --ostype ubuntu \
  --tags dashboard,homarr \
  --nameserver 10.2.7.2 \
  --searchdomain local.lan

pct start $CT_ID

# Install Docker from official repo (same procedure as CT 106)
# Then deploy Homarr via docker-compose:
pct exec $CT_ID -- bash -c '
  mkdir -p /opt/homarr
  cat > /opt/homarr/docker-compose.yml << EOF
services:
  homarr:
    image: ghcr.io/ajnart/homarr:latest
    container_name: homarr
    security_opt:
      - apparmor:unconfined
    ports:
      - 7575:7575
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - homarr_data:/app/data
    restart: unless-stopped

volumes:
  homarr_data:
EOF

  cd /opt/homarr && docker compose up -d
'
```

#### Restore Homarr Config

```bash
# Restore SQLite DB (dashboard layout)
HOMARR_MOUNT=$(pct exec $CT_ID -- docker volume inspect homarr_data --format '{{.Mountpoint}}')
# The DB is inside the volume — use docker cp to restore
pct exec $CT_ID -- docker cp /tmp/homarr-dump.sql homarr:/tmp/ 2>/dev/null || true
# Or mount the backup DB directly into the volume
```

---

### 2.10 Photos: Immich (CT 101)

**Previously installed by:** community-scripts Immich installer  
**Rebuild method:** Official [Immich GitHub](https://github.com/immich-app/immich)

```bash
CT_ID=101
pct create $CT_ID local:vztmpl/ubuntu-24.04-standard_24.04-2_amd64.tar.zst \
  --hostname immich \
  --storage local-zfs \
  --rootfs local-zfs:30 \
  --cores 2 \
  --memory 4096 \
  --swap 1024 \
  --net0 name=eth0,bridge=vmbr0,ip=10.2.7.44/24,gw=10.2.7.1 \
  --onboot 1 \
  --startup order=3,up=60 \
  --features keyctl=1,nesting=1 \
  --unprivileged 1 \
  --ostype ubuntu \
  --tags photos,immich \
  --nameserver 10.2.7.2 \
  --searchdomain local.lan

pct start $CT_ID
```

#### Install Immich (From GitHub, No Wrappers)

```bash
pct exec $CT_ID -- bash -c '
  # Install Docker from official repo (same as CT 106)

  # Clone Immich from official GitHub
  git clone https://github.com/immich-app/immich.git /opt/immich
  cd /opt/immich

  # Copy example env and customize
  cp example.env .env

  # Edit .env with your values
  sed -i "s/DB_PASSWORD=.*/DB_PASSWORD=<your-password>/" .env
  sed -i "s/DB_USERNAME=.*/DB_USERNAME=immich/" .env
  sed -i "s/DB_DATABASE_NAME=.*/DB_DATABASE_NAME=immich/" .env

  # Start Immich stack
  docker compose -f docker-compose.yml up -d
'
```

---

### 2.11 Storage: Nextcloud (CT 104)

**Previously installed by:** community-scripts Nextcloud installer (Apache-based)  
**Rebuild method:** Official Nextcloud tarball from [nextcloud.com](https://nextcloud.com/install) or Docker image

#### Option A: Apache/Tarball (Matches Current Setup)

```bash
CT_ID=104
pct create $CT_ID local:vztmpl/ubuntu-24.04-standard_24.04-2_amd64.tar.zst \
  --hostname nextcloud \
  --storage local-zfs \
  --rootfs local-zfs:30 \
  --cores 2 \
  --memory 4096 \
  --swap 1024 \
  --net0 name=eth0,bridge=vmbr0,ip=10.2.7.99/24,gw=10.2.7.1 \
  --onboot 1 \
  --startup order=3,up=60 \
  --features keyctl=1,nesting=1 \
  --unprivileged 1 \
  --ostype ubuntu \
  --tags cloud,nextcloud \
  --nameserver 10.2.7.2 \
  --searchdomain local.lan

pct start $CT_ID

pct exec $CT_ID -- bash -c '
  # Install LAMP stack (raw apt, no scripts)
  apt install -y apache2 mariadb-server libapache2-mod-php php-{gd,mysql,curl,xml,mbstring,zip,intl,imagick,bcmath,gmp,imap,ldap,smbclient}

  # Download Nextcloud from official source (no wrappers)
  VERSION="29.0.4"
  wget -q https://download.nextcloud.com/server/releases/nextcloud-${VERSION}.tar.bz2
  tar xjf nextcloud-${VERSION}.tar.bz2 -C /var/www/
  chown -R www-data:www-data /var/www/nextcloud

  # Configure Apache
  cat > /etc/apache2/sites-available/nextcloud.conf << EOF
Alias / "/var/www/nextcloud/"
<Directory /var/www/nextcloud/>
    Options +FollowSymlinks
    AllowOverride All
    Require all granted
</Directory>
EOF

  a2ensite nextcloud
  a2enmod rewrite headers env dir mime
  systemctl restart apache2

  # Create database
  mysql -u root << SQL
CREATE DATABASE IF NOT EXISTS nextcloud;
CREATE USER IF NOT EXISTS nextcloud@"localhost" IDENTIFIED BY "<db-password>";
GRANT ALL PRIVILEGES ON nextcloud.* TO nextcloud@"localhost";
FLUSH PRIVILEGES;
SQL

  # Complete install via occ (Nextcloud\'s own CLI tool)
  cd /var/www/nextcloud
  sudo -u www-data php occ maintenance:install \
    --database mysql \
    --database-name nextcloud \
    --database-user nextcloud \
    --database-pass "<db-password>" \
    --admin-user admin \
    --admin-pass "<admin-password>" \
    --data-dir /var/lib/nextcloud/data
'
```

#### Option B: Docker (Cleaner, Easier Backup)

```bash
# docker-compose.yml for Nextcloud + MariaDB:
pct exec $CT_ID -- bash -c '
  mkdir -p /opt/nextcloud
  cat > /opt/nextcloud/docker-compose.yml << EOF
services:
  db:
    image: mariadb:11
    container_name: nextcloud-db
    environment:
      - MYSQL_ROOT_PASSWORD=<root-pw>
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_PASSWORD=<user-pw>
    volumes:
      - nextcloud_db:/var/lib/mysql
    restart: unless-stopped

  app:
    image: nextcloud:29
    container_name: nextcloud
    ports:
      - 80:80
    volumes:
      - nextcloud_data:/var/www/html
    environment:
      - MYSQL_HOST=db
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_PASSWORD=<user-pw>
    restart: unless-stopped

volumes:
  nextcloud_db:
  nextcloud_data:
EOF

  cd /opt/nextcloud && docker compose up -d
'
```

---

### 2.12 Network Monitor: PiAlert (CT 102)

**Previously installed by:** community-scripts PiAlert installer  
**Rebuild method:** Official [Pi.Alert GitHub](https://github.com/leiweibau/Pi.Alert)

```bash
CT_ID=102
pct create $CT_ID local:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst \
  --hostname pialert \
  --storage local-zfs \
  --rootfs local-zfs:5 \
  --cores 2 \
  --memory 2048 \
  --swap 512 \
  --net0 name=eth0,bridge=vmbr0,ip=10.2.7.77/24,gw=10.2.7.1 \
  --onboot 1 \
  --startup order=3,up=30 \
  --unprivileged 1 \
  --ostype debian \
  --tags network,monitoring,pialert \
  --nameserver 10.2.7.2 \
  --searchdomain local.lan

pct start $CT_ID

pct exec $CT_ID -- bash -c '
  # Install system dependencies
  apt update && apt install -y \
    python3 python3-pip python3-venv \
    arp-scan net-tools nginx lighttpd 2>/dev/null || true

  # Clone Pi.Alert from official GitHub (git clone, not script)
  git clone https://github.com/leiweibau/Pi.Alert.git /opt/pialert
  cd /opt/pialert

  # Install Python deps
  pip3 install -r requirements.txt 2>/dev/null || pip3 install flask pyyaml

  # Create systemd unit (raw, no wrapper)
  cat > /etc/systemd/system/pialert.service << EOF
[Unit]
Description=Pi.Alert Network Monitor
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/pialert
ExecStart=/usr/bin/python3 /opt/pialert/back/pialert.py
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

  # Create Python scan cron (raw, no wrapper)
  echo "*/5 * * * * root cd /opt/pialert && /usr/bin/python3 back/pialert.py --scan" \
    > /etc/cron.d/pialert

  systemctl daemon-reload
  systemctl enable --now pialert
'
```

#### Restore PiAlert Config

```bash
pct push $CT_ID ~/rebuild-extract/pialert/pialert-config.tar.gz /tmp/
pct exec $CT_ID -- tar xzf /tmp/pialert-config.tar.gz -C /
pct exec $CT_ID -- systemctl restart pialert
```

---

### 2.13 Local LLM: Ollama (CT 108)

**Previously installed by:** community-scripts Ollama installer  
**Rebuild method:** Official [Ollama releases](https://github.com/ollama/ollama/releases) — direct binary, no script

```bash
CT_ID=108
pct create $CT_ID local:vztmpl/ubuntu-24.04-standard_24.04-2_amd64.tar.zst \
  --hostname ollama \
  --storage local-zfs \
  --rootfs local-zfs:40 \
  --cores 4 \
  --memory 8192 \
  --swap 2048 \
  --net0 name=eth0,bridge=vmbr0,ip=10.2.7.167/24,gw=10.2.7.1 \
  --onboot 1 \
  --startup order=3,up=30 \
  --features keyctl=1,nesting=1 \
  --unprivileged 0 \          # Ollama needs privileged for GPU passthrough
  --ostype ubuntu \
  --tags ai,ollama,llm,local \
  --nameserver 10.2.7.2 \
  --searchdomain local.lan \
  --mp0 /dev/dri,mp=/dev/dri   # GPU passthrough for Intel iGPU

pct start $CT_ID
```

#### Install Ollama (Direct Binary from GitHub Releases — No Script)

```bash
pct exec $CT_ID -- bash -c '
  # Install system dependencies
  apt update && apt install -y curl pciutils

  # Download Ollama binary directly from GitHub releases (no install.sh)
  curl -sL https://github.com/ollama/ollama/releases/latest/download/ollama-linux-amd64.tgz \
    -o /tmp/ollama.tgz

  # Extract to /usr (creates /usr/bin/ollama + lib)
  tar -C /usr -xzf /tmp/ollama.tgz
  rm /tmp/ollama.tgz

  # Create ollama user
  useradd -r -s /bin/false -m -d /usr/share/ollama ollama

  # Create systemd unit (raw, no wrapper)
  cat > /etc/systemd/system/ollama.service << EOF
[Unit]
Description=Ollama LLM Service
Documentation=https://ollama.com
After=network.target

[Service]
Type=simple
User=ollama
Group=ollama
Environment="OLLAMA_HOST=0.0.0.0"
Environment="OLLAMA_MODELS=/usr/share/ollama/.ollama/models"
ExecStart=/usr/bin/ollama serve
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

  # Verify GPU access (if iGPU passthrough is configured)
  ls -la /dev/dri/* 2>/dev/null || echo "No /dev/dri — GPU passthrough not configured"

  # If GPU is present, add ollama to video/render groups
  getent group video 2>/dev/null && usermod -aG video ollama
  getent group render 2>/dev/null && usermod -aG render ollama

  systemctl daemon-reload
  systemctl enable --now ollama

  # Verify
  sleep 3
  curl -s http://localhost:11434/api/tags || echo "Ollama not responding yet — check systemctl status ollama"
'
```

#### Pull Models (From Extraction Manifest)

```bash
pct exec $CT_ID -- bash -c '
  # Pull models that were in use (from extraction backup)
  ollama pull qwen2.5:3b
  ollama pull qwen2.5:7b
  ollama pull tinyllama:1.1b

  # Verify
  ollama list
'
```

---

## Appendix: Quick Reference — Script to No-Script Migration Map

| Original Script Path | Raw Alternative | What Changes |
|---------------------|----------------|--------------|
| `bash -c "$(wget -qLO - https://community-scripts.net/install)"` | `pct create` + `git clone` from upstream | No curl|bash. Each layer is explicit. |
| `pve_source` (helper repo toggle) | `rm` enterprise.list + `echo` no-subscription to sources | Two explicit commands vs one wrapper |
| `pve_postinstall` (nag removal) | Single `sed` on `Subscription.pm` | Transparent, reverts on package update |
| `Docker convenience script` | Docker APT repo via `download.docker.com` | GPG keys from official source, not get.docker.com |
| `Pi-hole automated install` | `git clone` upstream + manual pihole-FTL binary | Same upstream code, no pipe-to-bash |
| `Ollama install.sh` | `curl` binary tarball from GitHub releases + hand-written systemd unit | No script, explicit binary placement |
| `Community script LXC template` | `pct create` with explicit flags | Every feature flag documented inline |
| `Wazuh all-in-one installer` | `apt-get install` from official Wazuh repo + manual config | The official DEB is the same either way |
| `Nextcloud install.sh` | `wget` tarball from `download.nextcloud.com` + `occ` CLI | Same result, explicit steps |
| `Immich install script` | `git clone` from GitHub + `docker compose up -d` | Exactly what the script does, just uncoupled |

---

> **Document version:** 1.0 — Created 2026-05-25  
> **Related:** `00-pve-pbs-lifecycle-playbook.md` (operational commands), `DR-IR-PLAYBOOK.md` (emergency scenarios)  
> **Edge case:** If community-scripts.net ever goes offline or changes, this blueprint is the recovery path. No dependency on a third-party installer.
