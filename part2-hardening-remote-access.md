# Homelab — Part 2: Hardening, Remote Access & Automatic Maintenance

> This post continues from Part 1. The focus here is on hardening the existing setup, establishing secure remote access, and automating routine maintenance tasks.

## SSH Hardening

Several improvements were made to the SSH configuration beyond the initial key-based authentication established in Part 1.

A cloud-init override file was discovered to be silently re-enabling password authentication despite `PasswordAuthentication no` being set in the main config:

```bash
# /etc/ssh/sshd_config.d/50-cloud-init.conf was overriding the main config
# Fixed by setting:
PasswordAuthentication no
```

Additional hardening applied in `/etc/ssh/sshd_config`:

```
PermitRootLogin no
AllowUsers hector
MaxAuthTries 3
ClientAliveInterval 120
ClientAliveCountMax 30
```

- `PermitRootLogin no` — prevents direct root login over SSH entirely
- `AllowUsers hector` — only this user can authenticate via SSH
- `MaxAuthTries 3` — limits authentication attempts per connection, complementing fail2ban
- `ClientAliveInterval` / `ClientAliveCountMax` — automatically disconnects idle sessions after 60 minutes

A separate ed25519 key pair was generated for Termius (iPhone SSH client) and added to `authorized_keys`, giving authenticated mobile access without reusing the Windows key.

## fail2ban

Installed and configured to monitor SSH login attempts:

```bash
sudo apt install fail2ban -y
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

Configuration added to `/etc/fail2ban/jail.local`:

```ini
[sshd]
enabled = true
port = ssh
logpath = %(sshd_log)s
backend = %(sshd_backend)s
maxretry = 3
bantime = 1h
findtime = 10m
```

3 failed attempts within 10 minutes results in a 1 hour ban. With key-only authentication already enforced, fail2ban adds a second layer by banning IPs that repeatedly probe the SSH port, reducing log noise and attack surface.

## Remote Access — Tailscale

The D-Link R15 Quick VPN (L2TP/IPSec) was attempted but failed — the ISP blocks UDP ports 500 and 4500 required for the protocol. After exhausting standard troubleshooting the approach was abandoned in favour of Tailscale.

Tailscale creates an encrypted mesh network between authorised devices without requiring any open ports on the router.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Installed as a system service directly on the OS — not in Docker. Running it at the OS level means Docker outages do not affect remote access.

UFW updated to allow Tailscale's IP range to reach Portainer:

```bash
sudo ufw allow from 100.64.0.0/10 to any port 9000
```

Services are now accessible remotely via the Pi's Tailscale IP:

| Service | Remote Address |
| --- | --- |
| SSH | `ssh hector@100.x.x.x` |
| Portainer | `http://100.x.x.x:9000` |
| Pi-hole admin | `http://100.x.x.x/admin` |

## Portainer CE

Deployed as a Docker container for visual management of containers, images, volumes and stacks.

```yaml
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    ports:
      - "9000:9000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

volumes:
  portainer_data:
```

UFW restricted to local network and Tailscale range only — port 9000 is not exposed publicly:

```bash
sudo ufw allow from 192.168.0.0/24 to any port 9000
sudo ufw allow from 100.64.0.0/10 to any port 9000
```

## Watchtower

Automatic Docker image updates deployed as a container. Runs daily at 04:00 Stockholm time, removes old images after updating, and excludes Pi-hole to avoid unplanned DNS outages.

```yaml
services:
  watchtower:
    image: containrrr/watchtower
    container_name: watchtower
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - WATCHTOWER_SCHEDULE=0 0 4 * * *
      - WATCHTOWER_CLEANUP=true
      - TZ=Europe/Stockholm
      - DOCKER_API_VERSION=1.40
    labels:
      - "com.centurylinklabs.watchtower.enable=false"
```

`DOCKER_API_VERSION=1.40` was required to resolve an API version mismatch between Watchtower and the installed Docker daemon.

Pi-hole excluded by adding to its compose file:

```yaml
labels:
  - "com.centurylinklabs.watchtower.enable=false"
```

## Automatic System Updates

`unattended-upgrades` installed and configured for automatic OS package updates:

```bash
sudo apt install unattended-upgrades apt-listchanges -y
sudo dpkg-reconfigure -plow unattended-upgrades
```

Key settings in `/etc/apt/apt.conf.d/50unattended-upgrades`:

```
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Automatic-Reboot "false";
Unattended-Upgrade::Automatic-Reboot-Time "03:00";
```

Auto-reboot is intentionally disabled — a headless server should not reboot unattended.

## Docker Log Limits

Without explicit limits Docker log files grow indefinitely and can fill the SD card. Created `/etc/docker/daemon.json`:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

Each container is limited to 3 files of 10MB — 30MB maximum per container. All existing containers were recreated to apply the new config.

## Log Rotation

Verified that `logrotate` is correctly configured and running. All key log files are covered including fail2ban, ufw, unattended-upgrades and apt. Rotation is monthly with 12 months retention.

## Hardware Note

The Raspberry Pi 4 was purchased second hand and listed by the seller as a 4GB model. Confirmed via firmware query that it is in fact an 8GB model:

```bash
vcgencmd get_config total_mem
# Returns: total_mem=8192
```

## Next Steps

- Backup strategy for Docker volumes and configs
- Tailscale DNS → Pi-hole (filter DNS queries remotely via Tailscale)
- Nginx Proxy Manager + Pi-hole local DNS (friendly hostnames for services)
- Homepage dashboard (central management hub)
- Monitoring stack (Prometheus, Grafana, Loki, Promtail, cAdvisor)
- Snort IDS/IPS
- Cowrie Honeypot
- Trivy (container vulnerability scanning)
- CrowdSec
- CIS Benchmark Level 1 for Debian

## Related

- [Part 1](link-to-part-1)
- [Tailscale documentation](https://tailscale.com/kb)
- [Portainer documentation](https://docs.portainer.io)
- [fail2ban documentation](https://www.fail2ban.org/wiki/index.php/Main_Page)
- [containrrr/watchtower](https://containrrr.dev/watchtower)
