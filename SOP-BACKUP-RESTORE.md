# 📋 SOP: Backup Verification & Restore Drill

> **SOP ID:** SOP-BACKUP-001
> **Scope:** PBS backups → verified restore
> **Frequency:** Monthly (drill) / Weekly (verification)
> **MTTR Target:** < 30 min for full CT restore
> **Related:** `DR-IR-PLAYBOOK.md`, `15-pbs-setup.md`

---

## ⚡ Quick-Start (Emergency: Restore a CT NOW)

```bash
# 1. List available backups
pvesm list pbs

# 2. Restore to a temporary CT ID (e.g., 200)
pct restore 200 pbs:backup/<CT_ID>/latest --storage local --unique

# 3. Start and verify
pct start 200
pct enter 200 -- curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1/
# → Should return 200

# 4. Service-specific verification (check the service responds)
case <CT_ID> in
  101) curl -s http://10.2.7.44 | grep -qi immich ;;
  104) curl -s http://10.2.7.99 | grep -qi nextcloud ;;
  105) curl -sk https://10.2.7.110 | grep -qi wazuh ;;
  106) curl -s http://10.2.7.108:3000 | grep -qi grafana ;;
  107) curl -s http://10.2.7.2/admin | grep -qi pihole ;;
esac

# 5. If verified OK, stop temp CT and swap IPs
pct stop 200
pct restore <CT_ID> pbs:backup/<CT_ID>/latest --storage local --force
pct start <CT_ID>

# 6. If verification FAILED, troubleshoot:
pct destroy 200
pct restore 200 pbs:backup/<CT_ID>/<EARLIER_BACKUP> --storage local --unique
```

---

## 1. Weekly Backup Verification (5 min)

**When:** Every Monday at 9am (or after any infrastructure change)
**Goal:** Confirm backups are running and recent

```bash
# 1. Check backup task history
pvesh get /cluster/tasks --typefilter backup --limit 5

# 2. Verify all CTs have backups within the last 7 days
for ctid in 100 101 102 103 104 105 106 107; do
  backupdate=$(pvesm list pbs --vmid $ctid 2>/dev/null | awk '{print $2}' | sort -r | head -1)
  days_ago=$(( ($(date +%s) - $(date -d "$backupdate" +%s)) / 86400 ))
  if [ "$days_ago" -gt 7 ]; then
    echo "⚠️  CT $ctid — last backup $days_ago days ago (stale!)"
  else
    echo "✅ CT $ctid — backed up $days_ago days ago"
  fi
done

# 3. Check PBS datastore health
pbs_datastore_usage=$(ssh root@10.2.7.65 "proxmox-backup-manager datastore list" 2>/dev/null)
echo "$pbs_datastore_usage"

# 4. Check PBS ZFS health
ssh root@10.2.7.65 "zpool status backup | grep -E 'state:|errors:|scan:'"
```

**Expected state:**
- All CTs: backup age < 7 days
- PBS datastore: usage < 80%
- ZFS pool: `state: ONLINE`, `errors: No known data errors`

---

## 2. Monthly Restore Drill (30 min)

**When:** First Saturday of each month
**Goal:** Prove backups are restorable — rotate which CT gets tested

### Pre-Drill Checklist

- [ ] Enough free space on PVE for a temp CT (`pvesm status` — need at least 5GB free)
- [ ] PBS is reachable: `curl -sk https://10.2.7.65:8007`
- [ ] CREDENTIALS.md accessible (if whole system is up, it's fine; test also with offline copy)

### Drill Procedure

```bash
# 1. Select CT to test (rotate monthly: 101 Jan, 102 Feb, etc.)
CT_TO_TEST=${1:-101}  # Pass as arg or use rotation
TEMP_ID=200

# 2. Get latest backup
LATEST=$(pvesm list pbs --vmid $CT_TO_TEST 2>/dev/null | awk '{print $2}' | sort -r | head -1)
echo "Restoring CT $CT_TO_TEST from backup dated: $LATEST"

# 3. Restore to temp ID
pct restore $TEMP_ID pbs:backup/$CT_TO_TEST/latest --storage local --unique
if [ $? -ne 0 ]; then
  echo "❌ RESTORE FAILED — escalate to DR playbook"
  exit 1
fi

# 4. Start container
pct start $TEMP_ID
sleep 5

# 5. Verify service responds
TEMP_IP=$(pct exec $TEMP_ID -- hostname -I 2>/dev/null | awk '{print $1}')
echo "Temp CT started at $TEMP_IP"
pct exec $TEMP_ID -- curl -s -o /dev/null -w "HTTP %{http_code}\n" http://127.0.0.1/

# 6. Clean up
pct stop $TEMP_ID
pct destroy $TEMP_ID
echo "✅ Restore drill PASSED for CT $CT_TO_TEST"
```

### Drill Log Template

| Date | CT Tested | Backup Date | Restore Time | HTTP Code | Result | Notes |
|------|-----------|-------------|--------------|-----------|--------|-------|
| YYYY-MM-DD | 101 | YYYY-MM-DD | 2m14s | 200 | ✅ PASS | |
| | | | | | | |

---

## 3. PBS Maintenance (Monthly)

```bash
# Run garbage collection (removes orphaned chunks)
ssh root@10.2.7.65 "proxmox-backup-manager garbage-collection start main"

# Verify prune job is active
ssh root@10.2.7.65 "proxmox-backup-manager prune-job list"

# ZFS scrub (checks disk integrity)
ssh root@10.2.7.65 "zpool scrub backup"

# Check scrub status (run after scrub completes)
ssh root@10.2.7.65 "zpool status backup | grep -A1 scrub"

# Datastore verification (checksums all chunks)
ssh root@10.2.7.65 "proxmox-backup-manager verify-job list"
```

> **Note:** ZFS scrub on 2×2TB mirror takes ~2-3 hours. Schedule during low usage.

---

## 4. RTO/RPO Targets

| Tier | CTs | RPO (Backup Freq) | RTO (Restore Time) | Priority |
|------|-----|-------------------|--------------------|----------|
| **Critical** | 105 (Wazuh), 100 (Hermes) | 24h | < 15 min | 1 |
| **High** | 106 (Grafana/Docker), 107 (Pi-hole) | 24h | < 30 min | 2 |
| **Medium** | 101 (Immich), 104 (Nextcloud) | 24h | < 1 hour | 3 |
| **Low** | 102 (PiAlert), 103 (Homarr) | 48h | < 2 hours | 4 |

---

## 5. SOP Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-05-25 | Hermes | Initial SOP — derived from DR-IR-PLAYBOOK.md |