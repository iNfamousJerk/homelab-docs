# 01 - Proxmox VE

## Overview
Real-world use: Proxmox is the hypervisor that runs ALL containers. Think of it as the landlord of the apartment building — every container lives inside it. Real-world use: running multiple isolated services on one physical machine, ability to snapshot/backup/restore any container, hot-resize resources.

## Quick Reference
- Web UI: https://10.2.7.64:8006
- SSH: root@10.2.7.64 (password: 2proxtheworld)
- API: https://10.2.7.64:8006/api2/json
- Hostname: pve

## User Manual
- How to access web UI (browser → https://10.2.7.64:8006)
- How to check container list (pct list from SSH)
- How to SSH into a container (pct enter <CT_ID>)
- How to view logs (pct logs <CT_ID>)
- How to check resource usage (Web UI → Node → Summary)

## Maintenance
### Update Proxmox itself
```bash
ssh root@10.2.7.64
apt update && apt dist-upgrade -y
```
Then reboot if kernel was updated.

### Resize a container disk
```bash
pct resize <CT_ID> rootfs <new_size>G
# Then expand filesystem inside:
pct enter <CT_ID>
resize2fs /dev/sda1  # for ext4
# Or reboot the container
```

### Snapshot before risky changes
```bash
pct snapshot <CT_ID> before-update --description "Before XYZ change"
# Rollback: pct rollback <CT_ID> before-update
```

### Backup container
Use PBS integration for automated backups, or manual vzdump.

## Helper Scripts
List the Proxmox API helper script that was used for PiAlert resize:
```bash
# Auth script from Hermes:
TICKET=$(curl -sk -d 'username=root@pam&password=2proxtheworld' https://10.2.7.64:8006/api2/json/access/ticket | python3 -c "import sys,json; print(json.load(sys.stdin)['data']['ticket'])")
CSRF=$(curl -sk -d 'username=root@pam&password=2proxtheworld' https://10.2.7.64:8006/api2/json/access/ticket | python3 -c "import sys,json; print(json.load(sys.stdin)['data']['CSRFPreventionToken'])")
```

## Logs & Monitoring
- System logs: /var/log/syslog, /var/log/pve*
- Container logs: pct logs <CT_ID>
- VM logs: qm logs <VM_ID>
- Grafana dashboard monitors: CPU, RAM, disk via node_exporter (installed on host)

## Troubleshooting
- Can't reach Web UI? Check if pveproxy is running: systemctl status pveproxy
- Container won't start? pct start <CT_ID> --debug for verbose output
- Disk full? Check: df -h, then pct resize if needed