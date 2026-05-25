# 🌐 Homelab Network Topology & IPAM

> **Purpose:** Single source of truth for IP assignments, VLAN structure, and physical cabling.
> **Last Verified:** 2026-05-25
> **Related:** `03-network.md` (operational), `SOC-UPGRADE-PLAN.md` (target state)

---

## 1. Logical Topology (Current — Flat)

```
┌─ WAN (192.168.0.171/24) ──────────────────────────────────────┐
│                            │                                    │
│                     OPNsense (10.2.7.1)                         │
│                     ┌──── Firewall ────┐                       │
│                     │  DHCP / NAT / DNS │                       │
│                     └────────┬─────────┘                       │
│                              │                                  │
│                    vmbr0 (10.2.7.0/24)                          │
│                              │                                  │
│   ┌────┬────┬────┬────┬────┬┴┬────┬────┬────┬────┬────┐      │
│  100  101  102  103  104  105 106  107  PVE  PBS  .1          │
│ Her  Immich PiA  Homr NxtC Waz  Graf PiH  .64  .65  OPN       │
│ .107 .44  .77  .105 .99 .110 .108 .2                          │
└────────────────────────────────────────────────────────────────┘
```

## 2. Target Logical Topology (Planned — Segmented)

```
See SOC-UPGRADE-PLAN.md §1 for the target VLAN architecture.
Summary:
  VLAN 10 (Mgmt):    10.2.10.0/24 — PVE, PBS, Pi-hole, OPNsense
  VLAN 20 (Core):    10.2.20.0/24 — Immich, Nextcloud, Homarr, PiAlert, Gitea
  VLAN 30 (Security):10.2.30.0/24 — Wazuh, Grafana, Prometheus, Uptime Kuma
  VLAN 40 (DMZ):     10.2.40.0/24 — Jellyfin, Hermes Agent, Reverse Proxy
```

## 3. IP Address Management (IPAM) Table

| Hostname | CT ID | IP Address | VLAN (Current) | VLAN (Target) | MAC Addr | Services | Notes |
|----------|-------|------------|----------------|---------------|----------|----------|-------|
| OPNsense | — | 10.2.7.1 | 1 (flat) | 10 | `xx:xx:xx:xx:xx:01` | Router, Firewall, DHCP | Zimaboard 2 planned |
| Pi-hole | 107 | 10.2.7.2 | 1 (flat) | 10 | | DNS, DHCP | Tailscale node |
| PVE | — | 10.2.7.64 | 1 (flat) | 10 | | Hypervisor | i7-6700, 64GB |
| PBS | — | 10.2.7.65 | 1 (flat) | 10 | | Backup Server | Z230, 2×2TB ZFS mirror |
| PiAlert | 102 | 10.2.7.77 | 1 (flat) | 20 | | Network monitoring | |
| Immich | 101 | 10.2.7.44 | 1 (flat) | 20 | | Photo backup | |
| Nextcloud | 104 | 10.2.7.99 | 1 (flat) | 20 | | File sync | |
| Homarr | 103 | 10.2.7.105 | 1 (flat) | 20 | | Dashboard | |
| Grafana Stack | 106 | 10.2.7.108 | 1 (flat) | 30 | | Grafana, Prometheus, Docker, Gitea | 10 Docker containers |
| Wazuh | 105 | 10.2.7.110 | 1 (flat) | 30 | | SIEM Manager | All-in-one node |
| Hermes Agent | 100 | 10.2.7.107 | 1 (flat) | 40 | | AI agent | API at :8642 |

## 4. Physical Topology

```
ISP Modem
   │
   └── OPNsense (ex-Optiplex or Zimaboard 2)
          │
          ├── PVE Host (HP ProDesk / custom build)
          │     ├── CTs 100-107 (on 1TB PNY SSD, LVM-thin)
          │     └── USB: CyberPower CP1500 UPS
          │
          ├── PBS (HP Z230)
          │     ├── 256GB SSD (OS)
          │     └── 2×2TB Seagate (ZFS mirror — /backup/datastore)
          │
          └── [Planned] Managed Switch
                ├── PVE2 (media)
                ├── Ripper (ex-OPNsense Optiplex)
                └── WiFi AP (IoT & Guest)
```

## 5. Firewall Rules (OPNsense)

| # | Direction | Source | Destination | Protocol:Port | Purpose | Status |
|---|-----------|--------|-------------|---------------|---------|--------|
| 1 | LAN→WAN | `10.2.7.0/24` | Any | ANY | Internet access | ✅ Active |
| 2 | LAN→LAN | `10.2.7.0/24` | `10.2.7.0/24` | ANY | Inter-CT traffic | ✅ Active |
| 3 | WAN→LAN | Any | `10.2.7.0/24` | **DENY ALL** | No inbound | ✅ Active |
| 4 | SSH | `Tailscale` | CTs | TCP:22 | Remote management | ✅ Active |

## 6. DHCP Scope

| Scope | Subnet | Router | DNS | Lease Time | Status |
|-------|--------|--------|-----|------------|--------|
| LAN | 10.2.7.100–250 | 10.2.7.1 | 10.2.7.2 | 24h | ✅ Active (Pi-hole) |
| Static leases | All CTs via Pi-hole MAC binding | — | — | — | ✅ Configured |

## 7. DNS Configuration

- **Primary DNS (all CTs):** 10.2.7.2 (Pi-hole)
- **Pi-hole upstream:** Cloudflare 1.1.1.1 / 1.0.0.1
- **Known issue:** Fresh PVE CTs get `/etc/resolv.conf` = `10.2.7.209` (old DNS). Fix: `echo 'nameserver 10.2.7.2' > /etc/resolv.conf`
- **Tailscale:** MagicDNS enabled via CT 107 (Pi-hole)
