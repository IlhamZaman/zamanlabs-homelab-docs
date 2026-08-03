<!--
Organized from: STEP 1.0 proxmox_ve_9_1_hardening_checklist(2).txt
Source wording and technical details were preserved as closely as possible.
-->

> [!CAUTION]
> This is a historical implementation record. Verify versions, addresses, paths, and commands against the live environment before applying changes.

# Proxmox VE 9.1 Hardening Checklist

## 1. Network Exposure

Do this first.

- Do not expose Proxmox web UI :8006 to the internet.
- Access Proxmox only from:
  - your LAN,
  - a dedicated management VLAN,
  - Tailscale/WireGuard VPN,
  - or one trusted admin IP.
- Put Proxmox, IPMI/BMC, TrueNAS, and switches on a management network if possible.
- Keep Minecraft/game servers on their own VM/VLAN, not on the Proxmox management network.

Suggested layout:

LAN / Home devices        192.168.1.0/24
Proxmox management        192.168.1.x only
VM services               separate bridge or VLAN later
IPMI/BMC                  separate static IP, LAN only
Remote access             Tailscale, not port forwarding


## 2. Update Repositories Correctly

If you do not have a subscription, use the official no-subscription repo, not random third-party repo scripts.

Check repos:

```text
cat /etc/apt/sources.list
ls /etc/apt/sources.list.d/
```

Then update:

```text
apt update
apt full-upgrade
reboot
```

Do this before you start hardening SSH/firewall.


## 3. Create a Non-Root Admin User

Do not daily-drive root@pam.

In Proxmox GUI:

Datacenter > Permissions > Users > Add

Example:

```text
username: gang
realm: pve
```

Then give admin rights:

Datacenter > Permissions > Add > User Permission
Path: /
User: gang@pve
Role: Administrator

Keep root@pam as emergency access, but use your normal admin account for daily work.


## 4. Enable 2FA

Turn on 2FA for every admin account.

GUI path:

Datacenter > Permissions > Two Factor

Then enforce it on the realm:

Datacenter > Permissions > Realms

Important:
Before forcing 2FA, keep one SSH/root session open so you do not lock yourself out.


## 5. Harden SSH

Edit:

```text
nano /etc/ssh/sshd_config
```

Use something like:

```text
PermitRootLogin prohibit-password
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
X11Forwarding no
AllowUsers youradminuser
```

Then test config:

```text
sshd -t
systemctl reload ssh
```

Do not close your current SSH session until you confirm a second SSH login works.

Test:

```text
ssh youradminuser@proxmox-ip
```

Only after that works, close the old session.


## 6. Lock Down Proxmox Firewall

Basic node firewall idea:

Allow only your trusted LAN/VPN IPs to:

8006/tcp   Proxmox web UI
22/tcp     SSH
5900-5999  console/VNC/SPICE if needed

Block everything else inbound.

GUI path:

Node > Firewall > Rules

Suggested allow rules:

ACCEPT  tcp  from 192.168.1.0/24  to node  port 8006
ACCEPT  tcp  from 192.168.1.0/24  to node  port 22
DROP    all

If you use Tailscale, allow your Tailnet subnet too.

Do not enable DROP rules until you confirm your allow rules are correct. Otherwise, you can lock yourself out.


## 7. Secure IPMI/BMC Separately

Your Intel BMC/IPMI is a whole separate computer inside the server. Treat it like a security risk.

Do this:

- Change default IPMI password.
- Disable anonymous/unused users.
- Keep IPMI LAN-only.
- Never port forward IPMI.
- Put IPMI on management VLAN if possible.
- Disable remote media/virtual console features you do not use.
- Update BMC firmware only from trusted vendor sources.

For the Wiwynn/Intel server, IPMI should be reachable only from your home LAN or VPN.


## 8. Use Tailscale/WireGuard Instead of Port Forwarding

Good:

Laptop/phone -> Tailscale -> Proxmox web UI

Bad:

Internet -> Port 8006 -> Proxmox

Do not forward:

8006
22
IPMI ports
TrueNAS UI

For Minecraft, only forward the Minecraft VM’s port if needed, not the Proxmox host.


## 9. Separate VMs from the Host

Basic rule:

Proxmox host = management only
VMs = services

Do not run random services directly on the Proxmox host. No Plex, no Minecraft, no Docker stacks directly on the host unless you really know why.

