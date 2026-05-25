# SOC Upgrade Project Plan: Flat Homelab → Enterprise Network

**Project Lead:** Anthony Piper (iNfamousJerk)  
**Audience:** Interview panel / Technical portfolio review  
**Target Architecture:** VLAN-segmented Proxmox SOC with SIEM, IAM, GitOps, and DMZ isolation  

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Phase 0 — Prerequisites & Dependencies](#2-phase-0--prerequisites--dependencies)
3. [Phase 1 — VLAN Segmentation & Firewall Hardening](#3-phase-1--vlan-segmentation--firewall-hardening)
4. [Phase 2 — Ingress, Reverse Proxy & IAM (MFA)](#4-phase-2--ingress-reverse-proxy--iam-mfa)
5. [Phase 3 — GitOps & Automation (Ansible + Gitea)](#5-phase-3--gitops--automation-ansible--gitea)
6. [Phase 4 — Advanced SIEM (Syslog + Custom Decoders)](#6-phase-4--advanced-siem-syslog--custom-decoders)
7. [Verification & Validation](#7-verification--validation)
8. [Interview Talking Points](#8-interview-talking-points)

---

## 1. Architecture Overview

### Current State (Flat)
```
10.2.7.0/24
├── .1   OPNsense (router/firewall)
├── .2   Pi-hole (DNS)
├── .64  PVE (hypervisor)
├── .44  Immich
├── .77  PiAlert
├── .99  Nextcloud
├── .105 Homarr
├── .107 Hermes Agent
├── .108 CT 106 (Grafana/Prometheus/Docker/Gitea/Uptime Kuma)
├── .110 Wazuh Manager
├── .65  PBS (Z230 — separate physical host)
```

### Target State (Segmented)
```
VLAN 10 — Mgmt (10.2.10.0/24)
├── .1   OPNsense (router/gateway for all VLANs)
├── .2   Pi-hole (DNS for all VLANs — shared service)
├── .64  PVE host
├── .65  PBS host
├── .254 Managed switch (management IP)

VLAN 20 — Core Services (10.2.20.0/24)
├── .44  Immich
├── .99  Nextcloud
├── .108 Gitea (internal Git)
├── .105 Homarr (dashboard)
├── .77  PiAlert

VLAN 30 — Security & Monitoring (10.2.30.0/24)
├── .110 Wazuh Manager + Indexer + Dashboard
├── .108 Grafana / Prometheus / Alertmanager
├── .108 Uptime Kuma

VLAN 40 — DMZ (10.2.40.0/24)
├── .X   Jellyfin (media streaming)
├── .X   Hermes Agent (AI agent — external-facing)
├── .X   Reverse Proxy (Nginx/Traefik)
```

### Traffic Flow Rules (Least Privilege)

| Rule | From | To | Protocol:Port | Purpose |
|------|------|----|---------------|---------|
| 1 | VLAN 10 (Mgmt) | Any | ANY | Admins manage everything |
| 2 | VLAN 20 (Core) | VLAN 30 (Sec) | TCP:1514-1516 | Wazuh agent → manager |
| 3 | VLAN 20 (Core) | VLAN 10 (Mgmt) | TCP:8006 (PVE) | Backup agents reach PVE API |
| 4 | VLAN 30 (Sec) | VLAN 10 (Mgmt) | ICMP, TCP:8006 | Monitoring probes PVE |
| 5 | VLAN 40 (DMZ) | VLAN 30 (Sec) | TCP:1514-1516 | DMZ agents → Wazuh |
| 6 | VLAN 40 (DMZ) | VLAN 10/20 | **DENY ALL** | DMZ cannot initiate to core/mgmt |
| 7 | VLAN 40 (DMZ) | WAN (Internet) | TCP:443,80,8096 | Jellyfin streaming outbound only |
| 8 | Any | VLAN 40 (DMZ) | TCP:443 (via RP), TCP:8096 | Inbound only through RP |

**Critical:** Rule 6 is the kill switch. VLAN 40 can only *respond* to initiated connections from VLAN 10/20 — never initiate. An attacker who compromises Jellyfin cannot touch PVE, Wazuh, or Nextcloud.

---

## 2. Phase 0 — Prerequisites & Dependencies

**Duration:** 1-2 days  
**Risk if skipped:** Broken routing during migration, data loss, unrecoverable CTs

### Checklist

- [ ] **Backup every CT to PBS** — verified, restorable
  ```bash
  for ctid in 100 101 102 103 104 105 106 107; do
    vzdump $ctid --storage pbs --mode snapshot --compress zstd
  done
  pvesm list pbs  # Verify all backups present
  ```

- [ ] **Export OPNsense current config** (rollback safety net)
  ```bash
  # OPNsense UI → System → Configuration → Backups → Download
  # Or SCP from firewall:
  ssh root@10.2.7.1 "cp /conf/config.xml /root/config-backup-flat.xml"
  ```

- [ ] **Document current CT IPs, MACs, and service ports** (this file serves as that)

- [ ] **Verify Wazuh agents on all CTs** — known baseline before subnet changes
  ```bash
  ssh root@10.2.7.110 "/var/ossec/bin/agent_control -lc"  # List all agents
  ```

- [ ] **Test PBS restore** of one non-critical CT to prove backup integrity
  ```bash
  pct restore 102 pbs:backup/102/latest --storage local --unique
  pct start 102-RESTORED  # Verify it boots
  pct stop 102-RESTORED && pct destroy 102-RESTORED  # Clean up
  ```

- [ ] **Patch all CTs** to ensure we're migrating from a known-good state
  ```bash
  for ip in 10.2.7.44 10.2.7.77 10.2.7.99 10.2.7.105 10.2.7.108; do
    ssh root@$ip "apt update && apt upgrade -y && apt autoremove -y"
  done
  ```

### Go/No-Go Criteria
- All 8 CTs backed up to PBS ✓
- OPNsense config export saved ✓
- Wazuh agents online on all CTs ✓

---

## 3. Phase 1 — VLAN Segmentation & Firewall Hardening

**Duration:** 3-4 days  
**Difficulty:** High — network will be disrupted during migration  
**Strategy:** One CT at a time, starting with lowest dependency (VLAN 40 — DMZ)

### Step 1: Create VLANs in OPNsense

| VLAN | Tag | Subnet | Purpose |
|------|-----|--------|---------|
| Management | 10 | 10.2.10.0/24 | PVE, PBS, switches |
| Core | 20 | 10.2.20.0/24 | Nextcloud, Immich, Gitea, Homarr |
| Security | 30 | 10.2.30.0/24 | Wazuh, Grafana, Prometheus |
| DMZ | 40 | 10.2.40.0/24 | Jellyfin, Hermes, Reverse Proxy |

```bash
# OPNsense steps — via WebUI:
# Interfaces → Assignments → Add VLAN (tag 10, parent interface)
# Interfaces → VLAN 10 → Enable → Static IPv4 10.2.10.1/24
# Repeat for VLANs 20, 30, 40

# Firewall → Rules → [VLAN] — create rules per table above

# Services → DHCP Server → [VLAN] — enable DHCP for each VLAN
# Set DNS to 10.2.7.2 (Pi-hole) for all VLANs
```

### Step 2: Configure PVE Network

```bash
# PVE — /etc/network/interfaces (add VLAN interfaces)
# Current: vmbr0 bridges directly to physical NIC
# New: vmbr0 remains as trunk, VLANs defined on top

# --- /etc/network/interfaces excerpt ---
auto vmbr0
iface vmbr0 inet manual
    bridge-ports enp0s3
    bridge-stp off
    bridge-fd 0

# VLAN 10 — Management (PVE host IP lives here)
auto vmbr0.10
iface vmbr0.10 inet static
    address 10.2.10.64/24
    gateway 10.2.10.1

# VLAN 20 — Core
auto vmbr0.20
iface vmbr0.20 inet manual

# VLAN 30 — Security
auto vmbr0.30
iface vmbr0.30 inet manual

# VLAN 40 — DMZ
auto vmbr0.40
iface vmbr0.40 inet manual
```

**Verification:**
```bash
systemctl restart networking
ip a | grep vmbr0
ping 10.2.10.1  # Should reach OPNsense gateway
```

### Step 3: Migrate CTs One at a Time

Each CT needs its network interface updated to the new VLAN.

```bash
# For each CT — example: Nextcloud (CT 104) → VLAN 20
pct stop 104
pct set 104 --net0 name=eth0,bridge=vmbr0,tag=20,ip=10.2.20.99/24,gw=10.2.20.1
pct start 104

# Verify from CT:
pct enter 104 -- ping 10.2.20.1
pct enter 104 -- curl -s http://10.2.20.1  # Should hit OPNsense
pct enter 104 -- ping 10.2.7.2               # Cross-VLAN DNS (Pi-hole)
```

**Migration Order (dependency-aware):**

| Step | CT | Target VLAN | Notes |
|------|----|-------------|-------|
| 1 | 102 (PiAlert) | 20 | Lowest impact, no deps |
| 2 | 103 (Homarr) | 20 | Dashboard — useful to verify |
| 3 | 104 (Nextcloud) | 20 | Data service |
| 4 | 101 (Immich) | 20 | Data service |
| 5 | 107 (Pi-hole) | 10 | DNS — move early, keep reachable |
| 6 | 105 (Wazuh) | 30 | SIEM — keep collecting during migration |
| 7 | 106 (Grafana) | 30 | Monitoring — move after Wazuh |
| 8 | 100 (Hermes) | 40 | DMZ — move last |
| PVE | Host | 10 | Change its IP last |
| PBS | Host | 10 | Has own IP — change separately |

### Step 4: Pi-hole Cross-VLAN DNS

Pi-hole stays on VLAN 10. Enable recursive/forwarding for other VLANs:

```bash
# Pi-hole config — /etc/pihole/setupVars.conf
PIHOLE_DNS_1=10.2.10.1   # OPNsense as upstream
DNSMASQ_LISTENING=all         # Listen on all interfaces
INTERFACE=eth0

# OPNsense — Firewall → Rules → VLAN 10
# Allow DNS (UDP 53) FROM VLANs 20, 30, 40 TO Pi-hole (10.2.10.2)
```

### Step 5: Firewall Rules — The "DMZ Kill Switch"

```bash
# OPNsense — Firewall → Rules → VLAN 40
# Priority order matters (first match wins):

# Rule 1 (Pass): Established/related connections from internal VLANs
#   → allows responses to traffic initiated from VLAN 10/20
# Rule 2 (Block): Block ANY from VLAN 40 → RFC1918 (10.0.0.0/8)
#   → DMZ cannot touch ANY internal network
# Rule 3 (Pass): Allow DMZ → Internet for Jellyfin streaming
# Rule 4 (Pass): Allow DMZ → VLAN 30 (Wazuh agent reporting, UDP 1514)
# Rule 5 (Block): ANY VLAN 40 → ANY (Default deny)

# Test the kill switch:
# From a VLAN 40 CT — these should ALL fail:
ssh root@10.2.40.X "curl http://10.2.10.64:8006"    # Blocked
ssh root@10.2.40.X "ping 10.2.20.44"                # Blocked
ssh root@10.2.40.X "ping 10.2.30.110"               # Blocked (unless Wazuh rule)
```

---

## 4. Phase 2 — Ingress, Reverse Proxy & IAM (MFA)

**Duration:** 3-4 days  
**Difficulty:** Medium  
**Strategy:** Deploy side-by-side with existing services, cut over DNS when ready

### Step 1: Deploy Reverse Proxy (Nginx Proxy Manager or Traefik)

```bash
# Deploy on CT 106 (Docker host) — add to existing docker-compose.yml:

services:
  nginx-proxy-manager:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      - '80:80'
      - '443:443'
      - '81:81'        # Admin UI
    volumes:
      - nginx_data:/data
      - nginx_ssl:/etc/letsencrypt

volumes:
  nginx_data:
  nginx_ssl:

# Admin UI: http://10.2.30.108:81
# Default login: admin@example.com / changeme
```

**Why NPM over bare Nginx:** GUI-based SSL management, built-in Let's Encrypt with Cloudflare DNS-01, access list support for IAM integration later.

### Step 2: Cloudflare DNS-01 SSL (Wildcard Cert)

```bash
# In NPM:
# SSL Certificates → Add Certificate → Let's Encrypt
# Domain: *.infamousjerk.net
# DNS Challenge: Cloudflare
# API Token: [Cloudflare API token with DNS:Edit zone permission]
# Set TTL: 120
# Propagation Seconds: 60
```

**Verification:**
```bash
# DNS-01 challenge means NO open port 80 needed — cert is issued via DNS record
# Verify cert:
curl -vI https://*.infamousjerk.net 2>&1 | grep -i "SSL certificate"
```

### Step 3: Deploy Authentik (IAM + MFA)

```bash
# Add to docker-compose.yml on CT 106:
services:
  authentik-postgres:
    image: postgres:16-alpine
    container_name: authentik-db
    environment:
      POSTGRES_DB: authentik
      POSTGRES_USER: authentik
      POSTGRES_PASSWORD: [STRONG_PASSWORD]
    volumes:
      - authentik_db:/var/lib/postgresql/data
    restart: unless-stopped

  authentik-server:
    image: ghcr.io/goauthentik/server:latest
    container_name: authentik-server
    command: server
    environment:
      AUTHENTIK_REDIS__HOST: redis
      AUTHENTIK_POSTGRESQL__HOST: authentik-db
      AUTHENTIK_POSTGRESQL__USER: authentik
      AUTHENTIK_POSTGRESQL__PASSWORD: [STRONG_PASSWORD]
      AUTHENTIK_SECRET_KEY: [GENERATE_64_CHAR_HEX]
    ports:
      - '9000:9000'   # HTTP
      - '9443:9443'   # HTTPS
    volumes:
      - authentik_media:/media
      - authentik_custom:/custom-templates
    depends_on:
      - authentik-postgres
      - redis
    restart: unless-stopped

  authentik-worker:
    image: ghcr.io/goauthentik/server:latest
    container_name: authentik-worker
    command: worker
    environment:
      AUTHENTIK_REDIS__HOST: redis
      AUTHENTIK_POSTGRESQL__HOST: authentik-db
      AUTHENTIK_POSTGRESQL__USER: authentik
      AUTHENTIK_POSTGRESQL__PASSWORD: [STRONG_PASSWORD]
      AUTHENTIK_SECRET_KEY: [GENERATE_64_CHAR_HEX]
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - authentik_media:/media
      - authentik_custom:/custom-templates
    depends_on:
      - authentik-postgres
      - redis
    restart: unless-stopped

  redis:
    image: redis:alpine
    container_name: authentik-redis
    restart: unless-stopped

volumes:
  authentik_db:
  authentik_media:
  authentik_custom:
```

### Step 4: Configure Authentik Providers

1. **Create an OAuth2/OIDC provider** for each service
2. **Create an Outpost** (proxy provider) to wrap services that don't natively support OIDC
3. **Set up TOTP MFA** as enforcement policy

```yaml
# Authentik Outpost example — deploy as Docker container:
services:
  authentik-proxy-grafana:
    image: ghcr.io/goauthentik/proxy:latest
    container_name: authentik-proxy-grafana
    environment:
      AUTHENTIK_HOST: http://10.2.30.108:9000
      AUTHENTIK_TOKEN: [OUTPOST_TOKEN]
    ports:
      - '3002:9000'   # Forward 3002 → Grafana through Authentik
    restart: unless-stopped
```

**Integration flow:**  
User → `https://grafana.infamousjerk.net` → NPM proxy → Authentik (MFA challenge) → Grafana (authenticated session)

Verification: Access any proxied URL → should redirect to Authentik login → TOTP prompt → service dashboard.

---

## 5. Phase 3 — GitOps & Automation (Ansible + Gitea)

**Duration:** 2-3 days  
**Difficulty:** Medium  
**Strategy:** Gitea already deployed. Add Ansible control node, automate what you already do manually.

### Step 1: Deploy Ansible Control Node

```bash
# Deploy on CT 106 (Docker host) or as lightweight container:
pct create 109 local:vztmpl/ubuntu-24.04-standard_24.04-2_amd64.tar.zst \
  --storage local --memory 1024 --cores 1 --net0 name=eth0,bridge=vmbr0,tag=30,ip=10.2.30.109/24,gw=10.2.30.1
pct start 109

# Install Ansible:
ssh root@10.2.30.109
apt update && apt install -y ansible ansible-lint git sshpass
```

### Step 2: Configure Inventory (Gitea-Backed)

```ini
# /etc/ansible/hosts → stored in Gitea repo: ansible/inventory.yml
all:
  children:
    proxmox:
      hosts:
        pve:
          ansible_host: 10.2.10.64
          ansible_user: root
    containers:
      hosts:
        immich:
          ansible_host: 10.2.20.44
        nextcloud:
          ansible_host: 10.2.20.99
        homarr:
          ansible_host: 10.2.20.105
        wazuh:
          ansible_host: 10.2.30.110
        monitoring:
          ansible_host: 10.2.30.108
        pihole:
          ansible_host: 10.2.10.2
        pialert:
          ansible_host: 10.2.20.77
        hermes:
          ansible_host: 10.2.40.107
    pbs:
      hosts:
        pbs:
          ansible_host: 10.2.10.65
```

### Step 3: Create Automation Playbooks

```yaml
# playbooks/update-all.yml — Staggered patch management
---
- name: Patch infrastructure (VLAN 10)
  hosts: proxmox, pbs
  gather_facts: no
  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600
    - name: Upgrade all packages
      apt:
        upgrade: dist
        autoremove: yes
    - name: Check if reboot required
      stat:
        path: /var/run/reboot-required
      register: reboot_check
    - name: Reboot if needed
      reboot:
        reboot_timeout: 300
      when: reboot_check.stat.exists

- name: Patch core services (VLAN 20)
  hosts: immich, nextcloud, homarr, pialert
  gather_facts: no
  tasks:
    - name: Update and upgrade
      apt:
        update_cache: yes
        upgrade: dist
        autoremove: yes
```

```yaml
# playbooks/docker-backup.yml — Daily config snapshots to Gitea
---
- name: Backup Docker compose & configs to Gitea
  hosts: monitoring
  tasks:
    - name: Tar up config directories
      archive:
        path:
          - /opt/monitoring/docker-compose.yml
          - /opt/monitoring/prometheus/
          - /opt/monitoring/alertmanager/
          - /opt/monitoring/dashboards/
          - /opt/monitoring/provisioning/
        dest: /tmp/docker-configs-{{ ansible_date_time.date }}.tar.gz
    - name: Push to Gitea
      git:
        repo: http://10.2.30.108:3002/InfamousJerk/docker-configs.git
        dest: /tmp/docker-config-backup/
        force: yes
```

### Step 4: Schedule Automation via Cron

```bash
# On Ansible control node — add to crontab:
# Every 6h — Docker config backup
0 */6 * * * ansible-playbook /etc/ansible/playbooks/docker-backup.yml

# Every other day at 2am — staggered patching (replaces current manual cron)
0 2 */2 * * ansible-playbook /etc/ansible/playbooks/update-all.yml

# Weekly — full inventory check + report
0 8 * * 1 ansible all -m ping -o | mail -s "Weekly Health Check" admin@infamousjerk.net
```

**Verification:**
```bash
ansible all -m ping -o  # Should return pong from all CTs
ansible-playbook playbooks/update-all.yml --check  # Dry run
ansible-playbook playbooks/docker-backup.yml  # Should push to Gitea
```

---

## 6. Phase 4 — Advanced SIEM (Syslog + Custom Decoders)

**Duration:** 2-3 days  
**Difficulty:** Medium  
**Strategy:** Wazuh already deployed. Extend with network device logs and custom threat detection.

### Step 1: Forward OPNsense Syslog to Wazuh

```bash
# OPNsense — Services → Syslog → Settings
# Enable remote syslog:
#   Transport: UDP
#   Server: 10.2.30.110
#   Port: 514
#   Facility: all
#   Syslog messages: Include everything (notices, info, errors)

# Wazuh manager — enable UDP syslog listener:
cat >> /var/ossec/etc/ossec.conf << 'EOF'
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>10.2.0.0/16</allowed-ips>
</remote>
EOF

systemctl restart wazuh-manager
```

### Step 2: Forward Managed Switch Logs

```bash
# For each managed switch (assumes Netgear/Cisco/TP-Link CLI):
# configure terminal
# logging host 10.2.30.110
# logging trap informational
# logging source-interface vlan 10  # Source from mgmt IP
# end
# write memory
```

### Step 3: Forward Pi-hole Logs

```bash
# Pi-hole already logs all DNS queries to /var/log/pihole.log
# Two options:

# Option A (Recommended): Lightweight forwarder
ssh root@10.2.10.2 "
  apt install -y rsyslog
  echo '*.* @10.2.30.110:514' >> /etc/rsyslog.d/50-pihole-forward.conf
  systemctl restart rsyslog
"

# Option B (Advanced): Wazuh agent reads Pi-hole log
# Already done if Wazuh agent is on CT 107
# Add to /var/ossec/etc/ossec.conf on CT 107:
# <localfile>
#   <location>/var/log/pihole.log</location>
#   <log_format>syslog</log_format>
# </localfile>
```

### Step 4: Create Custom Wazuh Decoder for Jellyfin

```xml
<!-- /var/ossec/etc/decoders/local_decoder.xml -->
<!-- Decoder: Jellyfin authentication attempts -->
<decoder name="jellyfin-auth">
  <parent>syslog</parent>
  <type>syslog</type>
  <prematch>jellyfin:\s</prematch>
  <regex>from ip (\S+) user|User (\S+) authentication</regex>
  <order>srcip, user</order>
</decoder>
```

### Step 5: Create Alert Rule for Jellyfin Brute-Force

```xml
<!-- /var/ossec/etc/rules/local_rules.xml -->
<group name="jellyfin,">

  <!-- Single failed Jellyfin login -->
  <rule id="100010" level="5">
    <decoded_as>jellyfin-auth</decoded_as>
    <match>Authentication failed</match>
    <description>Jellyfin: Failed login attempt from $(srcip)</description>
  </rule>

  <!-- 10+ failed logins in 5 minutes = brute force -->
  <rule id="100011" level="10">
    <if_sid>100010</if_sid>
    <frequency>10</frequency>
    <timeframe>300</timeframe>
    <description>Jellyfin: Brute-force detected from $(srcip) — $(frequency) failures in under 5 minutes</description>
    <group>brute_force,</group>
  </rule>

  <!-- Successful login after failures = compromised -->
  <rule id="100012" level="12">
    <if_sid>100010</if_sid>
    <match>Authentication succeeded</match>
    <description>Jellyfin: Successful login from $(srcip) after prior failures</description>
    <group>compromised,</group>
  </rule>

</group>
```

**Verification:**
```bash
# Test from any CT:
curl -s -o /dev/null -w '%{http_code}' http://10.2.40.X:8096/auth/login -X POST -d '{"username":"admin","password":"wrong"}'

# Should fire rule 100010 → level 5 alert → Discord notification
# Repeat 10× in 5 min → should fire rule 100011 → level 10
```

### Step 6: Grafana SIEM Dashboard

Create a Grafana dashboard with Wazuh indexer as datasource:

```
Panels to include:
├── Real-time alert stream (last 50 alerts)
├── Alert count by severity (last 24h)
├── Top 10 source IPs by alert volume
├── Jellyfin brute-force attempts (time series)
├── Failed SSH by CT (bar chart)
└── Alert timeline (annotated scatter plot)
```

---

## 7. Verification & Validation

### Phase 1 — VLAN Segregation

| Test | Method | Expected Result |
|------|--------|----------------|
| Cross-VLAN connectivity | `ping 10.2.20.44` from VLAN 40 | ❌ Timeout |
| DMZ → Wazuh | `nc -z 10.2.30.110 1514` from VLAN 40 | ✅ Success |
| Management → All VLANs | `ssh root@<ANY_CT>` from VLAN 10 | ✅ Success |
| Pi-hole cross-VLAN | `nslookup google.com 10.2.10.2` from any VLAN | ✅ Resolves |
| DMZ → LAN (initiated) | `curl http://10.2.20.99` from VLAN 40 | ❌ Timeout |
| DMZ → LAN (responding) | From VLAN 20: `curl http://10.2.40.107:8642` | ✅ Must reach Hermes |

### Phase 2 — Reverse Proxy & IAM

| Test | Method | Expected Result |
|------|--------|----------------|
| SSL cert valid | `curl -vI https://*.infamousjerk.net` | ✅ Valid, Cloudflare-signed |
| Authentik login | Browse to service URL | → Redirect to Authentik |
| MFA enforced | Login with password only | ❌ Denied (TOTP prompt) |
| MFA bypass | Login with password + TOTP | ✅ Service dashboard shown |

### Phase 3 — Ansible Automation

| Test | Method | Expected Result |
|------|--------|----------------|
| All hosts reachable | `ansible all -m ping` | ✅ All pong |
| Update dry-run | `ansible-playbook update-all.yml --check` | ✅ No errors |
| Docker backup | `ansible-playbook docker-backup.yml` | ✅ Configs in Gitea |
| Cron running | `crontab -l` on Ansible node | ✅ 3 jobs listed |

### Phase 4 — SIEM

| Test | Method | Expected Result |
|------|--------|----------------|
| OPNsense logs in Wazuh | `grep 'OPNsense\|filterlog' /var/ossec/logs/alerts/alerts.json` | ✅ Firewall events present |
| Pi-hole logs in Wazuh | `grep 'pihole' /var/ossec/logs/alerts/alerts.json` | ✅ DNS queries visible |
| Jellyfin failed auth | `curl -X POST .../auth/login -d '{"username":"a","password":"b"}'` | ✅ Level 5 alert |
| Jellyfin brute-force | Repeat 10× in 5 min | ✅ Level 10 alert |
| Discord notification | Check Discord #alerts | ✅ Alert posted |

---

## 8. Interview Talking Points

### "Why did you segment your network?"

> *"My homelab runs 8 containers exposing web UIs, an AI agent, and a media server. In a flat network, a single compromised service — say Jellyfin with an unpatched CVE — gives an attacker immediate access to my entire infrastructure: PVE, Wazuh, Nextcloud, all of it. VLAN segmentation forces lateral movement. An attacker in the DMZ (VLAN 40) literally cannot even ping the management network. The firewall rule is three lines in OPNsense, but it's the single highest-ROI security control you can implement."*

### "How did you handle cross-VLAN dependencies like DNS and logging?"

> *"Pi-hole is on VLAN 10 (Management). Every VLAN needs DNS. I configured OPNsense firewall rules to allow UDP 53 from all VLANs to Pi-hole, and enabled `DNSMASQ_LISTENING=all` on Pi-hole. For logging, Wazuh agents are installed on every CT regardless of VLAN. The agents call home to the manager on VLAN 30 via TCP 1514, and I added OPNsense firewall pass rules for that specific traffic. The key design principle: allow only what's explicitly required, deny everything else."*

### "Why Ansible over just keeping the manual cron jobs?"

> *"Two reasons. First, consistency — my manual cron was one script that SSH'd into each CT and ran apt commands. If a CT was down or a new one was added, the script failed silently. Ansible gives me inventory management, idempotent playbooks, and clear success/failure reporting. Second, it's auditable. Playbooks live in Gitea. When an interviewer asks 'how do you patch your systems,' I show them the commit history — not a bash script in root's home directory."*

### "What's the most interesting Wazuh rule you wrote?"

> *"The Jellyfin brute-force detector. Jellyfin doesn't natively integrate with SIEM tools, so I wrote a custom Wazuh decoder that parses its authentication logs, extracts the source IP and username, and feeds that into a frequency-based rule: 10 failed logins in 5 minutes triggers a level 10 alert. What makes it interesting is the escalation logic — if a login succeeds after those failures, we get a level 12 alert for potential account compromise. That's the difference between monitoring for events and detecting actual threats."*

### "What would you do differently if you had to do it again?"

> *"I'd start with one VLAN (management) and the reverse proxy before anything else. Migrating 8 containers from a flat IP scheme to VLANs requires coordinating IP changes with OPNsense firewall rules, Pi-hole DNS updates, and service configs — all while keeping services running. The backup-to-PBS before migration was critical, but if I had the reverse proxy and IAM in place first, I could have tested the full authentication flow on the flat network before adding the complexity of VLAN routing. Also: document the before-state MAC addresses. I forgot to, and matching CTs to their new DHCP leases was annoying."*

### "How do you know the system is secure?"

> *"I don't assume it is — I validate. After every phase, I run a verification checklist: Can VLAN 40 reach VLAN 10? (It shouldn't.) Are Wazuh agents reporting from all CTs? Is the Jellyfin brute-force rule firing correctly? I treat the verification table in my project plan as a living document — every change gets tested against it. The day I stop verifying is the day I introduce a misconfiguration that an attacker will find before I do."*

---

## Appendix A: Estimated Timeline

| Phase | Tasks | Duration | Dependencies |
|-------|-------|----------|--------------|
| **0** | Backups, exports, patches | 1-2 days | None |
| **1** | VLANs, firewall, CT migration | 3-4 days | Phase 0 |
| **2** | NPM, Cloudflare DNS-01, Authentik | 3-4 days | Phase 1 (VLAN routing working) |
| **3** | Ansible, playbooks, cron | 2-3 days | Gitea already deployed |
| **4** | Syslog forwarders, decoders, Grafana | 2-3 days | Phase 1 (cross-VLAN logging) |
| **Buffer** | Bug fixes, edge cases | 2-3 days | — |

**Total: ~2-3 weeks** (evening/weekend work)

---

## Appendix B: Skills Demonstrated for Portfolio

| Skill | Where It Shows |
|-------|----------------|
| Network segmentation | 4 VLANs with least-privilege firewall rules |
| Firewall engineering | OPNsense rule creation, DMZ kill switch |
| SIEM engineering | Wazuh deployment, custom decoder/rule authoring |
| IAM & MFA | Authentik OIDC provider + TOTP enforcement |
| Reverse proxy | NPM + Cloudflare DNS-01 wildcard certs |
| Infrastructure as Code | Ansible playbooks in Gitea |
| Docker orchestration | Multi-container compose with provisioning |
| Backup & recovery | PBS with encrypted backups, tested restore |
| Incident response | 4-phase playbook (isolate → investigate → eradicate → recover) |
| Documentation | This entire project plan + homelab-docs repo |

---

> **Last Updated:** 2026-05-25  
> **Maintainer:** Anthony Piper (iNfamousJerk)  
> **Repo:** [homelab-docs](https://github.com/iNfamousJerk/homelab-docs) (GitHub) / Gitea (10.2.7.108:3002)