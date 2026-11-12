# My Raspberry Pi Homelab

I'm turning an old Raspberry Pi 3 into a home server to learn Linux, networking, and self-hosting. I'm building it one stage at a time and writing down everything, including the stuff that broke.

## Hardware

| Part | Details |
|---|---|
| Board | Raspberry Pi 3 Model B (Rev 1.2), 1 GB RAM |
| Storage | 32 GB microSD |
| Network | Wi-Fi (2.4 GHz) on my home network |
| OS | Raspberry Pi OS Lite 64-bit (Debian 13 "trixie"), no desktop |
| Managed from | MacBook Air over SSH |

## Stages

| Stage | What it's about | Status |
|---|---|---|
| 1. Hardened server | Headless Linux, key-only SSH, firewall, automatic security updates, Wi-Fi that survives reboots | ✅ Done ([notes](stage-1-hardening.md)) |
| 2. Containers | Docker + Docker Compose, Uptime Kuma, Homepage dashboard, Pi-hole | Up next |
| 3. Remote access | Tailscale VPN, reverse proxy for friendly names | Planned |
| 4. Sensors | ESP32 + temperature sensor → MQTT → Grafana graphs | Planned |
| 5. Backups and automation | restic backups, restore tests, alerts, maybe Ansible | Planned |
| 6. Expansion | Second machine with Proxmox, my own router for network experiments | Someday |

## What broke (so far)

My first build lost Wi-Fi after I ran a full system update. On a monitor, the Pi had no saved Wi-Fi network at all. It turned out the Wi-Fi settings from Raspberry Pi Imager were being regenerated at every boot into a temporary folder (`/run/`), and the update broke the setup chain that does that.

I wiped the card and rebuilt everything. This time I created my own permanent NetworkManager profile (saved in `/etc/`), gave it priority over the generated one, turned off Wi-Fi power saving, and tested that it survived a reboot and a full upgrade. Full details are in the [Stage 1 notes](stage-1-hardening.md#what-went-wrong-the-first-time).

## What I've learned so far

- How SSH keys work, and why turning off password logins matters
- The difference between settings that are temporary (`/run/`) and permanent (`/etc/`), and why a setting isn't really saved until it survives a reboot
- How to set up a firewall without locking myself out (allow SSH *before* turning it on)
- Why long jobs like system updates should run inside `tmux`
- Troubleshooting from the bottom up: power → network → ping → SSH

## Notes

- No passwords, keys, or Wi-Fi details are stored in this repo.
- IP addresses are written as placeholders like `<pi-ip>`.
