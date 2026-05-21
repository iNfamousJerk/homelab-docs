# 02 - LXC Containers

All containers run on Proxmox as LXC (Linux Containers). Most were deployed via [community-scripts](https://community-scripts.org).

## Container List

| ID | Hostname | IP | RAM | Cores | Disk | Purpose |
|----|----------|-----|-----|-------|------|---------|
| 100 | pihole | 10.2.7.2 | 2 GB | 2 | 8 GB | DNS + DHCP |
| 101 | immich | — | — | — | — | Photo backup |
| 102 | pialert | 10.2.7.103 | — | — | — | Network monitoring |
| 103 | homarr | 10.2.7.105 | 2 GB | 2 | 8 GB | Dashboard |
| 104 | nextcloud | — | — | — | — | Cloud storage |
| 105 | hermesagent | 10.2.7.107 | — | — | — | AI assistant |
| 106 | grafana | 10.2.7.108 | — | — | — | Monitoring |

## Container Features

- **Unprivileged:** Most containers are unprivileged for security
- **Nesting enabled:** containers 103+ have `nesting=1,keyctl=1` features
- **Boot order:** All set to `onboot=1` (auto-start)
- **OS:** Debian-based (Ubuntu 24.04 for grafana)

## How to Create a New Monitoring Container (CT 106)

This is the **exact process** to recreate the Grafana/Prometheus container from scratch:

### Step 1: Create the LXC Container

```bash
# SSH into Proxmox
ssh root@10.2.7.64

# Create container 106
pct create 106 local:vztmpl/ubuntu-24.04-standard_24.04-2_amd64.tar.zst \
  --hostname grafana \
  --storage local-lvm \
  --rootfs local-lvm:16 \
  --memory 2048 \
  --cores 2 \
  --net0 name=eth0,bridge=vmbr0,gw=10.2.7.1,ip=10.2.7.108/24 \
  --unprivileged 1 \
  --features nesting=1,keyctl=1 \
  --onboot 1 \
  --password GraphMyWorld

# Start it
pct start 106
```

### Step 2: Configure Container

```bash
# Enter the container
pct enter 106

# Update system
apt update && apt upgrade -y
apt install -y curl wget git openssh-server ufw ca-certificates

# Allow SSH and monitoring ports
ufw allow ssh
ufw allow 3000/tcp  # Grafana
ufw allow 9090/tcp  # Prometheus
ufw allow 9093/tcp  # Alertmanager
ufw allow 8080/tcp  # cAdvisor
ufw allow 9617/tcp  # Pi-hole exporter
ufw --force enable
```

### Step 3: Install Docker (inside container)

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Fix AppArmor for LXC
mkdir -p /etc/docker
cat > /etc/docker/daemon.json << 'EOF'
{
  "apparmor": false,
  "storage-driver": "overlay2"
}
EOF

systemctl restart docker
usermod -aG docker $USER
```

### Step 4: Install node_exporter (on PVE host)

```bash
# On the PVE host (NOT inside container)
ssh root@10.2.7.64

# Download and install
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
tar xvf node_exporter-1.8.2.linux-amd64.tar.gz
cp node_exporter-1.8.2.linux-amd64/node_exporter /usr/local/bin/
rm -rf node_exporter-1.8.2*

# Create systemd service
cat > /etc/systemd/system/node_exporter.service << 'EOF'
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=root
ExecStart=/usr/local/bin/node_exporter
Restart=always

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now node_exporter
systemctl status node_exporter  # Should show active (running)

# Repeat for CT 106:
# Install node_exporter on CT 106 too (same steps, inside the container)
```

## SSH Access

All containers configured with key-based auth from the PVE host and Hermes agent.
