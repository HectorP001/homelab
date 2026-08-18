# Homelab — Bash Scripting: Nightly OS Upgrade Automation

**Date:** August 18, 2026
**Environment:** Raspberry Pi 4 (8GB) · Raspberry Pi OS Lite 64-bit

---

## Overview

A small deviation from the monitoring stack series — a bash scripting exercise rather than a new service. `unattended-upgrades` was already running, but only covers the `-security` origin by default, so general package upgrades were sitting unapplied. Used this as an opportunity to write a proper script from scratch instead of just widening the existing config.

**What was built:**
- A script that runs `apt update && apt upgrade -y` nightly, with logging and error handling
- A cron schedule, ordered against the existing backup job
- A `logrotate` config for the log file
- A passive flag for pending reboots

---

## The Script

**`/usr/local/bin/nightly-upgrade.sh`:**

```bash
#!/bin/bash
set -euo pipefail

LOGFILE="/var/log/nightly-upgrade.log"

echo "=== Run started: $(date) ===" >> "$LOGFILE"
apt-get update >> "$LOGFILE" 2>&1
apt-get upgrade -y >> "$LOGFILE" 2>&1
echo "=== Run finished: $(date) ===" >> "$LOGFILE"

if [ -f /var/run/reboot-required ]; then
    echo "*** REBOOT REQUIRED ***" >> "$LOGFILE"
fi
```

```bash
sudo chmod +x /usr/local/bin/nightly-upgrade.sh
```

---

## Design Decisions

**`set -euo pipefail`** — exits on any failed command, treats unset variables as errors, and surfaces failures inside pipes. Standard practice for unattended scripts.

**`apt-get` over `apt`** — `apt` is the human-facing CLI and its output format isn't guaranteed stable across versions. `apt-get` is the script-safe interface.

**`>>` and `2>&1`** — `>>` appends so the log keeps a running history instead of being overwritten each run. `2>&1` redirects stderr into the same destination as stdout, so errors land in the log instead of vanishing when nothing's watching the terminal at 2 AM.

**No `sudo` inside the script** — it's root-owned and only ever invoked as root (via cron or `sudo ./script.sh`), so there's nothing left to elevate.

**Cron: 02:00, an hour before the existing 03:00 backup job** — avoids both jobs competing for resources, and means the backup captures the system in its already-updated state.

```
0 2 * * * /usr/local/bin/nightly-upgrade.sh
0 3 * * * /usr/local/bin/homelab-backup.sh
```

**No automatic reboot** — consistent with `unattended-upgrades` already having auto-reboot disabled here. This Pi runs Pi-hole (network-wide DNS) and is only reachable remotely via Tailscale, so an unattended reboot going wrong is a real risk. The script just flags `/var/run/reboot-required` in the log instead.

**Log rotation added** — same pattern used for the Docker logs elsewhere on this Pi:

```
/var/log/nightly-upgrade.log {
    monthly
    rotate 12
    compress
    missingok
    notifempty
}
```

---

## Verification

```bash
sudo /usr/local/bin/nightly-upgrade.sh
echo $?                              # 0 = success
cat /var/log/nightly-upgrade.log
grep CRON /var/log/syslog | tail -20 # confirm cron fired
```

---

## What's Next

Standalone exercise — the monitoring stack series continues separately with Grafana Alerting. Possible follow-up: route the reboot-required flag through Pushover once that's wired in for alerting.

---

*Part of an ongoing homelab build documented in [HectorP001/homelab](https://github.com/HectorP001/homelab).*
