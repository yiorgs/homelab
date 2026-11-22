# Ansible: Stage 1 as code

I built Stage 1 by hand first ([notes](../stage-1-hardening.md)). Then I wrote the whole setup as an Ansible playbook, so a freshly flashed Pi turns into the same hardened server with one command.

## What the playbook does

| Part | What it sets up |
|---|---|
| Packages and updates | Refreshes the package list, installs `tmux` and `unattended-upgrades`, turns on daily automatic security updates |
| Firewall | Installs `ufw`, allows SSH **first**, blocks all other incoming traffic, then turns the firewall on |
| SSH | Key-only login and no root login, in its own config file. The file is tested with `sshd -t` before it's installed, and an assert checks the settings are really in effect afterwards |
| Wi-Fi | A permanent NetworkManager profile that wins over the temporary one from Raspberry Pi Imager, with power saving off. The network name and password come from an encrypted Ansible Vault file |

## Files

| File | What it is |
|---|---|
| `stage1.yml` | The playbook |
| `ansible.cfg` | Settings: which inventory to use, and timing for each task |
| `inventory.example.ini` | Copy to `inventory.ini` and put in your Pi's address and username |
| `vault.example.yml` | Copy to `group_vars/homelab/vault.yml`, fill in your Wi-Fi details, then encrypt it |

My real `inventory.ini` and `vault.yml` are not in this repo (they're in `.gitignore`).

## How I run it

I run Ansible from my Mac (`brew install ansible`, which includes the `community.general` and `ansible.posix` collections this uses). The Pi just needs Raspberry Pi OS Lite with SSH key login set up in Imager.

```bash
cp inventory.example.ini inventory.ini              # then edit it
mkdir -p group_vars/homelab
cp vault.example.yml group_vars/homelab/vault.yml   # then edit it
ansible-vault encrypt group_vars/homelab/vault.yml

ansible homelab -m ping --ask-vault-pass                         # can Ansible reach the Pi?
ansible-playbook stage1.yml --check --diff -K --ask-vault-pass   # dry run
ansible-playbook stage1.yml -K --ask-vault-pass                  # real run
```

`-K` asks for the Pi's sudo password and `--ask-vault-pass` asks for the vault password. Running the playbook a second time should show `changed=0`, which means everything is already set up.

On a brand-new Pi, skip the dry run: it fails at the firewall tasks because `ufw` isn't really installed during a dry run.

## Rebuild test

To prove it works on a blank Pi, I flashed a second microSD card and ran the playbook on it.

| Check | Result |
|---|---|
| Playbook on the fresh Pi | `ok=14 changed=8 failed=0` in **2 min 41 s** |
| Reboot | Came back on Wi-Fi by itself, using the permanent profile in `/etc/` |
| Second run | `changed=0`, nothing left to fix |

Most of the 2 min 41 s was downloading and installing packages.

## What broke during the test

**The Pi never showed up on the network.** On a monitor, `ip link` showed no Wi-Fi interface at all, so it wasn't a wrong password. `dmesg` showed the Wi-Fi driver loading its firmware into the chip and failing a check (`Downloaded RAM image is corrupted`). After one reboot the same firmware loaded fine, so the file and the chip were OK. `vcgencmd get_throttled` showed `0x50000`, which means the Pi has had low voltage, so my power supply is the main suspect.

**SSH warned that the host key had changed.** That's expected after a fresh install, but it's also what an attack would look like. Before trusting the new key, I checked its fingerprint on the Pi's own screen (`ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub`), then removed the old one with `ssh-keygen -R`.

## What I learned

- **Describe the end state, not the steps.** Each task says how things should be (`state: present`), so running the playbook twice is safe, and a second run with `changed=0` proves it.
- **Order matters with a firewall.** Allow SSH before turning `ufw` on, or you lock yourself out.
- **Test before applying, check after.** `validate` stops a broken SSH config from being installed, and an `assert` proves the settings actually work.
- **Keep secrets out of the code.** The Wi-Fi details live in an encrypted vault, `no_log` keeps them out of the output, and only example files are public.
- **No Wi-Fi interface means look lower.** Check `ip link` and `dmesg` before blaming the Wi-Fi settings.