Use VMs/LXCs:

Minecraft VM
TrueNAS VM
Linux utility VM
Docker VM

This keeps the hypervisor clean.


## 10. Backups Before Anything Fancy

Set up backups before you start tuning, clustering, GPU passthrough, or big storage changes.

GUI path:

Datacenter > Backup

Minimum:

- Back up important VMs.
- Store backups on a different disk/pool.
- Test restore at least once.
- Keep one offline/off-server copy for important stuff.

Proxmox Backup Server is ideal if you want proper backup infrastructure.


## 11. Use Least-Privilege API Tokens

If you later use Terraform/Ansible, do not use root passwords in scripts.

GUI path:

Datacenter > Permissions > API Tokens

Better:

terraform@pve!token

Give it only the permissions it needs.

Avoid:

root@pam
root password in plain text
full admin token for everything


## 12. Disable What You Do Not Use

Check listening services:

```text
ss -tulpn
```

You mainly expect:

22      SSH
8006    Proxmox GUI/API
3128    spice proxy
5900+   console ports when active

If you install extra services, keep track of them.

Also check enabled services:

```text
systemctl --type=service --state=running
```

Do not randomly disable Proxmox services unless you know what they do.


## 13. VM Security Defaults

For important VMs:

- Use UEFI/OVMF when appropriate.
- Add TPM for Windows 11/security-sensitive VMs.
- Avoid privileged LXCs unless needed.
- Avoid nesting unless needed.
- Do not pass through host devices casually.
- Keep VM disks on proper storage, not random USB drives.
- Snapshot before big changes.


## 14. LXC Container Rules

Prefer:

Unprivileged container
No nesting unless needed
No host mounts unless needed
No Docker inside LXC unless you understand the tradeoff

Avoid:

Privileged LXC + bind mounts + Docker + internet-facing app

That combo can get sketchy fast.


## 15. Storage Hardening

For ZFS/TrueNAS-style storage:

- Use SMART monitoring.
- Enable email or notification alerts.
- Scrub ZFS pools regularly.
- Do not ignore checksum errors.
- Label drive bays physically.
- Keep boot drive separate from VM/storage drives.
- Do not mix random failing drives into your main pool.

Useful commands:

zpool status
zpool scrub poolname
smartctl -a /dev/sdX


## 16. Logging and Alerts

Set up email alerts or another notification method.

Check logs:

```text
journalctl -p warning..alert -b
journalctl -u pvedaemon -u pveproxy -u pvestatd
```

Check failed SSH attempts:

```text
journalctl -u ssh | grep "Failed password"
```

Optional:

```text
apt install fail2ban
```

Use Fail2ban mostly for SSH. Do not rely on it as your main protection. Firewall/VPN first.


## 17. Browser Certificate

The default self-signed Proxmox cert is okay for a lab, but a real cert is cleaner.

Options:

- Use Proxmox ACME/Let’s Encrypt if you have a domain.
- Use internal CA.
- Or accept the self-signed warning on LAN only.

Do not expose the GUI just to get a certificate.


## 18. Physical Security

Since this is a 1U server:

- Set BIOS/UEFI password.
- Disable booting from random USB after install.
- Keep IPMI password strong.
- Keep server where people cannot casually plug stuff into it.
- Document drive bay layout.


## 19. Things Not to Do Yet

For this setup, do not overcomplicate it on day one.

Skip for now:

Complex VLAN setup before Proxmox is stable
Full disk encryption on Proxmox boot
Public reverse proxy to Proxmox UI
CrowdSec/Wazuh/SIEM
Custom kernel hardening
Random GitHub hardening scripts

Get stable first. Then tighten further.


## 20. Best Order

Do it in this order:

1. Fresh install Proxmox
2. Update repos and packages
3. Confirm networking works
4. Set static IP
5. Create non-root admin user
6. Enable 2FA
7. Harden SSH with keys
8. Enable Proxmox firewall carefully
9. Lock down IPMI
10. Set up backups
11. Build VMs
12. Add Tailscale/VPN
13. Add VLANs later


Biggest Wins

For a home lab, the biggest wins are:

- No internet-exposed Proxmox
- 2FA
- SSH keys
- Firewall allowlist
- IPMI locked down
- Real backups

That gets you most of the way without breaking the setup.
