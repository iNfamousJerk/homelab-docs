# 13 - Wazuh SIEM

Wazuh is a free, open-source security information and event management (SIEM) platform. It provides threat detection, integrity monitoring, incident response, and compliance management.

## Initial Setup (CT 106 — Grafana Container)

Wazuh was initially deployed on CT 106 (Grafana, 10.2.7.108) alongside the monitoring stack using the official Docker images.

### Deployment Steps

```bash
# Clone the Wazuh Docker repo with a stable release tag
git clone https://github.com/wazuh/wazuh-docker.git
cd wazuh-docker && git checkout v4.14.5

# Generate certificates manually (cert-generator Docker fails in unprivileged LXC)
mkdir -p config/wazuh_indexer_ssl_certs/

# Fix AppArmor for LXC
# Add to each service in docker-compose.yml:
#   security_opt:
#     - apparmor=unconfined
```

### Exposed Ports (on CT 106)

| Service | Port |
|---------|------|
| Wazuh Dashboard UI | `https://10.2.7.108:443` |
| Wazuh Indexer API | `https://10.2.7.108:9200` |
| Wazuh Manager API | `10.2.7.108:55000` |
| Agent enrollment | `10.2.7.108:1515` |
| Agent events | `10.2.7.108:1514` |
| Syslog | `10.2.7.108:514/udp` |

## Dedicated Container (CT 108 — Hermes-Wazuh)

A dedicated container (CT 108) was later provisioned to run Wazuh independently from the Grafana monitoring stack to avoid resource contention. It used the same deployment process but with more RAM/disk allocated.

## Full Removal (May 2026)

Wazuh has been **fully removed from the homelab** — both CT 108 (dedicated container) and the Docker deployment on CT 106 were cleaned up:

| Step | Detail |
|------|--------|
| CT 108 | Proxmox container destroyed via API |
| CT 106 Docker | 3 containers, 13 volumes, 5 images (~9 GB) removed |
| Wazuh directory | `/opt/wazuh-docker/` deleted |

Reasons:
- Resource requirements (~2.5 GB RAM for indexer + manager + dashboard) were too high for the available 32 GB host
- Agent deployment and management overhead outweighed the security monitoring benefits for this small homelab
- Existing monitoring stack (Grafana + Prometheus + PiAlert) already covers most visibility needs

## Agent Installation (for reference)

If Wazuh is re-deployed in the future, install agents on any container:

```bash
curl -s https://packages.wazuh.com/4.x/keys/GPG-KEY-WAZUH | apt-key add -
echo "deb https://packages.wazuh.com/4.x/apt/ stable main" > /etc/apt/sources.list.d/wazuh.list
apt-get update && apt-get install -y wazuh-agent
/var/ossec/bin/agent-auth -m <WAZUH_MANAGER_IP> -p 1515
sed -i 's/MANAGER_IP/<WAZUH_MANAGER_IP>/' /var/ossec/etc/ossec.conf
systemctl enable wazuh-agent && systemctl start wazuh-agent
```

## Notes

- **Wazuh is heavy** — a full Wazuh cluster (indexer + manager + dashboard) needs ~3 GB RAM minimum
- **Use stable tags only** — the `main` branch tracks 5.0.0-dev whose Docker images may not exist
- **Certificate generation** is the trickiest part in unprivileged LXC — the cert-generator container fails due to AppArmor; generate certs manually with openssl
- **Consider alternatives** for lightweight SIEM: Guardian (single-binary), or just rely on PiAlert + Grafana alerts
