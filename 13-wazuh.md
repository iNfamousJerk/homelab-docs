# 13 - Wazuh SIEM

## Overview
Real-world use: Wazuh is a security camera system for your servers. It watches every login attempt, every file change, every running process, and every installed package. When something suspicious happens (SSH brute force, modified system binary, known vulnerability in a package), it alerts you. Think of it as a 24/7 security guard for your homelab.

## Architecture
Manager: CT 105 (10.2.7.110) — 4GB RAM, 4 cores
Agents deployed on all active containers + PVE host

| Component | Where | Purpose |
|-----------|-------|---------|
| Wazuh Manager | CT 105 (.110) | Collects and analyzes events |
| Wazuh Dashboard | https://10.2.7.110:443 | Web UI for alerts, agents, vulns |
| Wazuh Indexer | CT 105 (.110:9200) | Stores event data |
| Agent (x8) | Every CT + PVE | Ships logs, file changes, vuln data |

## Access
- Dashboard: https://10.2.7.110:443
- API: https://10.2.7.110:55000
- SSH: root@10.2.7.110

## User Manual
### View security events
Dashboard → Security Events module → filter by rule level
- Level 0-3: Info
- Level 4-7: Warning
- Level 8-10: Suspicious (SSH brute force attempts, new services)
- Level 11-14: Attack detected
- Level 15+: Severe (confirmed compromise)

### Check agent status
Dashboard → Agents module → shows all 8 agents with last keepalive
Gray = disconnected, Green = active

### View vulnerabilities
Dashboard → Vulnerabilities → shows CVEs detected on any monitored system

### Check file changes
Dashboard → Integrity Monitoring → shows what files changed and when

## Agent Deployment (for future new CTs)

> **Current Wazuh version:** 4.14.5 (latest 4.x stable line — 5.x requires clean install, see below)

```bash
# Add Wazuh repo (4.x — current stable)
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --no-tty --yes --dearmor > /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" > /etc/apt/sources.list.d/wazuh.list
apt-get update

# Install
apt-get install -y wazuh-agent

# Configure manager address
sed -i 's/MANAGER_IP/10.2.7.110/g' /var/ossec/etc/ossec.conf

# ⚠️ REQUIRED: Enroll with manager (without this, agent stays "disconnected")
/var/ossec/bin/agent-auth -m 10.2.7.110 -A $(hostname)

# Start
systemctl enable --now wazuh-agent

# Verify
/var/ossec/bin/agent_control -l
```

### Wazuh 5.x Upgrade — Future Project

Wazuh 5.0 **cannot be upgraded in-place** from 4.x. It requires a clean OS install with config migration:
1. Document all current custom rules, decoders, and configurations
2. Export agent enrollment keys
3. Perform clean OS install and deploy Wazuh 5.x
4. Re-import configurations and re-enroll all 8 agents
5. **Estimated effort:** 1 full day — schedule when you can tolerate downtime

See [Wazuh 5.0 migration guide](https://documentation.wazuh.com/5.0/upgrade-guide/migration-from-4-x.html) when ready.

## Maintenance
### Update Wazuh manager
```bash
pct enter 105
apt update && apt upgrade -y
# Check services: systemctl status wazuh-manager wazuh-indexer wazuh-dashboard
```

### Update agents
Can be done centrally via Wazuh dashboard, or per-container:
```bash
pct exec <CT_ID> -- apt update && apt upgrade -y
```

### Restart services
```bash
pct enter 105
systemctl restart wazuh-manager
```

## Logs
- Manager logs: /var/ossec/logs/ossec.log
- Agent logs: /var/ossec/logs/ossec.log inside each container
- Dashboard: check web UI events

## Troubleshooting
- Agent shows as disconnected? Check agent can reach manager: curl -k https://10.2.7.110:55000
- Agent not sending events? systemctl status wazuh-agent on the agent
- Manager not receiving? Check port 1514/udp is open on manager: ss -tulpn | grep 1514
- Too many alerts? Use dashboard filters to focus on level 8+ events only