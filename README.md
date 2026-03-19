# Homelab — Day 1: Raspberry Pi 4 + Docker + Pi-hole

> **This is a work in progress.** The configuration documented here represents a starting point, not a fully hardened system. Security work is ongoing and will be documented continuously in this repository.

A documented first step into homelabbing with a focus on performance, stability and network security — as part of my studies in IT security (Vocational Higher Education, Sweden).

---

## Hardware

| Component | Choice | Reason |
|---|---|---|
| Computer | Raspberry Pi 4 (4GB RAM) | Low power consumption, sufficient performance for home use |
| Cooling | Cooler Master Pi Case 40 | Passive cooling via aluminium shell — no fan, no noise, nothing mechanical to fail |
| Storage | Samsung 64GB SD card | Good write endurance for 24/7 operation |
| Network | Wired ethernet | More stable than Wi-Fi and one less attack surface — Wi-Fi is unnecessary exposure for a server that never moves |

---

## Operating System

**Raspberry Pi OS Lite 64-bit** was chosen over alternatives such as Ubuntu Server and Debian for the following reasons:

- Pi-optimised kernel provides better hardware support and lower memory usage (~150–180 MB idle vs ~280–350 MB for Ubuntu Server)
- No desktop environment (headless) — every package that isn't installed is a vulnerability that can't be exploited
- Debian base means CIS Debian benchmarks and standard tools (ufw, fail2ban, unattended-upgrades) work out of the box
- 64-bit required for full access to 4GB RAM and ARM64 Docker images

---

## Initial Configuration

### Raspberry Pi Imager
Everything was configured before first boot using Raspberry Pi Imager v2.0 — the Pi started directly in a known state without needing a screen or keyboard:

- Hostname set
- SSH enabled
- Custom username — the default user `pi` is not used because it is the first username automated attacks try
- Ethernet only — no Wi-Fi configured

### IP Reservation
A DHCP reservation was configured in the router admin panel (D-Link R15) based on the Pi's MAC address. No static IP settings were made on the Pi itself — this keeps all network configuration in one place and makes it easy to change without touching the server.

---

## Hardening

> The hardening below is an initial baseline, not a complete security implementation. Next steps include fail2ban, automatic security updates, further SSH hardening, and potentially CIS Benchmark Level 1 for Debian. See [Next Steps](#next-steps).

### SSH Keys Instead of Passwords

Password-based SSH login is vulnerable to brute force attacks — automated tools can test thousands of password combinations per second. Ed25519 keys solve this by requiring the private key for authentication, which never leaves the local machine.

```bash
# On local machine (Windows PowerShell)
ssh-keygen -t ed25519 -C "homelab"
cat ~/.ssh/id_ed25519.pub
# Copy the output

# On the Pi — paste the public key
mkdir -p ~/.ssh
nano ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

The file permissions (700/600) are not just convention — SSH will refuse to use `authorized_keys` if the file is readable by other users.

### Disable Password Login

Password login was only disabled after confirming that key authentication was working. The order matters — doing it the wrong way around locks you out.

```bash
sudo nano /etc/ssh/sshd_config
# Change:
# #PasswordAuthentication yes
# To:
PasswordAuthentication no

sudo systemctl restart ssh
```

### Firewall (ufw)

Default deny incoming means all inbound traffic is blocked unless explicitly allowed. This is a better starting point than default allow because new services are not automatically exposed if you forget to close a port.

```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh    # Port 22
sudo ufw allow 53     # DNS — Pi-hole
sudo ufw allow 80     # HTTP — Pi-hole web interface
sudo ufw enable
sudo ufw status
```

Current state — all inbound traffic blocked except SSH (22), DNS (53) and HTTP (80).

---

## Docker

### Why Docker?

With limited RAM (4GB), keeping overhead low matters. Docker containers share the operating system kernel instead of running separate OS instances like virtual machines:

- Lower memory usage per service
- Fast startup and shutdown
- Reproducible configuration via `docker-compose.yml` — if the SD card fails, the entire stack can be restored in minutes
- Clear separation between services without the cost of full virtualisation

### Installation

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
newgrp docker
sudo apt install docker-compose-plugin -y

# Verify
docker run hello-world
docker compose version
```

### Directory Structure

```
~/docker/
└── pihole/
    ├── docker-compose.yml
    ├── etc-pihole/
    └── etc-dnsmasq.d/
```

---

## Pi-hole

### Why Pi-hole?

DNS (Domain Name System) is involved in almost every connection a device makes on the internet — it is the protocol that translates domain names into IP addresses. Because all DNS traffic passes through a central point, it is an effective place to filter unwanted traffic.

From a security perspective, DNS filtering at the network level provides something browser-based adblockers cannot:

- Protects devices that lack their own adblockers — smart TVs, IoT devices, mobile apps
- Blocks telemetry and "phone home" traffic sent without the user's knowledge
- Provides visibility into what traffic is actually being generated on the network — which is itself valuable security information
- Can block known malicious domains (malware, phishing) before a device even establishes a connection

### docker-compose.yml

```yaml
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    network_mode: host
    environment:
      TZ: 'Europe/Stockholm'
      WEBPASSWORD: 'set-separately'
    volumes:
      - ./etc-pihole:/etc/pihole
      - ./etc-dnsmasq.d:/etc/dnsmasq.d
    restart: unless-stopped
```

`network_mode: host` means Pi-hole uses the Pi's network interface directly, simplifying DNS listening on port 53 without port mapping.

`restart: unless-stopped` means Pi-hole starts automatically when the Pi reboots — without this, DNS would stop working on the network after every restart.

### Start Pi-hole

```bash
cd ~/docker/pihole
docker compose up -d
docker ps
```

### Verify DNS is Working

```bash
sudo apt install dnsutils -y
dig google.com @127.0.0.1
# Expected response: status: NOERROR
```

### Blocklist

Pi-hole ships with a default list. Added on day 1:

```
https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts
```

Added via: Adlists → Add → Tools → Update Gravity

### Per-Device DNS Configuration

The D-Link R15 does not support custom DNS in its DHCP settings, meaning DNS cannot be automatically distributed to all devices via the router. The temporary solution is manual configuration per device. A more robust solution — having Pi-hole take over as the DHCP server — is planned. See [Next Steps](#next-steps).

---

## Results After Day 1

| Metric | Value |
|---|---|
| Total DNS queries | 3,071 |
| Blocked queries | 941 |
| Percentage blocked | 30.6% |
| Domains on blocklist | 81,935 |
| Active clients | 3 |
| Pi-hole memory usage | 2.6% |

---

## Next Steps

The following has been identified but not yet implemented. This is an active project and the list is updated continuously.

- [ ] Install and configure fail2ban for automatic blocking of repeated login attempts
- [ ] Enable automatic security updates (unattended-upgrades)
- [ ] Further SSH hardening (restrict login to specific users, consider changing default port)
- [ ] Set up Portainer for visual container management
- [ ] Evaluate CIS Benchmark Level 1 for Debian as a hardening reference
- [ ] Set up log monitoring

---

## Related

- [LinkedIn post](#) — shorter introduction for a broader audience
- [Raspberry Pi OS](https://www.raspberrypi.com/software/)
- [Pi-hole documentation](https://docs.pi-hole.net/)
- [Docker installation guide](https://docs.docker.com/engine/install/raspberry-pi-os/)
- [CIS Debian Benchmark](https://www.cisecurity.org/benchmark/debian_linux)
- [StevenBlack hosts blocklist](https://github.com/StevenBlack/hosts)
