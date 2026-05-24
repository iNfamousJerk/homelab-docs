# 14 - UPS Monitoring (NUT + CyberPower CP1500)

## Overview
The CyberPower CP1500 AVR UPS keeps the PVE host alive during power flickers. NUT (Network UPS Tools) monitors it and triggers a graceful shutdown when the battery runs low. Metrics flow through Prometheus into Grafana with Discord alerts.

## Architecture

```
CyberPower CP1500 AVR
       │
       └── USB ──→ PVE Host (10.2.7.64)
                       │
                       ├── NUT (nut-server + nut-monitor)
                       │   ├── upsd listens on :3493
                       │   ├── upsmon monitors battery status
                       │   └── nut-cgi web UI at /nut/upsstats.cgi
                       │
                       ├── Graceful shutdown script
                       │   └── /usr/local/bin/pve-ups-shutdown
                       │       Stops all CTs → signals UPS kill → shuts PVE
                       │
                       └── CT 106 Grafana (10.2.7.108)
                            └── nut-exporter (Docker container)
                                └── Prometheus scrapes :9999/metrics
                                    ├── Grafana dashboard
                                    └── Discord alerts via Alertmanager
```

## Setup Details

### NUT Configuration (PVE Host)

**Ups.conf** — `/etc/nut/ups.conf`
```
[cyberpower]
    driver = usbhid-ups
    port = auto
    desc = "CyberPower CP1500 AVR UPS"
```

**Nut.conf** — `/etc/nut/nut.conf`
```
MODE=netserver
```

**Upsd.conf** — `/etc/nut/upsd.conf`
```
LISTEN 127.0.0.1 3493
LISTEN 10.2.7.64 3493
```

**Upsd.users** — `/etc/nut/upsd.users`
```
[monitor]
    password = upsmonitor
    upsmon master
```

**Upsmon.conf** — `/etc/nut/upsmon.conf`
```
MONITOR cyberpower@localhost 1 monitor upsmonitor master
SHUTDOWNCMD "/usr/local/bin/pve-ups-shutdown"
POLLFREQ 5
POLLFREQALERT 1
HOSTSYNC 15
WARNTIME 10
DEADTIME 15
POWERDOWNFLAG /etc/killpower
```

**Hosts.conf** — `/etc/nut/hosts.conf` (for CGI web UI)
```
MONITOR cyberpower@localhost "CyberPower CP1500 AVR"
```

### Graceful Shutdown Script

Located at `/usr/local/bin/pve-ups-shutdown`. When triggered by upsmon (battery <10%):

1. Stops all running LXC containers gracefully (`pct shutdown --timeout 60`)
2. Waits 5 seconds
3. Signals UPS to kill power via `upsdrvctl shutdown`
4. Shuts down PVE host with 1-minute warning

Logs everything to `/var/log/pve-ups-shutdown.log`.

### NUT Web UI

**URL:** `http://10.2.7.64/nut/upsstats.cgi`

Requires Apache + nut-cgi package. Shows UPS status, battery charge, input/output voltage, load, and runtime in a table format.

### NUT Prometheus Exporter

Runs as a Docker container on CT 106 (Grafana stack). Custom Python script that queries NUT via TCP and exposes Prometheus metrics.

**Metrics endpoint:** `http://10.2.7.108:9999/metrics`

**Available metrics:**
- `nut_ups_status` — 1=On Line, 2=On Battery, 3=Low Battery
- `nut_battery_charge` — Battery percentage (0-100)
- `nut_battery_runtime` — Runtime remaining in seconds
- `nut_battery_voltage` — Battery voltage
- `nut_input_voltage` — Wall voltage
- `nut_output_voltage` — Output voltage
- `nut_ups_load` — Load percentage

### Grafana Dashboard

Provisioned dashboard: **UPS - CyberPower CP1500**

Panels:
- Status indicator (On Line / On Battery / Low Battery)
- Battery charge gauge (red <30%, green >80%)
- UPS load gauge (orange >50%, red >80%)
- Runtime countdown stat
- Battery voltage stat
- Input/Output voltage history chart
- Load history chart
- Battery charge history chart

### Discord Alerts (5 Rules)

| Alert | Condition | Severity | Description |
|-------|-----------|----------|-------------|
| UPSOnBattery | `nut_ups_status > 1` for 10s | 🔴 Critical | Power out, UPS running on battery |
| UPSBatteryLow | `nut_battery_charge < 30` for 30s | 🔴 Critical | Battery below 30% |
| UPSHighLoad | `nut_ups_load > 80` for 2m | 🟡 Warning | UPS load exceeds 80% |
| UPSLowRuntime | `nut_battery_runtime < 600` for 10s | 🔴 Critical | Less than 10 min runtime |
| NUTExporterDown | `up{job="nut"} == 0` for 2m | 🟡 Warning | Exporter unreachable |

All alerts route through Alertmanager → Discord webhook. Resolved notifications are also sent.

### Container Boot Order

After power is restored, containers start in this sequence (staggered delays prevent resource contention):

| Order | CT | Name | Delay |
|-------|----|------|-------|
| 1 | 107 | Pi-hole (DNS) | 30s |
| 2 | 106 | Grafana + Docker stack | 60s |
| 3 | 102 | PiAlert | 20s |
| 4 | 100 | Hermes Agent | 20s |
| 5 | 101 | Immich | 30s |
| 6 | 103 | Homarr | 20s |
| 7 | 105 | Wazuh | 30s |
| 8 | 104 | Nextcloud | 20s |

## Full Power Failure Flow

```
⚡ Power Out → UPS takes over
   │
   ├── Discord alert: UPSOnBattery
   ├── Battery drains...
   │   └── Discord: UPSBatteryLow at <30%
   ├── Battery critical (<10%)
   │   └── Shutdown script fires:
   │       ├── Stop all containers gracefully
   │       ├── Signal UPS to kill power
   │       └── Power off PVE host
   │
⚡ Power Returns
   ├── PVE boots (BIOS: Restore on AC Loss)
   └── Containers auto-start in order (1→8)
```

## UPS Hardware Specs

| Spec | Value |
|------|-------|
| Model | CyberPower CP1500 AVR |
| USB ID | 0764:0501 |
| Battery Type | Sealed Lead-Acid |
| Nominal Voltage | 12V |
| Input Range | 96-140V |
| Nominal Power | 375W real / 1500VA |
| Current Load | ~10% (very light) |
| Runtime | ~73 min at current load |
| NUT Driver | usbhid-ups / CyberPower HID 0.8 |

## Troubleshooting

**Can't access nut-cgi web UI:** Verify Apache is running and the CGI module is enabled.
```bash
systemctl status apache2
ls /etc/apache2/mods-enabled/cgi*
```

**Exporter not returning data:** Check the NUT connection from CT 106.
```bash
docker logs nut-exporter
echo -e "USERNAME monitor\nPASSWORD upsmonitor\nGET VAR cyberpower ups.load\n" | nc -w 3 10.2.7.64 3493
```

**UPS driver not connecting:** The kernel HID driver may have claimed the USB device.
```bash
systemctl restart nut-server
systemctl restart nut-monitor
```

**Init SSL warning:** Harmless. The local NUT connection doesn't use TLS.