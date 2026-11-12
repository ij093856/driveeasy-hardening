# DriveEasy Rentals: Ubuntu Host Hardening (Task 1)

## Environment
- Hypervisor: VirtualBox on Windows
- OS: Ubuntu 26.04 LTS (brief specified 22.04; 26.04 was already installed, and the steps used are identical on both versions)

## Hardening checklist
- [x] Non-root admin user (`adminops`) with sudo
- [x] Root SSH login disabled
- [x] Key-only SSH authentication (passwords disabled)
- [x] SSH limited to `adminops` (`AllowUsers`)
- [x] UFW enabled, default deny, only SSH allowed
- [x] Unused services removed (avahi, cups)
- [x] Unattended-upgrades enabled
- [x] Before/after logs captured

## Evidence
- `before/`: auth.log, ufw_status.txt, SSH settings, services and ports before hardening
- `after/`: the same files after hardening, plus `sshd_diff.txt`
- `screenshots/ssh_key_only.png`: key login works, password login refused

## What I did and why
I hardened a Linux host that stores customer and vehicle data. I removed the most common attack paths: password logins, root login, and unused network services. I added a default-deny firewall and automatic security updates, then proved the result with command output and logs.

See `hardening_steps.md` for every command with explanations.
