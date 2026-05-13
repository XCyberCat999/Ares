# ARES Network Scanner

A lightweight network monitoring tool for home and small office environments.
ARES scans your network for devices and services, detects unknown devices,
tracks service version changes, and pushes structured logs to a Grafana/Loki stack.

**Version:** 0.1.0

---

## Features

- ARP-based device discovery via nmap (L2 and L3 support)
- Service and version scan – detect software changes
- Multi-network support via config file
- Structured JSON logging
- Direct Loki push
- Alerting via Grafana
- systemd timer support

---

## Requirements

```bash
# Debian / Ubuntu
sudo apt install nmap python3 python3-requests sqlite3
```

---

## Installation

```bash
git clone https://github.com/XCyberCat999/Ares.git
cd Ares

# Install as system command
sudo cp ares-0.1.0.py /usr/local/bin/ares
sudo chmod +x /usr/local/bin/ares
```

---

## Configuration

```bash
sudo mkdir -p /etc/ares
sudo nano /etc/ares/ares.conf
```

Example `/etc/ares/ares.conf`:

```ini
[global]
loki_url  = http://10.8.0.1:3100

[network:lan]
network   = 192.168.0.0/24
interface = eth1
mode      = arp

[network:lan2]
network   = 10.10.0.0/24
interface = eth2
mode      = arp
```

### Scan modes

| Mode | Description |
|---|---|
| `arp` | ARP scan – MAC addresses visible. Use for local L2 networks. |
| `ip` | Ping scan – no MAC addresses. Use for remote or VPN networks. |

---

## Usage

ARES requires root privileges:

```bash
sudo ares [options]
```

### First run – scan and build baseline

```bash
# Scan all networks from config
sudo ares --scan-devices

# Or scan a specific network
sudo ares --scan-devices --network 192.168.0.0/24

# Full scan including services (slow)
sudo ares --scan-all

# Review output, then freeze baseline
sudo ares --accept-baseline
```

### Regular scan

```bash
# Device scan only (fast – for systemd timer)
sudo ares --scan-devices

# Service and version scan (slow – run occasionally)
sudo ares --scan-services

# Both together
sudo ares --scan-all
```

### Status overview

```bash
# Devices and services
sudo ares --status-all

# Devices only
sudo ares --status-devices

# Services only
sudo ares --status-services

# Filter by network
sudo ares --status-all --network 192.168.2.0/24
```

### Accept devices

```bash
# Accept a single device by IP
sudo ares --accept-device 192.168.0.50

# Accept all pending devices
sudo ares --accept-baseline

# Reset entire baseline
sudo ares --reset-baseline
```

---

## CLI Reference

| Argument | Description                                             |
|---|---------------------------------------------------------|
| `--network` | Target network e.g. `192.168.0.0/24` (overrides config) |
| `--interface` | Network interface e.g. `eth1` (optional)                |
| `--mode` | Scan mode: `arp` or `ip` (default: `arp`)               |
| `--loki-url` | Loki push URL e.g. `http://10.8.0.1:3100` (optional)    |
| `--scan-devices` | Run device discovery scan (fast)                        |
| `--scan-services` | Run service and version scan (slow)                     |
| `--scan-all` | Run device scan + service scan                          |
| `--accept-baseline` | Freeze current state as baseline                        |
| `--reset-baseline` | Reset baseline – all devices back to pending            |
| `--accept-device IP` | Accept a single device by IP                            |
| `--status` | Alias for `--status-all`                                |
| `--status-devices` | Show device status table                                |
| `--status-services` | Show services status table                              |
| `--status-all` | Show devices and services tables                        |

---

## systemd Timer

Create the service unit:

```ini
# /etc/systemd/system/ares-scanner.service
[Unit]
Description=ARES Network Scanner
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/ares --scan-devices
User=root
StandardOutput=journal
StandardError=journal
```

Create the timer unit:

```ini
# /etc/systemd/system/ares-scanner.timer
[Unit]
Description=ARES Network Scanner Timer
Requires=ares-scanner.service

[Timer]
OnBootSec=2min
OnUnitActiveSec=5min
AccuracySec=30s

[Install]
WantedBy=timers.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now ares-scanner.timer
```

---

## Loki + Grafana Stack

ARES pushes structured JSON events to Loki.
Available event types:

| Event | Level | Description |
|---|---|---|
| `scan_start` | info | Scan started |
| `device_seen` | info | Device found in network |
| `scan_complete` | info | Scan finished |
| `new_device` | warn | Device not in baseline |
| `new_port` | warn | New port on known device |
| `version_change` | warn | Service version changed |

Example Grafana query:
```
{job="ares-scanner", level="warn"} | json
```

---

## Database

ARES stores all data in SQLite at `/var/lib/ares/ares.db`.

```bash
# View all devices
sqlite3 /var/lib/ares/ares.db \
    ".mode column" ".headers on" "SELECT * FROM devices;"

# View baseline
sqlite3 /var/lib/ares/ares.db \
    ".mode column" ".headers on" "SELECT * FROM baseline;"
```

Tables: `devices`, `services`, `baseline`, `events`

---

## File Locations

| Path | Description |
|---|---|
| `/usr/local/bin/ares` | Executable |
| `/etc/ares/ares.conf` | Configuration file |
| `/var/lib/ares/ares.db` | SQLite database |
| `/var/log/ares/ares.log` | JSON log file |

---

## License

MIT
