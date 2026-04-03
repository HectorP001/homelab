# Homelab — Part 3: Backups, DNS Routing, Proxy, Dashboard & Subnet Access

This post continues from Part 2. The focus here is on building out the service layer: automated backups, routing DNS through Pi-hole over Tailscale, setting up Nginx Proxy Manager with friendly hostnames, deploying a Homepage dashboard, and configuring Tailscale subnet routing for full remote network access.

---

## Automated Backup

A backup script was written to run nightly via cron, capturing all critical configuration files to a USB drive mounted at `/mnt/backup`.

### USB Drive Setup

The drive is FAT32 formatted and auto-mounts on boot via `/etc/fstab`:

```
/dev/sda1  /mnt/backup  vfat  defaults,nofail  0  0
```

The `nofail` flag ensures the Pi boots normally even if the drive is not present.

### Backup Script

`/usr/local/bin/homelab-backup.sh`:

```bash
#!/bin/bash

DATE=$(date +%Y-%m-%d)
DEST="/mnt/backup/homelab/$DATE"

mkdir -p "$DEST"

cp -r ~/docker "$DEST/docker"
cp -r /etc/ssh "$DEST/ssh"
cp -r /etc/ufw "$DEST/ufw"
cp /etc/fail2ban/jail.local "$DEST/jail.local"
cp /etc/docker/daemon.json "$DEST/daemon.json"
cp /etc/apt/apt.conf.d/50unattended-upgrades "$DEST/50unattended-upgrades"

# Remove backups older than 30 days
find /mnt/backup/homelab/ -maxdepth 1 -type d -mtime +30 -exec rm -rf {} +
```

What is backed up:

- `~/docker/` — all compose files and service configs
- `/etc/ssh/` — SSH configuration
- `/etc/ufw/` — firewall rules
- `/etc/fail2ban/jail.local` — fail2ban config
- `/etc/docker/daemon.json` — Docker log limits
- `/etc/apt/apt.conf.d/50unattended-upgrades` — automatic update config

Retention is 30 days. The script is owned by root and executable:

```bash
sudo chmod +x /usr/local/bin/homelab-backup.sh
```

### Cron Schedule

Added to root's crontab (`sudo crontab -e`):

```
0 3 * * * /usr/local/bin/homelab-backup.sh
```

Runs daily at 03:00. To verify the latest backup:

```bash
ls /mnt/backup/homelab/$(ls /mnt/backup/homelab/ | tail -1)
```

---

## Tailscale DNS → Pi-hole

With Tailscale running on the Pi, iPhone, and Windows laptop, the next step was routing all DNS queries from Tailscale-connected devices through Pi-hole — so ad blocking and local DNS resolution work remotely, not just on the home network.

### Pi-hole: Allow All Origins

By default Pi-hole only accepts queries from the local network. Tailscale queries arrive from the `100.x.x.x` address space, so Pi-hole was configured to permit all origins:

Settings → DNS → Interface settings → **Permit all origins**

This is safe because Pi-hole is not exposed to the public internet — access is restricted to the local network and Tailscale range via UFW.

### UFW Rules

Port 53 opened for both local network and Tailscale range:

```bash
sudo ufw allow from 192.168.0.0/24 to any port 53
sudo ufw allow from 100.64.0.0/10 to any port 53
```

### Tailscale Admin Console

Configured at `login.tailscale.com/admin/dns`:

- **Custom nameserver:** Pi's Tailscale IP
- **Override local DNS:** enabled

With override local DNS enabled, all Tailscale-connected devices route their DNS through Pi-hole regardless of what DNS is configured locally on each device. This means ad blocking, telemetry filtering and local hostname resolution all work the same whether on the home network or connecting remotely.

### Pi: Prevent Tailscale from Overwriting DNS

On the Pi itself, Tailscale would overwrite `/etc/resolv.conf` with its own DNS settings when override local DNS is enabled. This was prevented with:

```bash
sudo tailscale set --accept-dns=false
```

The Pi's `/etc/resolv.conf` correctly points to the router, which forwards to Pi-hole. Allowing Tailscale to overwrite this would create a circular dependency where the Pi tries to resolve DNS through itself.

---

## Nginx Proxy Manager

Pi-hole was originally running on port 80. To support Nginx Proxy Manager (NPM) — which needs port 80 for HTTP and certificate handling — Pi-hole was migrated to port 8080 via its compose environment variable:

```yaml
environment:
  - FTLCONF_webserver_port=8080
```

### NPM Compose

`~/docker/npm/docker-compose.yml`:

```yaml
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      - '80:80'
      - '81:81'
      - '443:443'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```

NPM admin UI is accessible at port 81.

### UFW Rules

```bash
sudo ufw allow from 192.168.0.0/24 to any port 80
sudo ufw allow from 100.64.0.0/10 to any port 80
sudo ufw allow from 192.168.0.0/24 to any port 81
sudo ufw allow from 100.64.0.0/10 to any port 81
```

### Docker Bridge Traffic

An important subtlety: when NPM forwards a request to another service (e.g. Pi-hole), the traffic originates from Docker's internal bridge network — not from `192.168.0.x`. Without an explicit rule for Docker's bridge range, UFW blocks this internal forwarding silently.

Fix — added to UFW:

```bash
sudo ufw allow from 172.16.0.0/12
```

This covers all Docker bridge networks (`172.16.0.0/12` to `172.31.255.255`) in a single rule. Without it, friendly hostnames resolve correctly via DNS but the actual request gets dropped by UFW at the forwarding stage — services appear to load but return errors.

---

## Friendly Hostnames

With NPM running, services are accessible via readable names instead of IP:port combinations. This required two things working together: local DNS records in Pi-hole to resolve the names, and proxy hosts in NPM to forward the requests to the correct service.

