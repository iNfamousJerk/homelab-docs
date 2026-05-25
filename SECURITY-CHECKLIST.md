# 🔒 Infrastructure Expansion: Security Checklist

> **Mandatory review before adding any new service, container, API key, or external integration to this homelab.**
>
> Follow this checklist for every expansion. If any step is skipped, the expansion is not complete.

---

## Phase 1: Pre-Design (Before You Code)

### 1.1 Threat Model the New Service

- [ ] **What data does it touch?** — Photos? Passwords? Network config? If it stores user data, it needs encryption at rest.
- [ ] **What are the access boundaries?** — Does it need to be on VLAN 10 (infra) or VLAN 20 (media)? Internet-facing or LAN-only?
- [ ] **Does it need credentials?** — If yes, plan where they live (not in the code).
- [ ] **Does it need an API token?** — Generate one with the minimum scope needed. No wildcard tokens.
- [ ] **Does it expose a port?** — On which interface? Does it need a firewall rule in OPNsense?

### 1.2 Architecture Decision Record

- [ ] **Storage location** — local-lvm, PBS-backed, or separate mount? Document with `pvesm status`.
- [ ] **Network placement** — VLAN tag, static IP (reserved in Pi-hole DHCP), firewall rules documented.
- [ ] **Container spec** — RAM, CPU, disk size, boot order. Add to `02-containers.md`.
- [ ] **Authentication model** — SSO (Authentik), local auth, API key? No default passwords.

---

## Phase 2: Implementation (Credentials Isolation)

### 2.1 Never Hardcode Secrets

```
# ❌ BAD — Never do this:
POSTGRES_PASSWORD=myrealpassword123

# ✅ GOOD — Use placeholders in the code:
POSTGRES_PASSWORD=${POSTGRES_PASSWORD}

# ✅ Or use a .env file that is .gitignored and never committed:
POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
```

- [ ] All passwords, tokens, and API keys are referenced as `${VAR_NAME}` in tracked files.
- [ ] All actual credentials live in `~/.hermes/.env`, a `.env` file, or a password manager.
- [ ] A `.env.example` or `secrets.template` file documents what vars are needed (not their values).

### 2.2 Sensitive Data Checklist

| Data Type | Where It Goes | Never In |
|-----------|--------------|----------|
| Passwords | `.env` (gitignored), password manager | `.md` files, `docker-compose.yml`, config files |
| API Keys | `.env` (gitignored), Hermes memory | Code, tracked configs |
| Webhook URLs | `.env` (gitignored), Wazuh manager config | Public commit history |
| SSH Private Keys | `~/.ssh/` only | Any git repo |
| TLS Certificates | CT-specific store, Homarr trusted-certs | Repo (exception: documented public CA certs) |
| Encryption Keys | PBS offline backup, paper copy | Any digital store that syncs to cloud |

### 2.3 Git Isolation

- [ ] `.gitignore` exists and covers: `.env`, `*.pem`, `*.key`, `*.cert`, `*.log`
- [ ] Pre-commit hook is active: check with `ls -la .git/hooks/pre-commit`
- [ ] Test the hook: `git add sensitive-file && git commit -m "test"` — verify it blocks

---

## Phase 3: Deployment & Verification

### 3.1 Container Lifecycle

- [ ] Container created with explicit `pct create` flags (see `00-pve-pbs-lifecycle-playbook.md`).
- [ ] Boot order is set: `--startup order=N,up=M` and `--onboot 1`.
- [ ] DNS set to 10.2.7.2: `--nameserver 10.2.7.2`.
- [ ] Features set correctly: `--features keyctl=1,nesting=1` if Docker.
- [ ] Root password is set (unique, themed, documented in Hermes memory + password manager).
- [ ] SSH key deployed: `mkdir -p ~/.ssh && echo "<pubkey>" >> ~/.ssh/authorized_keys`.

### 3.2 Documentation Update

- [ ] Container specs added to `02-containers.md` (RAM, cores, disk, boot order, purpose).
- [ ] Service added to Homarr dashboard with ping_url and icon.
- [ ] If monitoring needed — Prometheus scrape target added to `prometheus.yml`.
- [ ] If Wazuh agent needed — installed and configured with `MANAGER_IP=10.2.7.110`.
- [ ] Backup configured: `vzdump <CT_ID> --storage pbs --mode snapshot --compress zstd`.
- [ ] Credentials listed in CREDENTIALS.md (as `[REDACTED]` for public, full in Gitea version).
- [ ] Service endpoint added to Homarr's service list.

### 3.3 Verification Tests

- [ ] Service responds on its port: `curl -s -o /dev/null -w '%{http_code}' http://<IP>:<PORT>/`
- [ ] Container auto-starts on boot: `pct reboot <CT_ID> && pct status <CT_ID>`
- [ ] Backup runs successfully: check `pvesh get /cluster/tasks --typefilter backup`
- [ ] Wazuh agent checks in: Dashboard → Agents → new agent listed as Active
- [ ] Homarr ping shows green: open dashboard, verify health dot

---

## Phase 4: Continuous Protection

### 4.1 Regular Audits

- [ ] **Weekly:** `git log --oneline -10` — review recent commits for credential slippage.
- [ ] **Monthly:** Run pre-commit scan manually: `python3 scripts/pre-commit-secret-scan.py`
- [ ] **Per-rebuild:** Full secret scan of git history (see `00-pve-pbs-rebuild-blueprint.md`).
- [ ] **Per-domain change:** When `infamousjerk.net` is registered, update all `[REDACTED]` references.

### 4.2 Incident Response (if a secret leaks to GitHub)

1. **Immediate:** Revoke the leaked credential (rotate password, invalidate token, change webhook URL).
2. **Remove from history:** Use `git filter-repo` to purge the commit:
   ```bash
   # Install: pip install git-filter-repo
   # Purge a specific file:
   git filter-repo --path CREDENTIALS.md --invert-paths
   # Or purge a specific string from all files:
   git filter-repo --replace-text <(echo "LEAKED_TOKEN==>***")
   ```
3. **Force push:** `git push origin --force --all`
4. **Check forks:** Warn any forks to delete their copies.
5. **Document:** Note the incident in DR-IR-PLAYBOOK.md.

---

## Appendix: Quick Reference

| Command | Purpose |
|---------|---------|
| `python3 scripts/pre-commit-secret-scan.py` | Manual secret scan (dry run) |
| `git commit --no-verify` | Bypass pre-commit hook (emergency only) |
| `git filter-repo --path file.txt --invert-paths` | Purge file from all history |
| `grep -rn 'password\|secret\|token\|webhook' --include='*.md' .` | Quick manual sweep |
| `ss -tulpn \| grep <PORT>` | Check if service is listening |
| `pvesh get /cluster/tasks --typefilter backup` | Verify latest backup task status |

---

> **Version:** 1.0 — Created 2026-05-25
> **Related:** `00-pve-pbs-rebuild-blueprint.md` (rebuild secrets mgmt), `DR-IR-PLAYBOOK.md` (incident response)