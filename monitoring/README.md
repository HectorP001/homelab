# Monitoring Stack

An incremental observability stack built on a headless Raspberry Pi 4 (8GB), using industry-standard open source tooling. The goal is full-stack visibility: hardware metrics, per-container metrics, log aggregation, and correlated dashboards — all self-hosted.

---

## Stack Overview

| Component | Role | Status |
|---|---|---|
| Node Exporter | Host hardware and OS metrics |  Complete |
| Prometheus | Metrics scraping and time-series storage |  Complete |
| cAdvisor | Per-container CPU, memory, and network metrics |  Planned |
| Grafana | Dashboards and visualisation |  Planned |
| Loki | Log aggregation and storage |  Planned |
| Promtail | Log shipping agent |  Planned |

---

## Sessions

| Part | Components | Description |
|---|---|---|
| [Part 1](./part1-node-exporter-prometheus.md) | Node Exporter + Prometheus | Host metrics collection, scrape config, UFW rules, reverse proxy setup |
| Part 2 | cAdvisor | Per-container metrics feeding into Prometheus |
| Part 3 | Grafana | Connecting to Prometheus, building dashboards |
| Part 4 | Loki + Promtail | Log aggregation — correlate a CPU spike with logs from the same time window |
| Part 5 | Alerts + polish | Alerting rules, custom panels, final tuning |

---

## Design Principles

**Node Exporter runs outside Docker.** A containerised Node Exporter sees a filtered view of the host — filesystem, network interface, and CPU metrics can be incomplete depending on volume mounts and cgroup isolation. Running it natively as a systemd service gives Prometheus an accurate picture of the actual hardware.

**HTTP for internal services.** All `.home` hostnames resolve only inside the local network or over Tailscale. Tailscale provides end-to-end WireGuard encryption for remote access, so traffic is already encrypted at the tunnel level. Self-signed certificates would add browser warnings on every device with no security gain. HTTPS via Let's Encrypt applies to anything publicly exposed — not to internal services.

**15-day Prometheus retention.** Chosen to limit SD card write volume. Revisable once the Pi migrates to a USB SSD.

---

*Part of the [homelab](../README.md) project.*
