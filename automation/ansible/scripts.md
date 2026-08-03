# Ansible Helper Scripts

## `maintenance.sh`

Current order:

1. General health check
2. TrueNAS health check
3. Configuration backups
4. Proxmox update
5. Fedora update
6. openSUSE update
7. APT update for Ubuntu, Debian, Hermes, and monitoring
8. AlmaLinux management update
9. TrueNAS update with apply and reboot enabled
10. Service checks
11. Reboot report

Because TrueNAS application and reboot are enabled by the script, run it only when that behavior is intended.

## Reboot Wrappers

- `reboot-pve.sh`
- `reboot-truenas.sh`
- `reboot-management.sh`
- `reboot-vms.sh`

Each wrapper supplies the confirmation variable expected by its playbook. The live repository does not currently contain a `reboot-hermes.sh` wrapper.
