# 📋 Homelab Documentation Change Log

> **Purpose:** Track who changed what, when, and why — for audit trails, rollback context, and MTTR reduction.
> **Format:** Reverse chronological. One entry per significant change batch.
> **Policy:** Every doc update MUST include a CHANGELOG.md entry.

---

## [2026-05-27] — SOC Monitoring Expansion + Homarr Office Column

- **Prometheus**: Added blackbox_http (16 targets), blackbox_tcp (10 targets), nut (UPS) jobs — 40/40 targets up
- **Node Exporters**: Installed on ALL 9 CTs (100-108) for per-container system metrics
- **Grafana**: Created "🏠 Homelab Overview" dashboard (41 panels) — system metrics per CT, service uptime, Office/Media service status panels, quick links to Homarr and every service
- **Homarr**: Added Office column (OnlyOffice, Vaultwarden, Actual Budget, LanguageTool), moved NPM to Networking, added PBS to Networking, removed old pirate Vaultwarden
- **Docs**: Updated 07-homarr.md (full layout table), 10-grafana.md (40 targets, new dashboard), services-roadmap.md (SOC monitoring status, Actual Budget deployed)

## [2026-05-27] — Office Stack Deployed + Vaultwarden Migration

| File | Change | Reason | Author |
|------|--------|--------|--------|
| `configs/nextcloud-office-stack.yml` | Created | Compose file for Vaultwarden, OnlyOffice, LanguageTool, Actual Budget on CT 104 | Hermes |
| `services-roadmap.md` | Updated Vaultwarden location, added OnlyOffice/LanguageTool/Actual/Nextcloud apps | Reflect deployed office stack | Hermes |
| `CREDENTIALS.md` | Added OnlyOffice, LanguageTool, Actual Budget entries | New services need login refs | Hermes |
| `16-pirate-media-stack.md` | Removed Vaultwarden from pirate services | Migrated to CT 104 | Hermes |

**Infrastructure changes:**
- Nextcloud CT bumped 512MB→4GB RAM, 1→2 vCPU, 30→50GB disk
- Vaultwarden migrated from CT 108 (pirate) → CT 104 (nextcloud)
- New Docker stack on CT 104: Vaultwarden (:8082), OnlyOffice (:8083), LanguageTool (:8010), Actual Budget (:5006)
- NPM proxy hosts added: vaultwarden.pirate.lan, budget.pirate.lan, languagetool.pirate.lan
- Pi-hole DNS wildcard *.pirate.lan → 10.2.7.109 added
- Nextcloud apps enabled: Deck, Mail, Calendar, Talk
- NPM login reset: admin@example.com → anthonypiper1@gmail.com / nppass
- Prometheus monitoring expanded: 40 targets across 9 jobs
- Node exporters installed on: Hermes (100), Immich (101), PiAlert (102), Homarr (103), Nextcloud (104), Wazuh (105), Pi-hole (107), Pirate (108)
- Blackbox HTTP probes: OnlyOffice, Actual, LanguageTool, Jellyfin, all *arrs, NPM
- Blackbox TCP probes: Vaultwarden, NPM port 80, Hermes API
- Added configs/prometheus.yml to docs repo

---

## [2026-05-26] — Pirate Media Stack Documentation

| File | Change | Reason | Author |
|------|--------|--------|--------|
| `16-pirate-media-stack.md` | Created | Full doc for CT 108 — NPM, Vaultwarden, *arrs, Jellyfin | Hermes |
| `README.md` | Added CT 108 to inventory, services, future plans | Missing from overview | Hermes |
| `02-containers.md` | Added CT 108 to storage table + update loop | Missing disk usage data | Hermes |
| `NETWORK-TOPOLOGY.md` | Added CT 108 to logical diagram + IPAM table | Missing from network map | Hermes |
| `CREDENTIALS.md` | Added CT 108 + NPM/Vaultwarden/Jellyfin entries | Missing service creds | Hermes |
| `services-roadmap.md` | Marked Vaultwarden, NPM, media stack ✅ | Roadmap now reflects deployed state | Hermes |

## [2026-05-25] — Docs Audit & Remediation (Phase 1)

| File | Change | Reason | Author |
|------|--------|--------|--------|
| `01-proxmox.md` | Fixed PVE Debian base: Bookworm→Trixie | PVE 9.x uses Debian 13 | Hermes |
| `CHANGELOG.md` | Created | Missing change control artifact | Hermes |
| `NETWORK-TOPOLOGY.md` | Created | Missing network/IPAM/VLAN documentation | Hermes |
| `SOP-BACKUP-RESTORE.md` | Created | Missing backup runbook | Hermes |
| `CHANGE-LOG-TEMPLATE.md` | Created | Missing standardized format | Hermes |

## [2026-05-25] — Previous State (Pre-Audit)

> Baseline: 20 docs covering PVE, CTs, networking, monitoring, DR/IR, SOC plan, security checklist.
> Repos: GitHub (origin) + Gitea (gitea) — both synced.
