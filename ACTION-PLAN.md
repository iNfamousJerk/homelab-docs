# 🎯 Homelab Hardening: Phased Action Plan

> **Objective:** Bridge gaps between current state and a hardened, proactive, enterprise-grade infrastructure.
> **Total Timeline:** ~4-6 weeks (evening/weekend work)
> **Priority Legend:** 🔴 Critical / 🟡 High / 🟢 Medium / 🔵 Low

---

## Phase 0: Quick Wins — Security Triage (Days 1-2)

| # | Task | Priority | Effort | Status | Notes |
|---|------|----------|--------|--------|-------|
| 0.1 | ✅ **Strip plaintext passwords from docs** | 🔴 | 15 min | **DONE** | 15-pbs-setup.md, docker-compose.yml fixed |
| 0.2 | ✅ **Fix docker-compose PIHOLE_HOSTNAME** | 🔴 | 1 min | **DONE** | Was pointing at 10.2.7.209 (stale) |
| 0.3 | ✅ **Create .env.example file** | 🟡 | 5 min | **DONE** | Seeded with ${VAR_NAME} placeholders |
| 0.4 | ✅ **Create NETWORK-TOPOLOGY.md (IPAM+VLANs)** | 🟡 | 15 min | **DONE** | Single source of truth for IPs |
| 0.5 | ✅ **Create SOP-BACKUP-RESTORE.md** | 🟡 | 20 min | **DONE** | Includes RTO/RPO targets, restore drill |
| 0.6 | ✅ **Create CHANGELOG.md** | 🟡 | 5 min | **DONE** | Reverse-chronological change tracking |
| 0.7 | ✅ **Create CHANGE-LOG-TEMPLATE.md** | 🟡 | 5 min | **DONE** | Standardized CR form |
| 0.8 | ✅ **Fix PVE Debian base version** | 🔴 | 1 min | **DONE** | Bookworm→Trixie |
| 0.9 | **Push all changes to GitHub + Gitea** | 🔴 | 5 min | **PENDING** | |
| 0.10 | **Deploy .env file to CT 106** | 🔴 | 10 min | **PENDING** | ssh to CT 106, create /opt/monitoring/.env |
| 0.11 | **Docker compose restart with .env** | 🔴 | 5 min | **PENDING** | `docker compose --env-file .env up -d` |

---

## Phase 1: Network Architecture & Segmentation (Days 3-7)

| # | Task | Priority | Effort | Dependency |
|---|------|----------|--------|------------|
| 1.1 | **Implement VLANs on OPNsense** | 🔴 | 2h | OPNsense access |
| 1.2 | **Configure PVE bridge VLANs (vmbr0.10-.40)** | 🔴 | 30 min | 1.1 |
| 1.3 | **Migrate CTs to VLANs (dependency order)** | 🔴 | 4h | 1.2 |
| 1.4 | **Configure cross-VLAN firewall rules** | 🔴 | 1h | 1.3 |
| 1.5 | **DMZ kill switch verification** | 🔴 | 30 min | 1.4 |
| 1.6 | **Pi-hole cross-VLAN DNS config** | 🟡 | 30 min | 1.3 |
| 1.7 | **Document all firewall rules in 03-network.md** | 🟡 | 30 min | 1.4 |
| 1.8 | **Update NETWORK-TOPOLOGY.md with target state** | 🟢 | 15 min | 1.3 |

### Migration Order (from SOC-UPGRADE-PLAN.md):
```
PiAlert (102) → VLAN 20  │  Homarr (103) → VLAN 20
Nextcloud (104) → VLAN 20 │  Immich (101) → VLAN 20
Pi-hole (107) → VLAN 10   │  Wazuh (105) → VLAN 30
Grafana (106) → VLAN 30   │  Hermes (100) → VLAN 40
```

---

## Phase 2: Identity & Access Management (Days 8-12)

| # | Task | Priority | Effort | Dependency |
|---|------|----------|--------|------------|
| 2.1 | **Deploy Nginx Proxy Manager (Docker)** | 🟡 | 1h | VLANs working |
| 2.2 | **Configure Cloudflare DNS-01 wildcard cert** | 🟡 | 1h | 2.1, buy infamousjerk.net domain |
| 2.3 | **Deploy Authentik + PostgreSQL + Redis** | 🟡 | 2h | 2.1 |
| 2.4 | **Configure TOTP MFA enforcement** | 🟡 | 1h | 2.3 |
| 2.5 | **Wire Grafana → NPM → Authentik** | 🟡 | 1h | 2.2, 2.3 |
| 2.6 | **Wire all web UIs through Authentik** | 🟢 | 2h | 2.5 |
| 2.7 | **Enable auth on Prometheus (basic auth or reverse-proxy)** | 🟡 | 30 min | 2.1 |
| 2.8 | **Enable auth on Alertmanager** | 🟡 | 15 min | 2.7 |
| 2.9 | **Update CREDENTIALS.md with Authentik admin** | 🟢 | 10 min | 2.3 |
| 2.10 | **Deploy Vaultwarden (CT 106 Docker)** | 🟢 | 30 min | Already in compose volumes |

---

## Phase 3: Storage & Backup Hardening (Days 13-16)

