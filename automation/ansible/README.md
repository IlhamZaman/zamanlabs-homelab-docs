# Current Ansible Homelab Automation

**Source of truth:** [`IlhamZaman/ansible-homelab-automation`](https://github.com/IlhamZaman/ansible-homelab-automation)  
**Reviewed:** August 3, 2026  
**Supersedes:** Uploaded `STEP 11 Ansible Maintenance Automation(3).txt`

The live repository manages the Proxmox hypervisor, TrueNAS SCALE, the AlmaLinux management VM, Linux VMs, and the monitoring LXC. Its current responsibilities include configuration backups, health and service checks, package updates, reboot reporting, and explicitly confirmed reboots.

## Control and Access Model

- **Control node:** AlmaLinux 9 Management VM
- **Transport:** SSH
- **Public SSH exposure:** None
- **Authentication:** SSH keys
- **Host verification:** Strict host-key checking with a dedicated Ansible `known_hosts` file
- **Privilege escalation:** `sudo`, enabled globally without prompting
- **Concurrency:** 15 forks
- **SSH multiplexing:** `ControlMaster=auto` with a 120-second persistent control connection
- **Pipelining:** Enabled

## Managed Inventory

| Group | Systems |
|---|---|
| `proxmox_hosts` | Proxmox VE hypervisor |
| `management_vm` | Local AlmaLinux management VM |
| `ubuntu_servers` | Cloudflare/Nginx gateway |
| `hermes_servers` | Hermes Agent VM |
| `debian_servers` | Jellyfin, two Minecraft servers, and Immich |
| `monitoring_servers` | Prometheus/Grafana monitoring LXC |
| `fedora_servers` | Uptime Kuma and Homepage VM |
| `opensuse_servers` | Technitium DNS VM |
| `truenas_servers` | TrueNAS SCALE VM |
| `linux_servers` | Ubuntu, Debian, Fedora, and openSUSE groups |
| `linux_maintenance_targets` | Linux systems, Hermes, monitoring, Proxmox, and management VM |
| `maintenance_targets` | Linux systems, Hermes, Proxmox, TrueNAS, and management VM |

The exact public inventory template is preserved as [`inventory.example.ini`](inventory.example.ini).

## Current Playbooks

| Playbook | Current responsibility |
|---|---|
| `backup-configs.yml` | Collects selected Linux, Proxmox, Homepage, gateway, Minecraft, and management configuration backups |
| `baseline-packages.yml` | Installs common packages across supported Linux distributions |
| `check-services.yml` | Checks important services across the homelab |
| `health-check.yml` | Reports general Linux and Proxmox health information |
| `install-qemu-agent.yml` | Installs and enables QEMU Guest Agent on supported VMs |
| `reboot-management.yml` | Reboots the management VM only after an explicit confirmation variable |
| `reboot-proxmox.yml` | Reboots Proxmox only after an explicit confirmation variable |
| `reboot-report.yml` | Reports uptime and reboot requirements after maintenance |
| `reboot-truenas.yml` | Checks middleware readiness and reboots TrueNAS only after explicit confirmation |
| `reboot-vms.yml` | Reboots Linux VMs and the monitoring LXC serially after explicit confirmation |
| `reboot-hermes.yml` | Reboots only the Hermes Agent VM after explicit confirmation |
| `truenas-health-check.yml` | Checks middleware readiness, version, and active alerts |
| `update-all.yml` | Imports the platform-specific update playbooks |
| `update-apt.yml` | Updates Ubuntu, Debian, Hermes, and monitoring targets |
| `update-fedora.yml` | Updates Fedora and reports whether a reboot may be needed |
| `update-management.yml` | Updates the AlmaLinux management VM |
| `update-opensuse.yml` | Refreshes and updates openSUSE, then checks affected processes |
| `update-proxmox.yml` | Verifies Proxmox and updates host packages |
| `update-truenas.yml` | Checks TrueNAS updates and optionally installs them and reboots |

## Current Helper Scripts

| Script | Behavior |
|---|---|
| `maintenance.sh` | Runs health checks, config backups, platform updates, TrueNAS updates, service checks, and reboot reporting |
| `reboot-pve.sh` | Runs the confirmed Proxmox reboot playbook |
| `reboot-truenas.sh` | Runs the confirmed TrueNAS reboot playbook |
| `reboot-management.sh` | Runs the confirmed management VM reboot playbook |
| `reboot-vms.sh` | Runs the confirmed Linux VM/LXC reboot playbook |

There is currently a `reboot-hermes.yml` playbook, but no separate `reboot-hermes.sh` helper script in the live repository.

## Important Current Maintenance Behavior

The live `maintenance.sh` does more than report available updates. It currently calls `update-truenas.yml` with:

```bash
-e truenas_apply_updates=true \\
-e truenas_reboot_after_update=true
```

That means the combined maintenance script is authorized to install an available TrueNAS update and request a reboot. Review that behavior before running the script during a normal maintenance window.

## Safety Controls

The current reboot playbooks use explicit Boolean confirmation variables and `serial: 1` where appropriate. Example:

```bash
ansible-playbook playbooks/reboot-proxmox.yml \
  -e confirm_proxmox_reboot=true
```

The TrueNAS update playbook defaults to `truenas_apply_updates: false`, but the current maintenance script overrides it to `true`.

## Running the Project

```bash
cd ~/homelab/ansible
export ANSIBLE_CONFIG="$(pwd)/ansible.cfg"
ansible-inventory --graph
ansible all -m ping
```

Run individual playbooks deliberately rather than treating the combined maintenance script as a harmless status command.
