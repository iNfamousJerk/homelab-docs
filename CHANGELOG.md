# 📋 Homelab Documentation Change Log

> **Purpose:** Track who changed what, when, and why — for audit trails, rollback context, and MTTR reduction.
> **Format:** Reverse chronological. One entry per significant change batch.
> **Policy:** Every doc update MUST include a CHANGELOG.md entry.

---

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
