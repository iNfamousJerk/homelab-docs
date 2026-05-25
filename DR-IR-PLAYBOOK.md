# 🛡️ Disaster Recovery & Incident Response Playbook

**Environment:** Proxmox VE / Proxmox Backup Server / Docker / Wazuh HIDS  
**Author:** Hermes Agent (iNfamousJerk)

---

## Table of Contents

1. [Total Node Failure (Hardware Loss)](#1-total-node-failure-hardware-loss)
2. [OS Drive Failure (Data Disks Intact)](#2-os-drive-failure-data-disks-intact)
3. [Docker Recovery](#3-docker-recovery)
4. [Wazuh Intrusion Monitoring](#4-wazuh-intrusion-monitoring)
5. [Active Breach Incident Response](#5-active-breach-incident-response)

---

## 1. Total Node Failure (Hardware Loss)

**Scenario:** PVE chassis dead. Replacement hardware arrives. Need to restore everything from PBS.

### Phase A — Install Fresh PVE

- [ ] Install PVE 9.x on replacement hardware (same hostname, IP)
- [ ] Configure network bridges to match:

```bash
# Network config: /etc/network/interfaces
# Current layout (recreate on new hardware):
auto lo
iface lo inet loopback

auto vmbr0
iface vmbr0 inet static
    address 10.2.7.64/24
    gateway 10.2.7.1
    bridge-ports enp0s3   # or whatever the new NIC is
    bridge-stp off
    bridge-fd 0

# No other bridges — VLANs handled via vmbr0 + VLAN tags
```

- [ ] Apply config: `systemctl restart networking`
- [ ] Test upstream: `ping 10.2.7.1`

### Phase B — Connect PBS

- [ ] Add PBS storage in PVE WebUI → Datacenter → Storage → Add → Proxmox Backup Server
- [ ] Or via CLI:

```bash
pvesm add pbs pbs \
  --server 10.2.7.65 \
  --datastore main \
  --username root@pam \
  --password '[REDACTED]' \
  --fingerprint $(ssh root@10.2.7.65 "proxmox-backup-manager cert info | grep Fingerprint | awk '{print \$NF}'")
```

### Phase C — Restore Encrypted Backups

**Prerequisite:** You backed up the encryption keys. If not, **backups are unrecoverable**.

```bash
# Decrypt key file location on PBS:
# /root/proxmox-backup-encryption-key.enc
# Passphrase stored in CREDENTIALS.md

# Restore a container from encrypted backup:
pct restore <CT_ID> pbs:backup/<CT_ID>/<BACKUP_ID> \
  --keyfile /path/to/encryption-key.enc \
  --key-passphrase 'your-passphrase'
```

### Phase D — Staggered Boot Order

Boot order determined by dependency chain. Wait for each tier to fully initialize before starting the next.

| Tier | CTs | Why This Order |
|------|-----|----------------|
| **1 — Infrastructure** | 107 (Pi-hole, DNS) | Every service needs DNS |
| **2 — Logging** | 105 (Wazuh) | Collect logs from everything below |
| **3 — Monitoring** | 106 (Grafana/Docker) | Watch recovery progress |
| **4 — Storage** | 101 (Immich), 104 (Nextcloud) | Data services need DB + storage ready |
| **5 — Supporting** | 102 (PiAlert), 103 (Homarr) | Network-aware services |
| **6 — AI** | 100 (Hermes) | Last — no dependencies |

```bash
# Restore & start in order:
pct restore 107 pbs:backup/107/latest --storage local
pct start 107
sleep 30  # Wait for Pi-hole DNS to be live

pct restore 105 pbs:backup/105/latest --storage local
pct start 105
sleep 20

# ... repeat for remaining CTs
```

---

## 2. OS Drive Failure (Data Disks Intact)

**Scenario:** PVE boot SSD dies. Data ZFS pools / LVM-thin are on separate disks that survived.

### Phase A — Reinstall PVE on New Boot Drive

- [ ] Install PVE 9.x on fresh boot SSD
- [ ] **DO NOT** select existing data disks during partitioning
- [ ] After install, do NOT create any local storage on data disks

### Phase B — Import Existing ZFS Pools

```bash
# Scan for available pools:
zpool import -d /dev/disk/by-id/

# Import by pool name (find names from scan above):
zpool import -f backup   # PBS ZFS pool if on same node
zpool import -f rpool    # Common Proxmox pool name

# Verify:
zpool status
zfs list
```

### Phase C — Scan for LVM Volume Groups

```bash
# Scan and activate LVM:
vgscan
vgchange -ay

# List logical volumes:
lvdisplay

# If LVM-thin pool found, check with:
pvesm status
```

### Phase D — Re-register VM/LXC Configs

Config files are stored on the local PVE filesystem (`/etc/pve/`). If you backed these up separately (recommended):

```bash
# Configs live on PBS as part of backups — restore via vzdump:
# List available backups:
pvesm list pbs

# Restore specific VMs/LXCs with --unique to avoid conflicts:
for ctid in 100 101 102 103 104 105 106 107; do
  pct restore $ctid pbs:backup/$ctid/latest --unique --storage local-zfs
done
```

**No config backup?** Recreate manually:

```bash
# Check backup contents for network config:
cd /tmp
proxmox-backup-client restore --ns <namespace> \
  pve/ct/<CT_ID>/<YYYY-MM-DD_HH:MM:SS> /tmp/restore.pxar

# Inside the restore, check /etc/network/interfaces for VLAN configs
```

**Pro tip:** Backup `/etc/pve/` directory regularly as a separate task:

```bash
# Add to PVE daily cron:
tar czf /backup/pve-configs-$(date +%F).tar.gz /etc/pve/
```

---

## 3. Docker Recovery

**Scenario:** CT 106 (monitoring) OS crashes or Docker daemon corrupt. Data volumes intact.

### Best Practices (Prevention)

- [ ] **All data in named/external volumes** — never in anonymous volumes
- [ ] **docker-compose.yml backed up** — already in `/opt/monitoring/docker-compose.yml`
- [ ] **.env files** stored alongside compose files (NOT inside containers)
- [ ] **Volume paths** use bind mounts under `/opt/monitoring/` or `/var/lib/docker/volumes/`
- [ ] **Grafana dashboards provisioned** — JSON files at `/opt/monitoring/dashboards/`
- [ ] **Grafana datasources provisioned** — YAML at `/opt/monitoring/provisioning/`

### Recovery Steps

```bash
# 1. Rebuild CT 106 from PBS backup (or fresh OS install)
pct restore 106 pbs:backup/106/latest

# 2. Verify Docker volumes are present (if using bind mounts):
ls -la /opt/monitoring/
ls -la /opt/monitoring/docker-compose.yml

# 3. If using named volumes, check they survived:
docker volume ls
docker volume inspect monitoring_grafana_data

# 4. Restore compose file if missing:
# From homelab-docs repo on Gitea:
git clone http://10.2.7.108:3002/InfamousJerk/homelab-docs.git
cp homelab-docs/docker-compose.yml /opt/monitoring/

# 5. Pull and restart:
cd /opt/monitoring
docker compose pull
docker compose up -d

# 6. Verify all services healthy:
docker compose ps
curl -s http://localhost:3000/api/health  # Grafana
curl -s http://localhost:9090/api/v1/targets | grep -c '"health":"up"'
```

### Volume Map (CT 106)

| Container | Volume Mount | Data |
|-----------|-------------|------|
| Grafana | `monitoring_grafana_data:/var/lib/grafana` | Dashboards, users, settings |
| Prometheus | Bind: `/opt/monitoring/prometheus/` | Configs only (data ephemeral) |
| Alertmanager | Bind: `/opt/monitoring/alertmanager/` | Configs only |
| nut-exporter | Stateless | No persistent data |
| pihole-exporter | Stateless | No persistent data |
| cAdvisor | Stateless | No persistent data |
| Uptime Kuma | `monitoring_uptime_kuma_data:/app/data` | Monitors, status pages |
| Gitea | `gitea_data:/data` | Repos, users, config |

---

## 4. Wazuh Intrusion Monitoring

### Agent Deployment

```bash
# PVE Host — install Wazuh agent directly:
ssh root@10.2.7.64
curl -s https://packages.wazuh.com/4.x/wazuh-install.sh | bash -s -- agent
/var/ossec/bin/manage_agents -a 10.2.7.110  # Manager IP

# CTs — install inside each container:
for ip in 10.2.7.44 10.2.7.77 10.2.7.105 10.2.7.99 \
         10.2.7.108 10.2.7.2 10.2.7.107; do
  ssh root@$ip "
    curl -s https://packages.wazuh.com/4.x/wazuh-install.sh | bash -s -- agent
  "
done

# Docker — use Wazuh container agent or syslog forwarding:
# Option A: Run Wazuh agent as Docker container
# Option B: Forward Docker logs to Wazuh manager via syslog
docker run -d --name wazuh-agent \
  -e WAZUH_MANAGER=10.2.7.110 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  wazuh/wazuh-agent:latest
```

### Critical Log Sources & Alerts

| Log Source | Path | What to Watch For |
|------------|------|-------------------|
| SSH auth | `/var/log/auth.log` | Failed login spikes, root SSH attempts |
| System | `/var/log/syslog` | Service crashes, OOM kills, weird cron jobs |
| PVE | `/var/log/pve/tasks/active` | Unauthorized API calls, VM creation |
| Docker | `docker logs <container>` | Container crashes, API abuse |
| NUT | `upsc ups@localhost` | UPS disconnected, battery critical |

### Custom Rules for Homelab

Create `/var/ossec/etc/rules/local_rules.xml` on Wazuh manager (CT 105):

```xml
<group name="homelab">
  <!-- Alert on >5 failed SSH attempts in 5 minutes -->
  <rule id="100001" level="7">
    <if_sid>5710</if_sid>
    <match>Failed password</match>
    <frequency>5</frequency>
    <timeframe>300</timeframe>
    <description>Brute-force SSH attempt detected</description>
  </rule>

  <!-- Alert on root SSH login -->
  <rule id="100002" level="5">
    <if_sid>5715</if_sid>
    <match>root</match>
    <description>Root SSH login</description>
  </rule>

  <!-- Alert on new cron jobs -->
  <rule id="100003" level="8">
    <if_sid>550</if_sid>
    <match>CRON</match>
    <description>Cron job modification detected</description>
  </rule>

  <!-- Alert on user account changes -->
  <rule id="100004" level="10">
    <if_sid>550</if_sid>
    <match>useradd|userdel|usermod|passwd</match>
    <description>User account change detected</description>
  </rule>
</group>
```

Apply: `systemctl restart wazuh-manager`

### Discord Alert Integration (Already Configured)

Level ≥5 alerts → Discord via custom integration at `/var/ossec/integrations/custom-discord.py`.  
Test with:

```bash
# Trigger a test alert:
ssh root@10.2.7.108 "logger -p auth.warn 'Test Wazuh alert from CT 106'"
```

---

## 5. Active Breach Incident Response

**Scenario:** Wazuh fires a level 10 alert — unauthorized root access or privilege escalation detected.

### Phase 1 — Containment (First 5 Minutes)

**Goal:** Stop the bleeding. Preserve evidence.

- [ ] **Isolate the compromised CT immediately** — network disconnect:

```bash
# Via PVE API (from Hermes):
curl -sk -b "PVEAuthCookie=$TICKET" \
  -X POST 'https://10.2.7.64:8006/api2/json/nodes/localhost/lxc/<CT_ID>/config' \
  -d 'net0=name=eth0,bridge=vmbr0,firewall=1,link_down=1'

# Or simpler — PVE WebUI → node → CT → Network → Disable NIC

# Via PVE CLI (if you can SSH to PVE):
pct set <CT_ID> --net0 name=eth0,bridge=vmbr0,link_down=1
```

- [ ] **Snapshot the CT for forensic analysis:**

```bash
vzdump <CT_ID> --mode snapshot --compress zstd --dumpdir /var/lib/vz/dump/
```

- [ ] **Don't power off** — RAM evidence is lost. Keep the CT running in isolation.

### Phase 2 — Investigation

**Goal:** Find the root cause. Check lateral movement.

```bash
# 1. Check recent auth attempts:
ssh root@<COMPROMISED_CT_IP> "
  last -50
  lastb -50
  cat /var/log/auth.log | grep -E '(Accepted|Failed)'
"

# 2. Check running processes (look for miners, shells):
ssh root@<COMPROMISED_CT_IP> "
  ps aux --forest
  netstat -tlnp
  lsof -i
"

# 3. Check for persistence mechanisms:
ssh root@<COMPROMISED_CT_IP> "
  crontab -l
  cat /etc/crontab
  ls -la /etc/cron.*/
  cat /root/.ssh/authorized_keys
  cat /home/*/.ssh/authorized_keys
  systemctl list-units --state=enabled | grep -v 'native'
"

# 4. Check PVE audit log:
ssh root@10.2.7.64 "journalctl -u pveproxy -n 100"
ssh root@10.2.7.64 "cat /var/log/pve/tasks/index | grep <COMPROMISED_CT_ID>"
```

**Lateral movement checklist:**

- [ ] Check Wazuh alerts for connections FROM the compromised CT to other CTs
- [ ] Scan SSH logs on ALL other CTs for login attempts matching the compromise time
- [ ] Check Docker container logs on CT 106 for unusual API calls
- [ ] Query Pi-hole DNS logs for suspicious outbound connections:

```bash
ssh root@10.2.7.2 "cat /etc/pihole/pihole.log | grep -E '$(date +%Y-%m-%d)' | grep -v '\.local\|\.lan\|pi-hole\.\|10\.0\.'"
```

### Phase 3 — Eradication

**Goal:** Remove the threat. Patch the hole.

```bash
# 1. Identify root cause:
#    - Weak/compromised password? → Force password change
#    - Unpatched vulnerability? → apt update && apt upgrade
#    - Exposed service? → Firewall that port
#    - Stolen SSH key? → Rotate ALL keys

# 2. Rebuild the compromised CT from clean PBS backup:
pct restore <CT_ID> pbs:backup/<CT_ID>/PRE_BREACH_BACKUP

# 3. Apply patches before reconnecting:
pct start <CT_ID>
ssh root@<CT_IP> "
  apt update && apt upgrade -y
  systemctl reboot
"

# 4. Change ALL passwords (since the compromise could be credential-based):
# Use passwords from CREDENTIALS.md
for ip in 10.2.7.44 10.2.7.77 10.2.7.105 10.2.7.99 \
         10.2.7.110 10.2.7.108 10.2.7.2; do
  ssh root@$ip "echo 'root:<NEW_PASSWORD>' | chpasswd"
done
```

### Phase 4 — Recovery

**Goal:** Safely restore services. Verify clean state.

```bash
# 1. Reconnect the CT (reverse the isolation):
pct set <CT_ID> --net0 name=eth0,bridge=vmbr0,link_down=0

# 2. Verify clean state:
ssh root@<CT_IP> "
  rkhunter --check --skip-keypress
  clamscan -r /home /root /tmp /var/tmp
"

# 3. Verify Wazuh agent reconnects:
ssh root@<CT_IP> "systemctl status wazuh-agent"

# 4. Monitor logs for 24 hours post-recovery:
ssh root@10.2.7.110 "tail -f /var/ossec/logs/alerts/alerts.json"

# 5. Create incident report (what, when, how, fix):
cat << EOF > /root/incident-$(date +%F).md
# Incident: $(date)
**Severity:** HIGH
**Detection:** Wazuh rule <ID>
**Compromised System:** CT <ID>
**Root Cause:** <summary>
**Containment:** Network disconnect at <time>
**Eradication:** PBS restore at <time>
**Post-Mortem:** <lessons learned>
EOF
```

### Post-Incident Hardening

| Action | Command |
|--------|---------|
| Disable root SSH login | `sed -i 's/PermitRootLogin yes/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config` |
| Enforce key-only auth | `sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config` |
| Rate-limit SSH | `apt install fail2ban` |
| Audit open ports | `netstat -tulpn \| grep LISTEN` |
| Patch everything | `apt update && apt upgrade -y` |

---

## Pre-Requisites Checklist

**These must be in place BEFORE a disaster strikes:**

- [ ] PBS backups running on schedule (verify with `pvesm list pbs`)
- [ ] Encryption keys backed up offline (USB drive / secondary machine)
- [ ] `/etc/pve/` configs backed up weekly
- [ ] docker-compose.yml in version control (Gitea)
- [ ] Grafana dashboards provisioned (not created ad-hoc in UI)
- [ ] Wazuh agents deployed on ALL CTs and PVE host
- [ ] Wazuh Discord alert integration tested
- [ ] UPS shutdown script tested (`systemctl status nut-monitor`)
- [ ] This playbook stored both locally AND printed (if PVE is down, you can't access files on it)

---

> **Last Updated:** 2026-05-25  
> **Maintainer:** Hermes Agent (iNfamousJerk)  
> **Next Review:** Quarterly, or after any major infrastructure change