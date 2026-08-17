# NikLabs — Homelab Monitoring & Update Automation (n8n)

An end-to-end n8n workflow that monitors a self-hosted Proxmox homelab and safely automates container updates with human-in-the-loop approval via Telegram.

Built and iteratively debugged against a real 5-node Proxmox homelab (LXC containers running Transmission, SABnzbd, Sonarr, Jellyfin, qBittorrent), with the container inventory externalized to a version-controlled GitHub config file rather than hardcoded into the workflow.

## What it does

**1. Resource monitoring (every 15 min)**
Polls the Proxmox API for host-level CPU, memory, and disk usage. Sends a Telegram alert only when a configurable threshold is breached (85% CPU/mem, 80% disk) — silent otherwise.

**2. SMART disk health checks (daily, 9am)**
SSH into the Proxmox host, runs `smartctl -a -j` across every attached drive, and parses the JSON output for reallocated/pending sectors, SMART pass/fail status, and drive temperature (alerting above 55°C).

**3. Weekly update scan + approval (Sunday, 9am)**
- Fetches the current list of monitored containers from a GitHub-hosted JSON file (see [Container Config](#container-config) below)
- SSH into each container and checks for pending `apt` package updates
- Builds a Telegram digest with inline "Approve" buttons per container
- On approval: applies the update, waits 60s, runs a health check (`systemctl is-active`), and reports success — or flags for manual rollback via the most recent nightly Proxmox Backup Server snapshot if the health check fails

## Architecture

```
Schedule Triggers (15min / daily / weekly)
        │
        ├── Proxmox API (resource stats) ──► Threshold check ──► Telegram alert
        │
        ├── SSH (SMART data) ──► Parse ──► Threshold check ──► Telegram alert
        │
        └── GitHub API (container list) ──► SSH (apt check) ──► Build digest
                                                                       │
                                                          Telegram inline buttons
                                                                       │
                                                    Telegram Trigger (callback_query)
                                                                       │
                                              Apply update ──► Health check ──► Report
```

## Container config

Rather than hardcoding the list of monitored containers inside the workflow, this project stores them as data in a separate JSON file (see `containers.json` in a companion config repo), fetched at runtime via the GitHub Contents API and decoded/parsed in an n8n Code node. Adding or removing a monitored container means editing that JSON file and committing — no changes to the workflow itself required.

```json
{
  "targets": [
    { "key": "lxc_transmission", "targetType": "lxc", "targetId": "104", "targetName": "transmission", "serviceName": "transmission-daemon" }
  ]
}
```

## Setup

This workflow is provided as a reference/portfolio artifact, not a plug-and-play install — it's tailored to a specific homelab topology. To adapt it:

1. Import `sanitized_workflow.json` into your own n8n instance
2. Create the following n8n credentials and re-link them to the relevant nodes:
   - **Header Auth** — Proxmox API token
   - **SSH (password or key)** — your Proxmox host
   - **Telegram API** — your bot's token (from [@BotFather](https://t.me/BotFather))
   - **Header Auth** — a GitHub fine-grained PAT (Contents: Read-only) if using the GitHub config pattern
3. Replace the placeholder values in the workflow:
   - `YOUR_TELEGRAM_BOT_TOKEN`
   - `YOUR_TELEGRAM_CHAT_ID`
   - `YOUR_PROXMOX_HOST_IP`
4. Set the workflow to **Active** — the Telegram approval buttons require an active webhook to function

## Notes on design decisions

- **Snapshots vs. existing backups for rollback**: `pct snapshot` isn't available on containers with a `dir`-type storage mount (e.g., a shared downloads folder on non-ZFS storage), since Proxmox requires every attached volume to support snapshots. Rather than restructure working storage, this workflow relies on the existing nightly Proxmox Backup Server job as the rollback point — a deliberate tradeoff of up to ~24h of potential state loss on rollback, acceptable for non-critical services.
- **No persistent skip-memory**: declined updates simply reappear on the next weekly scan rather than being tracked/snoozed — kept intentionally simple for a small container count.

## Stack

n8n · Proxmox VE · Proxmox Backup Server · Telegram Bot API · GitHub REST API

---

*Part of the [NikLabs homelab](https://niklabs.org) project.*
