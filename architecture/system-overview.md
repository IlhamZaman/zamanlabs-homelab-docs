# System Overview

## Platform

The homelab is centered on Proxmox VE. Infrastructure services run in dedicated VMs or an LXC rather than directly on the hypervisor.

## Management Plane

The AlmaLinux management VM is the control point for SSH, Ansible, Terraform, Git, and maintenance scripts. The current Ansible project also manages Proxmox and TrueNAS with platform-specific playbooks.

## Storage Plane

TrueNAS owns the physical data drives through HBA passthrough and provides SMB and NFS storage. Proxmox stores backups on a TrueNAS NFS dataset, while applications such as Jellyfin and Immich use network-backed media storage.

## Access Plane

The Ubuntu gateway centralizes Nginx and Cloudflare Tunnel functions. Some applications use direct HTTPS through Nginx, while dashboards and administrative services may use Cloudflare Tunnel depending on their requirements.

## Monitoring Plane

Prometheus, Grafana, and exporters run in the monitoring LXC. Uptime Kuma and Homepage share a Fedora VM and provide availability checks and a visual service portal.

## Application Plane

Dedicated VMs isolate Jellyfin, Immich, Minecraft, Technitium DNS, and Hermes Agent. Hermes uses a self-hosted Honcho stack on the same VM for local persistent memory.
