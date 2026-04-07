# Homelab: Security-Focused Infrastructure

A documented technical environment for testing network security, self-hosting, and systems administration. This lab serves as the primary practical application for my studies in IT Security (Vocational Higher Education, Sweden).

## Mission Statement

The objective of this project is to maintain a high-availability, secure, and observable home infrastructure. Rather than simply "making services work," this lab is built with a security-first mindset—applying industry-standard hardening, automated maintenance, and zero-trust principles to a consumer-grade environment.

This project serves as a living portfolio to demonstrate the bridge between theoretical security concepts and practical systems implementation.

## Security Philosophy

### Hardening by Default
The system is built on a "Default Deny" posture. Every layer of the stack—from the Raspberry Pi OS Lite (chosen for its minimal attack surface) to the UFW firewall rules—is configured to allow only the minimum necessary traffic. Identity is strictly managed via SSH ed25519 key-only authentication, with password logins entirely disabled to eliminate brute-force vectors.

### Zero-Trust Remote Access
The lab avoids traditional port forwarding. Instead, I utilize Tailscale to create an encrypted mesh network. This allows for secure, remote management of the infrastructure without exposing any services to the public internet, ensuring that my administrative interfaces remain invisible to external scanners.

### Resilience through Automation
Security is not a one-time setup, but an ongoing process. This lab utilizes automated workflows to maintain system integrity:
* **Automated Patching**: unattended-upgrades handles OS security patches daily.
* **Lifecycle Management**: Watchtower ensures container images are kept up to date.
* **Redundancy**: Daily automated backups to external media ensure the entire stack can be restored in minutes in the event of hardware or SD card failure.

## Roadmap and Learning Objectives

This lab is a continuous work in progress. My current focus areas include:
* **Observability**: Moving from basic logging to active monitoring and log aggregation (Prometheus, Grafana, and Loki).
* **Defensive Depth**: Implementation of Intrusion Detection Systems (Snort) and Honeypots (Cowrie) to analyze real-world attack patterns.
* **Compliance Auditing**: Benchmarking the environment against CIS (Center for Internet Security) Level 1 standards for Debian.

---
*Built for technical development and hardened for production-grade security.*
