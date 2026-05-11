# Monitoring Stack — Part 1: Node Exporter + Prometheus

**Date:** May 11, 2026  
**Environment:** Raspberry Pi 4 (8GB) · Raspberry Pi OS Lite 64-bit · Docker

---

## Overview

This session covers the first stage of a full observability stack built on a headless Raspberry Pi 4.
The goal was to get end-to-end metrics collection working: hardware stats captured at the OS level, scraped on a schedule by a time-series database, and accessible from the internal dashboard.

**Components deployed this session:**
- **Node Exporter** — host metrics exporter, running as a native systemd service
- **Prometheus** — metrics collection and storage, running as a Docker container

---

## Architecture Decisions

### Node Exporter runs outside Docker — intentionally

Node Exporter is installed directly on the OS rather than as a container. A containerised Node Exporter sees a filtered view of the host: filesystem stats, network interfaces, and certain CPU metrics can be incomplete or misleading depending on volume mounts and cgroup isolation. Running it natively gives Prometheus accurate data about the actual hardware.

The tradeoff is a slightly more involved setup — binary install, dedicated system user, and a systemd service file — compared to a single compose entry. For a tool whose entire purpose is to accurately represent the host, that tradeoff is worth it.

### Dedicated `node_exporter` system user

Standard hardening practice for any long-running service. The user has no home directory, no login shell, and ownership only over the binary it needs. If Node Exporter were ever compromised, the blast radius is contained. Same principle as running web servers under `www-data`.

### Prometheus data retention set to 15 days

Longer retention would accumulate significant write volume on an SD card over time. 15 days is enough to observe trends and debug incidents. This setting is revisable if the Pi moves to a USB SSD — a planned future upgrade.

### HTTP (not HTTPS) for internal services

All `.home` hostnames resolve only inside the local network or over Tailscale. Tailscale provides end-to-end WireGuard encryption for remote access, so traffic to internal services is already encrypted at the tunnel level. Forcing HTTPS on `.home` domains would require either self-signed certificates (browser warnings on every device) or a DNS-01 challenge with a real public domain — unnecessary complexity with no practical security gain in this context.

HTTPS via Let's Encrypt is relevant for anything exposed publicly, such as a future Tailscale Funnel setup. For everything behind the `.home` boundary, HTTP is the correct and deliberate choice.

---

## Implementation

### 1. Node Exporter — Install and Harden

```bash
# Create dedicated unprivileged system user
sudo useradd --no-create-home --shell /bin/false node_exporter

# Download and install binary
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-arm64.tar.gz
tar xvf node_exporter-1.8.2.linux-arm64.tar.gz
sudo cp node_exporter-1.8.2.linux-arm64/node_exporter /usr/local/bin/
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

**systemd service** — `/etc/systemd/system/node_exporter.service`:

```ini
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
```

**UFW rules** — Prometheus runs in Docker, so its traffic to Node Exporter originates from the Docker bridge network (`172.x.x.x`), not from `192.168.0.x`. Both ranges need to be allowed:

```bash
sudo ufw allow from 192.168.0.0/24 to any port 9100
sudo ufw allow from 172.16.0.0/12 to any port 9100   # Docker bridge networks
```

---

### 2. Prometheus — Docker Compose

**`~/docker/prometheus/docker-compose.yml`:**

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'

volumes:
  prometheus_data:
```

**`~/docker/prometheus/prometheus.yml`:**

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['192.168.0.220:9100']
```

**UFW rules for Prometheus UI:**

```bash
sudo ufw allow from 192.168.0.0/24 to any port 9090
sudo ufw allow from 100.64.0.0/10 to any port 9090   # Tailscale range
```

---

### 3. DNS and Reverse Proxy

Consistent with the rest of the setup — every service gets a friendly `.home` hostname.

**Pi-hole local DNS:** `prometheus.home → 192.168.0.220`

**Nginx Proxy Manager** — new proxy host:
- Domain: `prometheus.home`
- Forward to: `http://192.168.0.220:9090`
- Scheme: HTTP (see architecture decision above)

**Homepage dashboard** — added tile to `services.yaml`:

```yaml
- Prometheus:
    href: http://prometheus.home
    description: Metrics collection
    icon: prometheus.png
```

---

## Verification

```bash
# Node Exporter metrics endpoint
curl http://192.168.0.220:9100/metrics | head -20

# Prometheus self-check
curl http://192.168.0.220:9090/-/healthy
```

**Prometheus → Status → Targets:** both `prometheus` and `node_exporter` showing **State: UP**.

**Test query in Prometheus UI:**
```promql
node_cpu_seconds_total
```
Returns live per-core CPU time-series data.

---

## Current Service Stack

| Service | Role | Access |
|---|---|---|
| Pi-hole | DNS filtering + local DNS | `pihole.home` |
| Nginx Proxy Manager | Reverse proxy + friendly hostnames | `npm.home` |
| Portainer CE | Docker management UI | `portainer.home` |
| Watchtower | Automated container updates | — |
| Homepage | Internal dashboard | `hub.home` |
| Node Exporter | Host metrics exporter | port 9100 (internal) |
| Prometheus | Metrics collection + storage | `prometheus.home` |
| Tailscale | Remote access (WireGuard-based) | Tailscale IP |

---

## What's Next

The monitoring stack is being built incrementally:

| Stage | Component | Purpose |
|---|---|---|
| ✅ Part 1 | Node Exporter + Prometheus | Hardware metrics collection and storage |
| ⏳ Part 2 | cAdvisor | Per-container CPU, memory, and network metrics |
| ⏳ Part 3 | Grafana | Dashboards and visualisation |
| ⏳ Part 4 | Loki + Promtail | Log aggregation — correlate metrics and logs in one view |
| ⏳ Part 5 | Alerts + polish | Alerting rules, custom panels, final tuning |

---

*Part of an ongoing homelab build documented in [HectorP001/homelab](https://github.com/HectorP001/homelab).*
