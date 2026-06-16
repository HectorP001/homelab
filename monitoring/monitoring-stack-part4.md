# Monitoring Stack — Part 4: Loki + Promtail

**Date:** June 16, 2026  
**Environment:** Raspberry Pi 4 (8GB) · Raspberry Pi OS Lite 64-bit · Docker

---

## Overview

This is Part 4 of an incremental observability stack built on a headless Raspberry Pi 4.
Parts 1 through 3 covered Node Exporter, Prometheus, cAdvisor, and Grafana — metrics are collected, stored, and visualised. This session adds the log layer.

**Components deployed this session:**
- **Loki** — log aggregation and storage, running as a Docker container
- **Promtail** — log shipper, reads host and container logs and forwards them to Loki

With this in place, Grafana now has two data sources: Prometheus for metrics and Loki for logs. Both are queryable from the same Explore view.

---

## Architecture Decisions

### Why Loki over alternatives

Loki integrates natively with Grafana, which is already the visualisation layer for this stack. The practical benefit is being able to correlate a CPU spike on the Node Exporter dashboard with logs from that exact time window without switching tools. Loki indexes only log labels rather than the full log content, which keeps resource usage low — appropriate for a Pi on an SD card.

### Shared Docker network between Loki and Promtail

Loki and Promtail live in separate compose stacks under separate directories, consistent with the rest of the homelab. Because Promtail ships logs to Loki using the container hostname (`loki:3100`), both containers need to be on a shared named Docker network. Loki's compose file creates the network (`loki_network`), Promtail's compose joins it as `external: true`. Loki must be started first.

### Log retention set to 7 days

168 hours of retention is a deliberate conservative choice for an SD card. The same reasoning applies as with Prometheus retention (15 days) — write volume on flash storage matters for longevity. Both values are revisable if the Pi moves to a USB SSD, which is a planned future upgrade.

### Grafana data source uses LAN IP, not container hostname

Grafana and Loki are on separate Docker networks from separate sessions. Using the Pi's LAN IP (`192.168.0.220:3100`) to connect them is the clean solution without restructuring any existing network configuration.

### Promtail mounts host log directories read-only

Promtail reads from `/var/log` (host system logs) and `/var/lib/docker/containers` (Docker JSON logs) via read-only bind mounts. No write access is needed — Promtail only reads and ships.

---

## Implementation

### 1. Loki Config

**`~/docker/loki/loki-config.yml`:**

```yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  instance_addr: 127.0.0.1
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 100

schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

limits_config:
  retention_period: 168h

compactor:
  working_directory: /loki/compactor
  compaction_interval: 10m
  retention_enabled: true
  retention_delete_delay: 2h
  retention_delete_worker_count: 150
  delete_request_store: filesystem
```

### 2. Loki — Docker Compose

**`~/docker/loki/docker-compose.yml`:**

```yaml
services:
  loki:
    image: grafana/loki:latest
    container_name: loki
    restart: unless-stopped
    ports:
      - "3100:3100"
    volumes:
      - ./loki-config.yml:/etc/loki/local-config.yaml
      - loki_data:/loki
    command: -config.file=/etc/loki/local-config.yaml
    networks:
      - loki_network

volumes:
  loki_data:

networks:
  loki_network:
    name: loki_network
    driver: bridge
```

### 3. Promtail Config

**`~/docker/promtail/promtail-config.yml`:**

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          host: homelab
          __path__: /var/log/*log

  - job_name: docker_containers
    static_configs:
      - targets:
          - localhost
        labels:
          job: docker
          host: homelab
          __path__: /var/lib/docker/containers/*/*-json.log
    pipeline_stages:
      - json:
          expressions:
            output: log
            stream: stream
            attrs:
      - json:
          expressions:
            tag: attrs.tag
          source: attrs
      - regex:
          expression: (?P<container_name>(?:[^|]*[^ |]))
          source: tag
      - timestamp:
          format: RFC3339Nano
          source: time
      - labels:
          stream:
          container_name:
      - output:
          source: output
```

### 4. Promtail — Docker Compose

**`~/docker/promtail/docker-compose.yml`:**

```yaml
services:
  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    restart: unless-stopped
    volumes:
      - ./promtail-config.yml:/etc/promtail/config.yml
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    command: -config.file=/etc/promtail/config.yml
    networks:
      - loki_network

networks:
  loki_network:
    external: true
```

### 5. UFW Rules

```bash
sudo ufw allow from 192.168.0.0/24 to any port 3100 proto tcp
sudo ufw allow from 100.64.0.0/10 to any port 3100 proto tcp
sudo ufw allow from 172.16.0.0/12 to any port 3100 proto tcp
sudo ufw reload
```

### 6. Grafana — Add Loki Data Source

Grafana → Connections → Data sources → Add data source → Loki  
URL: `http://192.168.0.220:3100`  
Save & test — green confirmation.

### 7. DNS and Reverse Proxy

**Pi-hole local DNS:** `loki.home → 192.168.0.220`

**Nginx Proxy Manager:**
- Domain: `loki.home`
- Forward to: `http://192.168.0.220:3100`
- Scheme: HTTP

**Homepage dashboard** — added tile to `services.yaml`:

```yaml
- Loki:
    href: http://loki.home
    description: Log aggregation
    icon: loki.png
```

---

## Verification

```bash
# Loki ready check
curl http://192.168.0.220:3100/ready
# Expected: ready

# Container status
docker ps | grep -E "loki|promtail"
```

**Grafana → Explore → Loki → `{job="varlogs"}`:** host system logs flowing.

**Grafana → Explore → Loki → `{job="docker"}`:** per-container logs flowing, filterable by `container_name`.

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
| Loki | Log aggregation and storage | `loki.home` |
| Promtail | Log shipper | internal only |
| Tailscale | Remote access (WireGuard-based) | Tailscale IP |

---

## What's Next

| Stage | Component | Purpose |
|---|---|---|
| ✅ Part 1 | Node Exporter + Prometheus | Hardware metrics collection and storage |
| ✅ Part 2 | cAdvisor | Per-container CPU, memory, and network metrics |
| ✅ Part 3 | Grafana | Dashboards and visualisation |
| ✅ Part 4 | Loki + Promtail | Log aggregation and correlation with metrics |
| ⏳ Part 5 | Alerts + polish | Alerting rules, custom panels, final tuning |

---

*Part of an ongoing homelab build documented in [HectorP001/homelab](https://github.com/HectorP001/homelab).*
