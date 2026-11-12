# Stage 1: Hardened Linux server

**Goal:** a headless Raspberry Pi that I only reach over SSH with a key, that patches itself, has a firewall, and comes back on its own after a reboot or an update.

**Status:** done.

`<pi-ip>` below means the Pi's address on my home network. My router gives it the same address every time (a DHCP reservation).

---

## What went wrong the first time

My first build (November 2026) worked until I ran a full system update. After that, SSH just hung and the Pi didn't show up on the network.

**How I tracked it down:**

1. SSH hung with no error, so the problem was reaching the Pi, not my keys.
2. `ping` got 100% packet loss, and the router listed the Pi as inactive.
3. With a monitor and keyboard plugged in, the Pi had booted fine but said `My IP address is 127.0.1.1`, which means no network connection was up.
4. `nmcli device wifi list` showed the Pi could *see* my Wi-Fi, so it wasn't distance or the antenna.
5. `nmcli connection show` showed **no saved Wi-Fi profile at all**.

**Cause:** Raspberry Pi Imager's Wi-Fi settings are applied at boot through a chain: cloud-init reads them, netplan turns them into a NetworkManager profile, and that profile is stored in `/run/`, which is wiped at every shutdown. The update most likely broke that chain, so after a reboot nothing recreated the profile.

**Fix:** I wiped the card, rebuilt from scratch, and created my own Wi-Fi profile saved permanently in `/etc/` (step 7 below).

---

## 1. Fresh SSH key (on the Mac)

```bash
ssh-keygen -t ed25519 -C "george homelab"
```

The private key stays on the Mac and the public key goes on the Pi. I set a passphrase so the key is useless even if the file gets copied. I also cleared old fingerprints with `ssh-keygen -R <pi-ip>` so the rebuilt Pi wouldn't trigger a "host identification changed" warning.

## 2. Flash the card with Raspberry Pi Imager

I cleared Imager's saved settings first (it remembers your old key and Wi-Fi), then chose:

- Raspberry Pi 3 → **Raspberry Pi OS Lite (64-bit)**. Lite has no desktop, so it saves RAM on a 1 GB board.
- Hostname `pi-homelab-george`, user `george`, my time zone, Wi-Fi with country set to US
- SSH on, **public key only**, with the new `.pub` key pasted in
- Raspberry Pi Connect **off**. SSH already covers remote access, and fewer services means less to secure.

## 3. First boot and login

I put the card in, plugged in power last, and waited 5 minutes. Then I checked from the bottom up:

```bash
ping -c 4 <pi-ip>       # is it on the network?
ssh george@<pi-ip>      # then log in
```

## 4. Make the Wi-Fi permanent (the actual fix)

First, check where the Wi-Fi profile is stored:

```bash
nmcli -f NAME,TYPE,DEVICE,FILENAME connection show
```

On a fresh install, everything was in `/run/`, which is temporary. That's the same fragile setup as the first build.

Next, create my own profile. The password goes into a variable so it doesn't end up in my shell history:

```bash
read -rs -p "Wi-Fi password: " WIFIPW; echo
sudo nmcli connection add type wifi ifname wlan0 con-name "homelab-wifi" \
  ssid "<my-network>" wifi-sec.key-mgmt wpa-psk wifi-sec.psk "$WIFIPW" \
  connection.autoconnect yes connection.autoconnect-priority 10 \
  802-11-wireless.powersave 2
unset WIFIPW
```

- **Priority 10** beats the generated profile (priority 0), so mine gets picked at boot.
- **`powersave 2`** turns Wi-Fi power saving off. With it on, the radio naps between activity, which can delay or drop incoming SSH connections. That's fine for a phone on battery, but bad for a server.

Then switch over, check, and do a reboot test:

```bash
sudo nmcli connection up homelab-wifi
nmcli -f NAME,DEVICE connection show --active   # homelab-wifi on wlan0
iw wlan0 get power_save                         # off
sudo reboot
```

After the reboot it came back on `homelab-wifi` by itself.

## 5. Health check before the big update

| Check | Command | My result |
|---|---|---|
| Power | `vcgencmd get_throttled` | `0x0` (no under-voltage) |
| Temperature | `vcgencmd measure_temp` | 44°C |
| Memory | `free -h` | 762 MB available of 905 MB |
| Disk | `df -h /` | 29 GB, 10% used |

A weak power supply can corrupt the SD card during a big update, so I wanted `0x0` before going further.

## 6. Full update inside tmux

If SSH drops mid-update, everything running in that session gets killed and you can end up with half-installed packages. `tmux` keeps the session running on the Pi.

```bash
sudo apt install tmux -y
tmux new -s update
sudo apt update && sudo apt full-upgrade -y
sudo dpkg --audit      # no output = nothing half-installed
sudo reboot
```

After the reboot, the Wi-Fi was still on `homelab-wifi`. That's exactly the point where my first build broke.

## 7. Automatic security updates

```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades
sudo unattended-upgrade --dry-run --debug 2>&1 | tail -n 15
```

This installs Debian security fixes daily but never reboots on its own. Bigger updates I still do by hand about once a month, inside tmux.

## 8. Firewall (ufw)

The order matters. If you enable the firewall before allowing SSH, you lock yourself out.

```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw show added    # make sure "allow 22/tcp" is there
sudo ufw enable
sudo ufw status verbose
```

I tested SSH from a new Terminal window while keeping the old one open, just in case.

## 9. Confirm password logins are off

```bash
sudo sshd -T | grep -Ei 'passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication'
```

Then I tried to log in with a password from the Mac on purpose:

```bash
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password george@<pi-ip>
```

Result: `Permission denied (publickey)`. The Pi only accepts my key.

## 10. Final reboot checklist

After one last reboot, everything should come back without me touching it:

| Check | Command | Expected |
|---|---|---|
| Wi-Fi profile | `nmcli -f NAME,DEVICE connection show --active` | `homelab-wifi` on `wlan0` |
| Power saving | `iw wlan0 get power_save` | off |
| Firewall | `sudo ufw status` | active, 22/tcp allowed |
| SSH | `systemctl is-active ssh` | active |
| Passwords off | `sudo sshd -T \| grep -i '^passwordauthentication'` | no |
| Auto-updates | `systemctl list-timers 'apt-daily*'` | two timers scheduled |
| Power | `vcgencmd get_throttled` | `0x0` |

---

## Lessons

- **Check bottom-up:** power → network → ping → SSH.
- **A setting isn't saved until it survives a reboot.** Know what lives in `/run/` (temporary) and what lives in `/etc/` (permanent).
- **Verify, don't assume.** I tested the password lockout from the outside instead of trusting the config file.
- **Run long jobs in tmux.**
- **Always `sudo shutdown now` before unplugging.** Pulling the power on a running Pi is the easiest way to corrupt the SD card.
