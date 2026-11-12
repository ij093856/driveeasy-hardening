# Hardening Steps: DriveEasy Rentals Ubuntu Host

## 1. Environment
- VirtualBox on Windows, Ubuntu 26.04 LTS (brief specified 22.04)
- SSH reached from Windows via NAT port forward: host 127.0.0.1:2222 -> guest 22
- Commands run in the VM console as `ikram`, unless marked (Windows PowerShell)

## 2. Capture pre-hardening evidence
| Command | Why |
|---|---|
| `sudo cp /var/log/auth.log ~/before/auth.log` | Baseline auth log |
| `sudo ufw status > ~/before/ufw_status.txt` | Baseline firewall (inactive) |
| `sudo sshd -T \| sort > ~/before/sshd_before.txt` | Baseline SSH settings |
| `sudo ss -tulpn > ~/before/listening_before.txt` | Baseline open ports |

## 3. Create a non-root admin user
- `sudo adduser adminops`: dedicated admin account
- `sudo usermod -aG sudo adminops`: grants sudo
- Verify: `getent group sudo`

## 4. Use ssh.service instead of ssh.socket
- `sudo systemctl disable --now ssh.socket`
- `sudo systemctl enable --now ssh.service`
- Why: Ubuntu 26.04 starts SSH via a socket; the service is simpler to configure.
- Verify: `systemctl is-active ssh.socket ssh.service` shows `inactive` then `active`

## 5. Install SSH key for adminops
- Generate on Windows (PowerShell): `ssh-keygen -t ed25519`
- Copy the public key into `/home/adminops/.ssh/authorized_keys`
- Verify: key login works with no password prompt

## 6. Harden SSH (`/etc/ssh/sshd_config`)
| Setting | Why |
|---|---|
| `PermitRootLogin no` | Blocks the most-attacked account |
| `PasswordAuthentication no` | Stops password brute force |
| `PubkeyAuthentication yes` | Allows key login |
| `MaxAuthTries 3` | Limits guesses per connection |
| `X11Forwarding no` | Removes an unneeded feature |
| `AllowUsers adminops` | Only this user may log in |

- Validate: `sudo sshd -t`
- Apply: `sudo systemctl restart ssh`
- Verify: `sudo sshd -T | grep -Ei 'permitrootlogin|passwordauthentication|pubkeyauthentication|allowusers|maxauthtries|x11forwarding'`

## 7. Firewall (UFW)
- `sudo ufw default deny incoming`
- `sudo ufw default allow outgoing`
- `sudo ufw allow OpenSSH`
- `sudo ufw enable`
- Verify: `sudo ufw status verbose`

## 8. Remove unused services
- `sudo systemctl disable --now avahi-daemon.service avahi-daemon.socket`
- `sudo systemctl disable --now cups.service cups.socket cups.path cups-browsed.service`
- `sudo systemctl mask avahi-daemon.service cups.service`
- Why: network discovery and printing are not needed on a server; fewer open ports.
- Verify: `sudo ss -tulpn` shows only SSH (22) and the local DNS resolver (53).

## 9. Automatic updates
- `sudo apt install unattended-upgrades`
- `sudo dpkg-reconfigure -plow unattended-upgrades`
- Verify: `cat /etc/apt/apt.conf.d/20auto-upgrades` (both values "1")

## 10. Proof of key-only access (Windows PowerShell)
- Key login succeeds: `ssh -p 2222 adminops@127.0.0.1`
- Password forced: `ssh -p 2222 -o PubkeyAuthentication=no adminops@127.0.0.1` -> `Permission denied (publickey)`
- Other user: `ssh -p 2222 ikram@127.0.0.1` -> `Permission denied (publickey)`
- Screenshot: `screenshots/ssh_key_only.png`
- Log evidence: `after/auth.log` shows `Accepted publickey for adminops` (4 times) and `User ikram from 10.0.2.2 not allowed because not listed in AllowUsers`

## 11. Before/after comparison
| Item | Before | After |
|---|---|---|
| UFW | inactive | active, deny incoming, only SSH allowed |
| Root SSH | `prohibit-password` | `no` |
| Password auth | `yes` | `no` |
| MaxAuthTries | 6 | 3 |
| X11 forwarding | yes | no |
| AllowUsers | not set (any user) | `adminops` only |
| Listening ports | 22, 631 (cups, localhost only), 5353 (avahi), DNS | 22, DNS |

Full proof: `before/sshd_before.txt`, `after/sshd_after.txt` and `after/sshd_diff.txt`.