| # | Task | Priority | Effort | Dependency |
|---|------|----------|--------|------------|
| 3.1 | **Enable PBS encryption** | 🔴 | 1h | Physical access to PBS for key backup |
| 3.2 | **Print PBS encryption key to paper + store offline** | 🔴 | 15 min | 3.1 |
| 3.3 | **Set up PBS ZFS scrub schedule (monthly cron)** | 🟡 | 15 min | SSH to PBS |
| 3.4 | **Configure PBS email notifications on failure** | 🟡 | 30 min | SMTP relay or Discord webhook |
| 3.5 | **Run first monthly restore drill (SOP-BACKUP-001)** | 🟡 | 30 min | 3.1 |
| 3.6 | **Add PBS-failure scenario to DR-IR-PLAYBOOK.md** | 🟡 | 1h | Understanding of PBS recovery |
| 3.7 | **Set Prometheus retention policy** | 🟡 | 15 min | `--storage.tsdb.retention.time=30d` |
| 3.8 | **Add PBS datastore disk usage alert to Grafana** | 🟢 | 30 min | Existing nut-exporter pattern |
| 3.9 | **Document RTO/RPO targets in DR-IR-PLAYBOOK.md** | 🟢 | 15 min | Already in SOP-BACKUP-RESTORE.md |

---

## Phase 4: Monitoring & SIEM Upgrade (Days 17-22)

| # | Task | Priority | Effort | Dependency |
|---|------|----------|--------|------------|
| 4.1 | **Upgrade Wazuh to 5.x** | 🔴 | 2h | SSH to CT 105 |
| 4.2 | **Update Wazuh agent deployment script (13-wazuh.md)** | 🔴 | 30 min | 4.1 |
| 4.3 | **Fix Wazuh agent enrollment (add auth key step)** | 🔴 | 30 min | 4.1 |
| 4.4 | **Create agent inventory table in 13-wazuh.md** | 🟡 | 15 min | — |
| 4.5 | **Configure Wazuh indexer retention policy** | 🟡 | 30 min | 4.1 |
| 4.6 | **Forward OPNsense syslog to Wazuh** | 🟡 | 30 min | VLANs working |
| 4.7 | **Forward Pi-hole logs to Wazuh** | 🟡 | 30 min | 4.1 |
| 4.8 | **Pin Docker images to specific versions** | 🟡 | 30 min | Replace `:latest` with `:X.Y.Z` |
| 4.9 | **Replace cAdvisor GCR image with GHCR-based** | 🟡 | 15 min | GCR deprecated |
| 4.10 | **Add Docker healthchecks to compose file** | 🟢 | 30 min | — |
| 4.11 | **Set up self-monitoring (Grafana monitors itself)** | 🟢 | 1h | — |

---

## Phase 5: GitOps & Automation (Days 23-28)

| # | Task | Priority | Effort | Dependency |
|---|------|----------|--------|------------|
| 5.1 | **Deploy Ansible control node (CT 109)** | 🟡 | 30 min | VLANs working |
| 5.2 | **Configure Ansible inventory in Gitea** | 🟡 | 30 min | 5.1 |
| 5.3 | **Create update-all playbook** | 🟡 | 1h | 5.1 |
| 5.4 | **Create docker-backup playbook** | 🟢 | 30 min | 5.1 |
| 5.5 | **Schedule Ansible crons** | 🟢 | 15 min | 5.3, 5.4 |
| 5.6 | **Set up auto-push: docs → GitHub + Gitea (cron job via Hermes)** | 🟡 | 30 min | — |
| 5.7 | **Configure pre-commit hook on clone hosts** | 🟢 | 15 min | Already in scripts/ |

---

## Phase 6: Documentation Completeness (Days 29-35)

| # | Task | Priority | Effort | Dependency |
|---|------|----------|--------|------------|
| 6.1 | **Fix README stale date (Dec 2026 → May 2026)** | 🟡 | 1 min | — |
| 6.2 | **Add 15-pbs-setup.md to README categorized tables** | 🟡 | 5 min | — |
| 6.3 | **Add services-roadmap.md, PLANS/ to README** | 🟢 | 5 min | — |
| 6.4 | **Remove or fix broken reference to monitoring-stack.md** | 🟡 | 5 min | — |
| 6.5 | **Add per-CT OS/distro version field to 02-containers.md** | 🟡 | 15 min | SSH audit |
| 6.6 | **Add per-CT Docker nesting field to 02-containers.md** | 🟡 | 10 min | — |
| 6.7 | **Fix 10-grafana.md circular SSH command** | 🟡 | 1 min | Line 492 |
| 6.8 | **Add DNS config section to 01-proxmox.md** | 🟢 | 10 min | — |
| 6.9 | **Add physical network diagram SVG** | 🔵 | 2h | Draw.io or excalidraw |
| 6.10 | **Consolidate CREDENTIALS.md into Gitea-only version** | 🟢 | 15 min | Remove IP schema from public GitHub |

---

## Hardware Shopping List (v2 Expansion)

| Item | Qty | Est. Cost | Phase |
|------|-----|-----------|-------|
| 2×4TB CMR NAS HDD (PBS upgrade) | 2 | ~$160-240 | 3 |
| Managed switch (16-24 port, PoE optional) | 1 | ~$50-200 | 1 |
| Wi-Fi AP (U6 Lite / EAP610) | 1 | ~$60-100 | 1 |
| Patch panel + shelf + cables | 1 set | ~$60 | 1 |
| Zimaboard 2 (future router) | 1 | ~$200 | Future |

---

## Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| VLAN migration breaks connectivity | Medium | High | Full PBS backup before start; rollback plan ready |
| PBS encryption key lost | Low | Critical | Paper backup in safe + encrypted USB |
| Wazuh 5.x upgrade breaks agents | Medium | Medium | Test on 1 CT first; keep 4.x backup |
| Authentik misconfig locks out services | Medium | High | Keep local admin bypass; test with 1 service first |
| Domain (infamousjerk.net) not yet registered | High | Medium | Can use Cloudflare Tunnel as alternative |