### Pi-hole — Local DNS Records

Added under Pi-hole admin → Local DNS → DNS Records:

| Domain | Target |
|---|---|
| `pihole.home` | Pi IP |
| `portainer.home` | Pi IP |
| `npm.home` | Pi IP |
| `hub.home` | Pi IP |

All `.home` domains resolve to the Pi's local IP. Any device using Pi-hole as its DNS server gets these answers automatically — no manual host file entries needed on individual machines.

### NPM — Proxy Hosts

Configured in the NPM admin UI:

| Domain | Forwards To |
|---|---|
| `pihole.home` | `Pi IP:8080` |
| `portainer.home` | `Pi IP:9000` |
| `npm.home` | `Pi IP:81` |
| `hub.home` | `Pi IP:3000` |

NPM receives the incoming request on port 80, matches the hostname, and forwards it to the correct service and port internally. The result is that typing `http://portainer.home` in a browser on the local network — or over Tailscale — reaches Portainer without needing to remember port numbers.

---

## Homepage Dashboard

With several services now running, having a single place to see everything at a glance became useful. Homepage was chosen over alternatives like Homarr and Heimdall for two reasons: live service integration tiles that pull real data from Pi-hole and Portainer, and YAML-based configuration that fits naturally alongside the rest of the compose setup.

`~/docker/homepage/docker-compose.yml`:

```yaml
services:
  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    container_name: homepage
    restart: unless-stopped
    ports:
      - '3000:3000'
    volumes:
      - ./config:/app/config
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - HOMEPAGE_ALLOWED_HOSTS=hub.home,192.168.0.x
```

### Docker Socket Integration

Mounting `/var/run/docker.sock` gives Homepage direct read access to the Docker daemon. This allows it to display live container state — running, stopped, restarting — for every container on the host without any additional configuration per service. Each tile on the dashboard reflects the actual current state rather than a static link.

### Service Integrations

Beyond container state, Homepage has native integrations for Pi-hole and other services via their APIs. The Pi-hole tile displays live stats directly on the dashboard — queries today, percentage blocked, and current status — so there is no need to open the Pi-hole admin panel just to check whether it is healthy.

### Access

Homepage is accessible at `http://hub.home` on the local network and at the Pi's Tailscale IP on port 3000 when remote. `HOMEPAGE_ALLOWED_HOSTS` restricts which hostnames can load the dashboard, preventing it from serving to unexpected origins.

The end result is a single URL that gives a live overview of the entire homelab — container health, DNS stats, and quick links to every admin interface — accessible from anywhere via Tailscale.

---

## Tailscale Subnet Routing

By default Tailscale only provides access to the Pi itself. Subnet routing extends this — the Pi advertises the entire home network (`192.168.0.0/24`) to Tailscale, so any connected device can reach any IP on the network as if it were local. This includes the router admin panel and any other devices on the network.

### Enable IP Forwarding

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### Advertise the Subnet

```bash
sudo tailscale up --advertise-routes=192.168.0.0/24
```

### Approve in Tailscale Admin Console

The advertised route must be explicitly approved at `login.tailscale.com/admin/machines`. Once approved, all Tailscale-connected devices can route traffic through the Pi to any address on the home network.

This behaves like a client-to-site VPN — connecting remotely gives full network access without exposing any ports on the router. Remote access summary:

| Service | Remote Address |
|---|---|
| SSH | `ssh hector@100.x.x.x` |
| Pi-hole admin | `http://100.x.x.x:8080/admin` |
| Portainer | `http://100.x.x.x:9000` |
| Homepage | `http://100.x.x.x:3000` |
| Router admin | `http://192.168.0.1` (via subnet routing) |

**One caveat:** if a remote network happens to also use `192.168.0.x` addressing, routing ambiguity can occur since both the local and remote network share the same subnet.

---

## UFW — Full Ruleset at This Point

For reference, the complete UFW ruleset after all services were configured:

```
22/tcp        ALLOW    Anywhere          # SSH
53            ALLOW    192.168.0.0/24    # DNS (Pi-hole, local)
53            ALLOW    100.64.0.0/10     # DNS (Pi-hole, Tailscale)
80            ALLOW    192.168.0.0/24    # HTTP / NPM
80            ALLOW    100.64.0.0/10
81            ALLOW    192.168.0.0/24    # NPM admin
81            ALLOW    100.64.0.0/10
8080          ALLOW    192.168.0.0/24    # Pi-hole admin
8080          ALLOW    100.64.0.0/10
3000          ALLOW    192.168.0.0/24    # Homepage
3000          ALLOW    100.64.0.0/10
9000          ALLOW    192.168.0.0/24    # Portainer
9000          ALLOW    100.64.0.0/10
lo            ALLOW    Anywhere          # Loopback
172.16.0.0/12 ALLOW    Anywhere          # Docker bridge (NPM internal forwarding)
```

All ports are restricted to the local network and Tailscale range. Nothing is exposed publicly.

---

## Next Steps

- Monitoring stack — Prometheus, Node Exporter, cAdvisor, Grafana, Loki, Promtail
- Snort IDS/IPS
- Cowrie Honeypot
- Trivy (container image vulnerability scanning)
- CrowdSec
- CIS Benchmark Level 1 for Debian
- Migrate compose secrets to `.env` files

---

## Related

- [Tailscale subnet routing documentation](https://tailscale.com/kb/1019/subnets)
- [Nginx Proxy Manager documentation](https://nginxproxymanager.com/guide/)
- [Pi-hole documentation](https://docs.pi-hole.net/)
- [gethomepage/homepage](https://gethomepage.dev/)
