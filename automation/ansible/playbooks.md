# Ansible Playbook Reference

This list reflects the live repository reviewed on August 3, 2026.

## Health and Baseline

- `baseline-packages.yml`
- `health-check.yml`
- `truenas-health-check.yml`
- `check-services.yml`
- `install-qemu-agent.yml`

## Backups

- `backup-configs.yml`

The backup playbook collects selected configuration files into a local `backups/<inventory-hostname>/` structure. Backups can contain sensitive operational data and should stay outside the public documentation repository.

## Updates

- `update-all.yml`
- `update-apt.yml`
- `update-fedora.yml`
- `update-opensuse.yml`
- `update-management.yml`
- `update-proxmox.yml`
- `update-truenas.yml`

`update-all.yml` imports the platform-specific update playbooks. TrueNAS has its own application and reboot variables and should be reviewed separately before use.

## Reboots

- `reboot-report.yml`
- `reboot-proxmox.yml`
- `reboot-truenas.yml`
- `reboot-vms.yml`
- `reboot-management.yml`
- `reboot-hermes.yml`

The state-changing reboot playbooks require explicit confirmation variables. Do not remove those checks merely to shorten commands.
