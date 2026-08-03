<!--
Organized from: STEP 8 Proxmox Backups on TrueNAS NFS(2).txt
Source wording and technical details were preserved as closely as possible.
-->

> [!CAUTION]
> This is a historical implementation record. Verify versions, addresses, paths, and commands against the live environment before applying changes.

# STEP 8 - Proxmox Backups on TrueNAS NFS

Goal:
Set up Proxmox VE backups so they are stored on the TrueNAS RAIDZ pool using an NFS share.

Important Concept:
Since the LSI/HBA and RAID drives are passed through to the TrueNAS VM, Proxmox does not directly see those disks anymore. That is normal.

The correct flow is:

Proxmox VM backups -> TrueNAS NFS share -> TrueNAS RAIDZ drives

Recommended Protocol:
Use NFS instead of SMB for Proxmox backup storage.

NFS is cleaner for Linux-to-Linux storage and works well with Proxmox backup jobs.

---
## 1. Create a Backup Dataset in TrueNAS

In TrueNAS:

1. Go to Datasets.
2. Select the TrueNAS pool, for example:

   SeagateNAS

3. Add a new dataset.
4. Name it something like:

   Proxmox-Backups

Example full path:

```text
   /mnt/SeagateNAS/Proxmox-Backups
```

This dataset will store Proxmox VZDump backup files.

---
### 2. Enable the NFS Service in TrueNAS

In TrueNAS:

1. Go to System Settings > Services.
2. Find NFS.
3. Start the NFS service.
4. Enable Start Automatically.

Do not put Proxmox port 8006 anywhere in the NFS settings.

Port 8006 is only for the Proxmox web UI:

   https://PROXMOX-IP:8006

NFS uses its own service ports automatically.

The main NFS port is usually 2049, but you normally do not manually enter this in the Proxmox or TrueNAS GUI.

---
### 3. Create the NFS Share in TrueNAS

In TrueNAS:

1. Go to Shares.
2. Go to Unix Shares (NFS).
3. Add a new NFS share.
4. Select the dataset path:

```text
   /mnt/SeagateNAS/Proxmox-Backups
```

## 5. Add a description, for example:

   PVEX Backups

## 6. Make sure Enabled is checked.

---
### 4. Allowed Host / Network Setup

TrueNAS has two different ways to allow access:

Option A - Hosts:
Use this if you want to allow one specific Proxmox host.

   Hosts: 192.168.1.242

Do not add a port.
Do not use https://.
Do not use :8006.

Correct:

## 192.168.1.242

Wrong:

```text
   192.168.1.242:8006
   https://192.168.1.242:8006
```

Option B - Networks:
Use this if entering the Proxmox IP under Networks.

For one specific Proxmox host:

   Network: 192.168.1.242
   CIDR: /32

This shows as:

## 192.168.1.242/32

That means only this one Proxmox host can access the NFS share.

For the whole LAN, you could use:

## 192.168.1.0/24

But for better security, only allowing the Proxmox IP is preferred.

Recommended for this setup:

## 192.168.1.242/32

---
### 5. NFS Permission Fix: Maproot

If Proxmox can mount the share but fails to create the backup folder, you may see an error like:

   create storage failed: mkdir /mnt/pve/TrueNAS-PVE-BAK/dump: Permission denied

This means Proxmox mounted the NFS share, but TrueNAS is not allowing Proxmox to create the required dump folder.

Proxmox needs to create:

```text
   /mnt/pve/TrueNAS-PVE-BAK/dump
```

To fix this:

1. In TrueNAS, go to Shares > Unix Shares (NFS).
2. Edit the Proxmox-Backups NFS share.
3. Click Advanced Options.
4. Set:

   Maproot User: root
   Maproot Group: wheel

5. Save the NFS share.
6. Restart the NFS service.

Go to:

   System Settings > Services > NFS > Restart

This fixes the common NFS root squash / permission issue for Proxmox backups.

---
### 6. Dataset Permissions in TrueNAS

Also check the dataset permissions.

In TrueNAS:

1. Go to Datasets.
2. Select:

   SeagateNAS > Proxmox-Backups

3. Edit Permissions.
4. For a simple homelab backup-only dataset, use:

   Owner User: root
   Owner Group: wheel

5. Make sure owner/group have write permission.
6. Apply recursively if needed.

