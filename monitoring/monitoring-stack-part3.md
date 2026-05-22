# Monitoring Stack — Part 3: Grafana

**Date:** May 22, 2026  
**Environment:** Raspberry Pi 4 (8GB) · Raspberry Pi OS Lite 64-bit · Docker

---

## Overview

This is Part 3 of an incremental observability stack built on a headless Raspberry Pi 4.
Parts 1 and 2 covered Node Exporter, Prometheus, and cAdvisor — metrics are being collected and stored. This session adds Grafana, which turns that raw data into usable dashboards.

**Component deployed this session:**
- **Grafana** — visualisation layer, connected to Prometheus as a data source, with two community dashboards imported for host and container metrics

---

## Architecture Decisions

### Port 3001 instead of 3000

Grafana's default internal port is 3000, which is already occupied on the host by Homepage. Rather than reconfigure either service, Grafana's internal 3000 is mapped to host port 3001. This is the same pattern used throughout the stack — cAdvisor's internal 8080 maps to host 8090 because Pi-hole already owns 8080. Port conflicts are resolved at the host mapping level, not by reconfiguring existing services.

### Named Docker volume for persistent data

Grafana stores dashboards, data source configuration, and user settings in `/var/lib/grafana`. Using a named volume (`grafana_data`) means this persists across container restarts and image updates. Watchtower can pull a new Grafana image without losing any configuration.

---

## Implementation

### 1. Grafana — Docker Compose

```bash
mkdir -p ~/docker/grafana
nano ~/docker/grafana/docker-compose.yml
```

**`~/docker/grafana/docker-compose.yml`:**

```yaml
services:
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3001:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SERVER_ROOT_URL=http://grafana.home
      - TZ=Europe/Stockholm

volumes:
  grafana_data:
```

```bash
cd ~/docker/grafana
docker compose up -d
```

**UFW rules** — Grafana needs to be reachable from the local network, Tailscale, and from other Docker containers via the bridge range:

```bash
sudo ufw allow from 192.168.0.0/24 to any port 3001
sudo ufw allow from 100.64.0.0/10 to any port 3001   # Tailscale
sudo ufw allow from 172.16.0.0/12 to any port 3001   # Docker bridge
```

---

### 2. Connect Prometheus as a Data Source

In the Grafana UI:

1. Hamburger menu → **Connections** → **Data sources** → **Add data source**
2. Select **Prometheus**
3. URL: `http://192.168.0.220:9090`
4. **Save & test** — expected result: green banner confirming the Prometheus API responded

The host IP is used rather than a container hostname because Prometheus and Grafana are in separate compose stacks with no shared Docker network — consistent with how Prometheus already reaches Node Exporter and cAdvisor.

---

### 3. Import Community Dashboards

Both dashboards are imported directly from [grafana.com](https://grafana.com/grafana/dashboards/) using their IDs. No manual PromQL required — they are pre-built for exactly these exporters.

**Dashboards → New → Import → enter ID → select Prometheus data source → Import**

| Dashboard ID | Name | Coverage |
|---|---|---|
| 1860 | Node Exporter Full | CPU, RAM, disk I/O, network throughput, load average, filesystem usage |
| 14282 | cAdvisor exporter | Per-container CPU, memory, network I/O broken out by container name |

---

### 4. DNS and Reverse Proxy

Consistent with the rest of the setup.

**Pi-hole local DNS:** `grafana.home → 192.168.0.220`

**Nginx Proxy Manager** — new proxy host:
- Domain: `grafana.home`
- Forward to: `http://192.168.0.220:3001`
- Scheme: HTTP

**Homepage dashboard** — added tile to `services.yaml`:

```yaml
- Grafana:
    href: http://grafana.home
    description: Metrics visualisation
    icon: grafana.png
```

---

## Troubleshooting Notes

### cAdvisor dashboard showing "No data" on first load

Dashboard 14282 has Host and Container variable dropdowns at the top of the page. On first import these are unset, which causes all panels to return empty — the queries run but filter against no value. Setting both dropdowns to **All** populated every panel immediately. Worth checking variable dropdowns before assuming a data source or scrape problem.

### Memory panels showing 0 B on ARM

cAdvisor does not expose certain memory metrics on ARM architecture. Memory panels in dashboard 14282 show 0 B for all containers. This is a known ARM/cAdvisor limitation, not a misconfiguration. CPU, network I/O, and per-container breakdown work correctly.

### Homepage `services.yaml` format

The correct entry format uses the service name as the YAML key, not as a `name:` value:

```yaml
# Correct
- Grafana:
    href: http://grafana.home
    description: Metrics visualisation
    icon: grafana.png

# Incorrect — causes a bad indentation error
- name: Grafana
    href: http://grafana.home
```

---

## Verification

```bash
# Confirm Grafana is running
docker ps | grep grafana

# Check logs for startup errors
docker logs grafana --tail 30
```

**Grafana → Connections → Data sources → Prometheus:** green status confirming data source is connected.

**Node Exporter Full dashboard (1860):** CPU, memory, disk, and network panels populated with live data.

**cAdvisor dashboard (14282):** CPU panel broken out per container — all running Docker services visible by name with real-time usage.

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
| Grafana | Dashboards and visualisation | `grafana.home` |
| Tailscale | Remote access (WireGuard-based) | Tailscale IP |

---

## What's Next

| Stage | Component | Purpose |
|---|---|---|
| ✅ Part 1 | Node Exporter + Prometheus | Hardware metrics collection and storage |
| ✅ Part 2 | cAdvisor | Per-container CPU, memory, and network metrics |
| ✅ Part 3 | Grafana | Dashboards and visualisation |
| ⏳ Part 4 | Loki + Promtail | Log aggregation — correlate metrics and logs in one view |
| ⏳ Part 5 | Alerts + polish | Alerting rules, custom panels, final tuning |

The practical payoff of adding Loki: when a CPU spike appears on the Node Exporter dashboard, it will be possible to pull up the logs from that exact time window in the same Grafana interface — without switching tools or SSHing into the Pi.

---

*Part of an ongoing homelab build documented in [HectorP001/homelab](https://github.com/HectorP001/homelab).*
