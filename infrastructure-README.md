# Infrastructure

This section covers the foundational build of the homelab — from bare hardware to a fully hardened, remotely accessible, self-hosted service stack running on a Raspberry Pi 4.

The three parts are meant to be read in order. Each one builds directly on the previous.

---

## Parts

| Part | Title | Description |
|---|---|---|
| [Part 1](./part1-initial-setup-pihole-docker.md) | Initial Setup, Docker & Pi-hole | Hardware choices, OS selection, SSH key auth, UFW baseline, Docker install, Pi-hole deployment |
| [Part 2](./part2-hardening-remote-access-maintenance.md) | Hardening, Remote Access & Automatic Maintenance | SSH hardening, fail2ban, Tailscale VPN, Portainer, Watchtower, unattended-upgrades, log limits |
| [Part 3](./part3-backups-dns-proxy-dashboard-subnet.md) | Backups, DNS Routing, Proxy, Dashboard & Subnet Access | Nightly backups, Tailscale DNS via Pi-hole, Nginx Proxy Manager, friendly hostnames, Homepage dashboard, subnet routing |

---

## What Was Built

A headless Raspberry Pi 4 (8GB) running Raspberry Pi OS Lite 64-bit, serving as a self-hosted home server with the following characteristics:

**Security posture:** SSH key-only authentication, root login disabled, UFW default-deny with explicit allowlist per service, fail2ban banning repeated SSH probes, automatic OS security updates, Docker log limits to prevent SD card exhaustion.

**Remote access:** Tailscale mesh VPN running as a system service — no open ports on the router, no port forwarding, WireGuard-based encryption. Subnet routing gives full remote access to the entire home network (`192.168.0.0/24`), including the router admin panel.

**Service layer:** Nginx Proxy Manager handles reverse proxying and friendly `.home` hostnames. Pi-hole serves as the network-wide DNS filter and local DNS resolver — including for Tailscale-connected devices. Homepage dashboard provides a live overview of all services and container health.

**Maintenance:** Watchtower handles automatic Docker image updates nightly (Pi-hole excluded). Unattended-upgrades handles OS package updates. A cron-driven backup script runs nightly and writes configs and compose files to a USB drive with 30-day retention.

---

## Service Stack After Part 3

| Service | Role | Access |
|---|---|---|
| Pi-hole | DNS filtering + local DNS resolver | `pihole.home` |
| Nginx Proxy Manager | Reverse proxy + friendly hostnames | `npm.home` |
| Portainer CE | Docker management UI | `portainer.home` |
| Watchtower | Automated container image updates | — |
| Homepage | Internal dashboard | `hub.home` |
| Tailscale | Remote access (WireGuard-based) | Tailscale IP |

---

## Key Design Decisions

**Raspberry Pi OS Lite over Ubuntu Server.** Pi-optimised kernel, ~150MB idle RAM vs ~300MB for Ubuntu Server, Debian base meaning standard tooling works out of the box, and no desktop environment — every uninstalled package is a vulnerability that cannot be exploited.

**Wired ethernet only.** A server that never moves has no reason for Wi-Fi. One less radio interface is a smaller attack surface.

**Docker over bare metal installs.** If the SD card fails, a compose file restores the entire stack in minutes. Containers isolate services cleanly without the overhead of full virtualisation — important on hardware where RAM matters.

**Tailscale over self-hosted VPN.** The router's built-in L2TP/IPSec VPN was attempted and abandoned — ISP interference and firmware quirks made it unreliable. Tailscale requires zero port forwarding, works from any network, and was running in under 10 minutes. WireGuard (wg-easy) remains an option if a fully self-hosted alternative is ever needed.

**UFW Docker bridge rule.** When NPM forwards requests internally to other containers, the traffic originates from Docker's bridge network (`172.x.x.x`), not from `192.168.0.x`. Without an explicit rule for `172.16.0.0/12`, UFW silently drops this traffic — services resolve via DNS but return errors at the forwarding stage.

---

*Part of the [homelab](../README.md) project.*
