# Monitoring Stack — Part 2: cAdvisor

**Date:** May 2026  
**Environment:** Raspberry Pi 4 (8GB) · Raspberry Pi OS Lite 64-bit · Docker

---

## Overview

This is Part 2 of an incremental observability stack built on a headless Raspberry Pi 4.
Part 1 covered Node Exporter and Prometheus — host-level hardware metrics scraped on a 15-second interval.
This session adds cAdvisor, which shifts the visibility from the host down to the individual container level.

**Component deployed this session:**
- **cAdvisor** — per-container CPU, memory, network, and disk I/O metrics, feeding directly into the existing Prometheus instance

---

## Architecture Decisions

### cAdvisor runs in Docker — intentionally

Node Exporter runs as a native systemd service because it needs direct access to host hardware to report accurate metrics. cAdvisor is the opposite — its job is to observe containers from the outside, and it does that by reading Docker's own data through mounted volumes (`/var/lib/docker`, `/sys`, `/var/run`). Running it as a Docker container is the correct approach for this purpose, and it fits naturally alongside the rest of the stack.

### Why `privileged: true` is required

cAdvisor reads cgroup and kernel data to collect per-container resource usage. On ARM (the Pi 4), it requires elevated access to do this — without it, the container either fails to start or returns incomplete metrics. This is a known requirement for cAdvisor on ARM, not a misconfiguration.

### Port 8090 instead of 8080

cAdvisor's default internal port is 8080, which is already occupied on the host by Pi-hole. Rather than reconfigure either service, the compose file maps cAdvisor's internal 8080 to host port 8090. Clean solution with no side effects.

### `--web.enable-lifecycle` added to Prometheus

Discovered during this session that Prometheus's live config reload endpoint (`/-/reload`) requires this flag to be explicitly enabled — it is off by default. The alternative is a full container restart every time `prometheus.yml` is edited, which interrupts scraping. For a homelab on a private network behind UFW, enabling the lifecycle API is the right tradeoff. The flag was added to the Prometheus `command:` block alongside the existing flags.

---

## Implementation

### 1. cAdvisor — Docker Compose

```bash
mkdir -p ~/docker/cadvisor
nano ~/docker/cadvisor/docker-compose.yml
```

**`~/docker/cadvisor/docker-compose.yml`:**

```yaml
services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    restart: unless-stopped
    privileged: true
    ports:
      - "8090:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    devices:
      - /dev/kmsg
```

```bash
cd ~/docker/cadvisor
docker compose up -d
```

**UFW rules** — cAdvisor needs to be reachable from the local network, Tailscale, and from Prometheus (which originates from the Docker bridge range):

```bash
sudo ufw allow from 192.168.0.0/24 to any port 8090
sudo ufw allow from 100.64.0.0/10 to any port 8090
sudo ufw allow from 172.16.0.0/12 to any port 8090   # Docker bridge (Prometheus scraping cAdvisor)
```

---

### 2. Add cAdvisor as a Prometheus Scrape Target

**`~/docker/prometheus/prometheus.yml`** — add the cAdvisor job:

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

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['192.168.0.220:8090']
```

The host IP (`192.168.0.220`) is used rather than the container name because Prometheus and cAdvisor are in separate compose stacks with no shared Docker network. This is consistent with how Prometheus already reaches Node Exporter.

---

### 3. Enable Prometheus Live Config Reload

`--web.enable-lifecycle` was added to the Prometheus compose file so `prometheus.yml` changes can be applied without a full container restart:

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
      - '--web.enable-lifecycle'

volumes:
  prometheus_data:
```

```bash
cd ~/docker/prometheus
docker compose down && docker compose up -d
```

Reload config after editing `prometheus.yml`:

```bash
curl -X POST http://localhost:9090/-/reload
```

---

### 4. DNS and Reverse Proxy

Consistent with the rest of the setup.

**Pi-hole local DNS:** `cadvisor.home → 192.168.0.220`

**Nginx Proxy Manager** — new proxy host:
- Domain: `cadvisor.home`
- Forward to: `http://192.168.0.220:8090`
- Websockets support: enabled
- Scheme: HTTP

**Homepage dashboard** — added tile to `services.yaml`:

```yaml
- cAdvisor:
    href: http://cadvisor.home
    description: Container metrics
    icon: docker.png
```

---

## Verification

```bash
# Confirm cAdvisor is running
docker ps | grep cadvisor

# Confirm metrics endpoint is responding
curl -s http://localhost:8090/metrics | head -20
```

**Prometheus → Status → Targets:** all three jobs should show **State: UP**:

| Job | Target | Expected State |
|---|---|---|
| prometheus | localhost:9090 | UP |
| node_exporter | 192.168.0.220:9100 | UP |
| cadvisor | 192.168.0.220:8090 | UP |

**Test query in Prometheus UI** (`http://prometheus.home` → Graph):

```promql
container_memory_usage_bytes{name="prometheus"}
```

Returns the current memory usage of the Prometheus container itself. A result confirms cAdvisor is collecting data and Prometheus is storing it.

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
| cAdvisor | Per-container metrics | `cadvisor.home` |
| Tailscale | Remote access (WireGuard-based) | Tailscale IP |

---

## What's Next

| Stage | Component | Purpose |
|---|---|---|
| ✅ Part 1 | Node Exporter + Prometheus | Hardware metrics collection and storage |
| ✅ Part 2 | cAdvisor | Per-container CPU, memory, and network metrics |
| ⏳ Part 3 | Grafana | Dashboards and visualisation |
| ⏳ Part 4 | Loki + Promtail | Log aggregation — correlate metrics and logs in one view |
| ⏳ Part 5 | Alerts + polish | Alerting rules, custom panels, final tuning |

---

*Part of an ongoing homelab build documented in [HectorP001/homelab](https://github.com/HectorP001/homelab).*
