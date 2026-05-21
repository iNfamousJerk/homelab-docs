# 01 - Proxmox Server

## Hardware

- **CPU:** Intel i5-7500 @ 3.40GHz (4 cores)
- **RAM:** 31.2 GB total
- **Storage:** ~94 GB on local-lvm (LVM thin pool)
- **OS:** Proxmox Virtual Environment 9.1.1

## Networking

- **Hostname:** pve
- **Web UI:** https://10.2.7.64:8006
- **Subnet:** 10.2.7.0/24
- **Gateway:** 10.2.7.1 (OPNsense)

## API Access

The Proxmox API is accessible at:

```
POST https://10.2.7.64:8006/api2/json/access/ticket
```

Authentication payload:
```json
{
  "username": "root@pam",
  "password": "<password>"
}
```

Returns a ticket (cookie) and CSRFPreventionToken for subsequent requests.

### Useful API Commands

```bash
# List all containers
curl -k -b "PVEAuthCookie=$TICKET" \
  https://10.2.7.64:8006/api2/json/nodes/pve/lxc

# Get container config
curl -k -b "PVEAuthCookie=$TICKET" \
  https://10.2.7.64:8006/api2/json/nodes/pve/lxc/103/config

# Get node status
curl -k -b "PVEAuthCookie=$TICKET" \
  https://10.2.7.64:8006/api2/json/nodes/pve/status

# Full auth example
TICKET=$(curl -sk -d 'username=root@pam&password=YOUR_PASSWORD' \
  https://10.2.7.64:8006/api2/json/access/ticket | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['data']['ticket'])")
```

## Creating an LXC Container

This is the **exact flow** for deploying a new LXC container:

```bash
# 1️⃣ SSH into the Proxmox host
ssh root@10.2.7.64

# 2️⃣ Download a template (if you don't have one)
pveam update
pveam download local ubuntu-24.04-standard_24.04-2_amd64.tar.zst

# 3️⃣ Create the container
pct create 107 local:vztmpl/ubuntu-24.04-standard_24.04-2_amd64.tar.zst \
  --hostname my-new-service \
  --storage local-lvm \
  --rootfs local-lvm:8 \
  --memory 2048 \
  --cores 2 \
  --net0 name=eth0,bridge=vmbr0,gw=10.2.7.1,ip=10.2.7.109/24 \
  --unprivileged 1 \
  --features nesting=1,keyctl=1 \
  --onboot 1 \
  --password YOUR_CONTAINER_PASSWORD

# 4️⃣ Start it
pct start 107

# 5️⃣ SSH into the container (from the PVE host)
pct enter 107
# Or from outside PVE:
# ssh root@10.2.7.109

# 6️⃣ Update and install basics
apt update && apt upgrade -y
apt install -y curl wget git ufw
```

## Resource Usage

| Resource | Usage |
|----------|-------|
| CPU | ~3% idle |
| RAM | ~19% |
| Disk | ~14% |
| Swap | ~4% |