This dataset is dedicated to Proxmox backups, so using root/wheel here is fine for this homelab use case.

---
### 7. Do Not Change Global NFS Ports

The global NFS service settings may show fields like:

   mountd bind port
   rpc.statd bind port
   rpc.lockd bind port

Leave these blank.

Those are not needed for this setup.

The permission error is not a missing port problem. It is a share/dataset permissions and Maproot issue.

---
### 8. Add the NFS Storage in Proxmox

In Proxmox:

1. Go to Datacenter.
2. Go to Storage.
3. Click Add.
4. Choose NFS.

Fill it out like this:

```text
   ID: TrueNAS-PVE-BAK
   Server: TrueNAS-IP
   Export: /mnt/SeagateNAS/Proxmox-Backups
   Content: VZDump backup file
   Nodes: All or your Proxmox node
   Enable: checked
```

Important:

Server must be the TrueNAS IP, not the Proxmox IP.

Example:

   Server: 192.168.1.xxx

Use the IP address of the TrueNAS VM.

Do not put the Proxmox web UI port 8006.

Content should be:

   VZDump backup file

Not:

   Disk image

This storage is for backups, not VM disks.

---
### 9. If Export Dropdown Does Not Populate

If the Export dropdown does not show the TrueNAS NFS export:

Check these things:

1. TrueNAS NFS service is running.
2. The NFS share is enabled.
3. The Proxmox IP is allowed in the NFS share.
4. TrueNAS and Proxmox can reach each other on the network.
5. The TrueNAS IP entered in Proxmox is correct.
6. No firewall rule is blocking NFS traffic.

---
### 10. Test from Proxmox Shell

After adding the NFS storage, test it from the Proxmox shell.

Run:

```text
   pvesm status
```

You should see the TrueNAS NFS storage listed as active.

Then test folder creation:

```text
   mkdir -p /mnt/pve/TrueNAS-PVE-BAK/dump
   touch /mnt/pve/TrueNAS-PVE-BAK/dump/test.txt
   ls -l /mnt/pve/TrueNAS-PVE-BAK/dump
```

If the touch command works, Proxmox can write to the NFS share.

---
### 11. Create a Proxmox Backup Job

In Proxmox:

1. Go to Datacenter.
2. Go to Backup.
3. Click Add.

Recommended settings:

```text
   Storage: TrueNAS-PVE-BAK
   Mode: Snapshot
   Compression: ZSTD
   Schedule: Daily or Weekly
   Selection mode: Selected VMs or All VMs
```

Good VMs to back up:

   Technitium DNS VM
   Cloudflare Tunnel VM
   Jellyfin VM
   Minecraft Server VM
   Alma/Terraform management VM

---
### 12. Be Careful Backing Up the TrueNAS VM

Do not rely on backing up the TrueNAS VM to storage that is provided by the TrueNAS VM itself.

That creates a circular dependency:

   TrueNAS VM provides the backup storage
   Proxmox tries to back up TrueNAS into that same storage

Better TrueNAS backup plan:

1. Export and save the TrueNAS configuration file.
2. Keep notes/screenshots of pool, dataset, users, and share settings.
3. Know how to import the pool again if TrueNAS is rebuilt.
4. Optionally keep a separate backup of the TrueNAS boot disk/config somewhere else.

For TrueNAS, the config backup is more important than trying to back up the whole VM to itself.

---
### 13. Manual Backup Test

To manually test a VM backup from Proxmox shell:

   vzdump 100 --storage TrueNAS-PVE-BAK --mode snapshot --compress zstd

Replace 100 with the VM ID you want to test.

Example:

   vzdump 101 --storage TrueNAS-PVE-BAK --mode snapshot --compress zstd

If this completes successfully, the Proxmox-to-TrueNAS backup setup is working.

---
### Final Notes

Use NFS for Proxmox backups.
Do not use port 8006 for NFS.
Use the TrueNAS IP as the NFS server in Proxmox.
Use the Proxmox IP as the allowed host/network in TrueNAS.
Use VZDump backup file as the Proxmox storage content type.
If there is a permission denied error, set Maproot User to root and Maproot Group to wheel on the NFS share.
Leave global NFS service port fields blank unless there is a special firewall reason to set them manually.

This setup lets Proxmox store VM backups safely on the TrueNAS RAIDZ drives.
