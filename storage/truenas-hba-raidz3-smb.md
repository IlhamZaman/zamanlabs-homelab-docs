<!--
Organized from: STEP 5 truenas_lsi_raidz3_smb_setup(2).txt
Source wording and technical details were preserved as closely as possible.
-->

> [!CAUTION]
> This is a historical implementation record. Verify versions, addresses, paths, and commands against the live environment before applying changes.

# TrueNAS Setup Documentation

### Environment

Host:
- Proxmox VE 9.1
- TrueNAS running as a VM inside Proxmox
- LSI/Broadcom SAS2008 HBA, commonly sold as LSI 9211-8i
- 8 x 1TB-class Seagate ST91000640NS drives
- Goal: TrueNAS should directly control the physical drives through the passed-through LSI HBA

Important storage design:
- Pool type: RAIDZ3
- Number of vdevs: 1
- Drives in vdev: 8
- Parity drives: 3
- Usable data drives: 5
- TrueNAS reported usable capacity: about 4.01 TiB


### 1. Initial Issue: Duplicate Serial Numbers

TrueNAS originally showed a warning like:

    Disks have duplicate serial numbers: None (sda, sdb, sdc, sdd, sde, sdf, sdg, sdh, sdi)

The first check inside the TrueNAS shell was:

```text
    lsblk -o NAME,SIZE,MODEL,SERIAL,WWN
```

The disks showed as QEMU virtual disks:

    sdb    931.5G QEMU HARDDISK drive-scsi1
    sdc    931.5G QEMU HARDDISK drive-scsi2
    sdd    931.5G QEMU HARDDISK drive-scsi3
    sde    931.5G QEMU HARDDISK drive-scsi4
    sdf    931.5G QEMU HARDDISK drive-scsi5
    sdg    931.5G QEMU HARDDISK drive-scsi6
    sdh    931.5G QEMU HARDDISK drive-scsi7
    sdi    931.5G QEMU HARDDISK drive-scsi8

Conclusion:
- TrueNAS was not seeing the real disks.
- Proxmox was passing the drives as virtual QEMU disks.
- This hides the real serial numbers, WWNs, SMART data, and drive model information.
- This is not the preferred setup for TrueNAS/ZFS.

Correct design:

```text
    Proxmox host
    └── TrueNAS VM
        └── PCI passthrough of the whole LSI 9211-8i / SAS2008 HBA
            └── TrueNAS sees the real physical drives directly
```


### 2. Proxmox TrueNAS VM Hardware Cleanup

Before passing through the LSI card, the individual QEMU disks were removed from the TrueNAS VM.

In Proxmox:

    TrueNAS VM → Hardware

Removed the old virtual data disks:

    scsi1
    scsi2
    scsi3
    scsi4
    scsi5
    scsi6
    scsi7
    scsi8

Important:
- Do not remove the TrueNAS boot disk.
- The boot disk is usually scsi0 or virtio0.
- Only remove the 8 virtual data disks that were representing the physical drives.

After cleanup, the TrueNAS VM should keep only things like:
- TrueNAS boot disk
- Network device
- Display
- Memory
- CPU
- Later, the passed-through LSI PCI device


### 3. Proxmox Settings for PCI Passthrough

The TrueNAS VM was already using:

    Machine: q35

This is good for PCI passthrough.

Recommended TrueNAS VM settings:
- Machine: q35
- CPU type: host
- SCSI Controller: VirtIO SCSI single
- Network Device: VirtIO paravirtualized
- Memory: at least 10GB, preferably 16GB+ if available
- Boot disk: SCSI on VirtIO SCSI single
- Boot disk advanced options if on SSD:
    - SSD emulation: enabled
    - Discard: enabled

If IOMMU is not already enabled on Proxmox, enable it.

Edit GRUB:

```text
    nano /etc/default/grub
```

Find:

```text
    GRUB_CMDLINE_LINUX_DEFAULT="quiet"
```

For Intel systems, change to:

```text
    GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"
```

Then update GRUB:

```text
    update-grub
```

Add VFIO modules:

```text
    nano /etc/modules
```

Add:

    vfio
    vfio_iommu_type1
    vfio_pci
    vfio_virqfd

Reboot Proxmox:

```text
    reboot
```

After reboot, verify IOMMU:

    dmesg | grep -e DMAR -e IOMMU


### 4. Identifying the LSI HBA in Proxmox

In Proxmox shell, the LSI card can be identified with:

```text
    lspci | grep -i sas
```

The card appeared as:

    0000:82:00.0 Broadcom / LSI SAS2008 PCI-Express Fusion-MPT SAS-2

This is the LSI 9211-8i / SAS2008 HBA.


### 5. Passing the LSI Card Through to TrueNAS

In the Proxmox web UI:

    TrueNAS VM → Hardware → Add → PCI Device

Use:
- Raw Device, not Mapped Device
- Select:

    0000:82:00.0 Broadcom / LSI SAS2008 PCI-Express Fusion-MPT SAS-2

Recommended options:
- All Functions: checked
- PCI-Express: checked
- Primary GPU: unchecked
- ROM-Bar: enabled is okay, but if the VM fails to start, try disabling it

Then click:

    Add

After adding the PCI device, start the TrueNAS VM.


### 6. Verifying TrueNAS Sees the Real Drives

Inside TrueNAS:

    System → Shell

Run:

```text
    lsblk -o NAME,SIZE,MODEL,SERIAL,WWN
```

After successful passthrough, the drives showed as real Seagate disks:

    sdb    931.5G ST91000640NS  9XG6FANX    0x5000c50066ab592d
    sdc    931.5G ST91000640NS  9XG50XC8    0x5000c5006436b95f
    sdd    931.5G ST91000640NS  9XG5X6PE    0x5000c50065ea9d81
    sde    931.5G ST91000640NS  9XG53E23    0x5000c5003cf8a8b2
    sdf    931.5G ST91000640NS  9XG4GD3Q    0x5000c50063dc6826
    sdg    931.5G ST91000640NS  9XG59V02    0x5000c500647fb243
    sdh    931.5G ST91000640NS  9XG5X4C4    0x5000c50065eb7b85
    sdi    931.5G ST91000640NS  9XG5X62V    0x5000c50065ead947

This confirmed:
- LSI passthrough worked.
- TrueNAS now sees real disk models.
- TrueNAS now sees unique serial numbers.
- TrueNAS now sees unique WWNs.
- The duplicate serial number warning was fixed.


### 7. RAIDZ3 Pool Setup

Goal:
- 8 total drives
- 3 parity drives
- 5 usable data drives

In TrueNAS:

    Storage → Create Pool

Selected the 8 physical data drives:

    sdb
    sdc
    sdd
    sde
    sdf
    sdg
    sdh
    sdi

Pool layout:

    1 vdev
    RAIDZ3
    8 disks in the vdev

Capacity math:
- Simple raw math:

    8 total drives - 3 parity drives = 5 usable drives

- Each drive is about 931.5 GiB.
- 5 x 931.5 GiB = about 4.55 TiB raw usable before ZFS overhead.

TrueNAS reported usable capacity:

    4.01 TiB

Why it is 4.01 TiB instead of 4.55 TiB:
- ZFS metadata overhead
- RAIDZ padding overhead
- Sector alignment / ashift overhead
- Reserved/slop space
- Real TiB accounting
- RAIDZ3 with 8 drives has 5 data columns + 3 parity columns, which is safe but not perfectly space-efficient

Conclusion:
- 4.01 TiB usable is normal for this layout.
- The pool is not broken.


### 8. Dataset vs Zvol Decision

For normal file sharing, use datasets.

Definitions:
- Pool: the full RAIDZ3 storage tank
- Dataset: a folder-like ZFS filesystem inside the pool
- Zvol: a block device, mostly used for iSCSI or VM disks

For SMB file sharing:
- Create datasets.
- Do not create zvols.

Zvols are only needed for:
- iSCSI targets
- VM disks
- Block-level storage use cases

For this setup:
- Use dataset: yes
- Use zvol: no


### 9. SMB Dataset Setup

In TrueNAS:

    Storage → Datasets

Selected the pool and created a dataset.

Dataset path shown later:

```text
    /mnt/seagateNAS/samba
```

Dataset/share name used:

    samba

Recommended dataset settings:
- Name: samba
- Share Type: SMB

This dataset is the location that the SMB share points to.


### 10. SMB User Setup

A main user was created.

Important distinction:
- For an SMB-only user, do not give admin, shell, or SSH access.
- For the main TrueNAS admin user, Full Admin access is okay.

The user's actual TrueNAS username was:

    ilham

Not:

    ilham_zaman

This mattered later because using the wrong username caused permission denied errors.

For the main admin user, selected options:
- SMB Access: checked
- TrueNAS Access: Full Admin
- Shell Access: optional/checked
- SSH Access: optional, but recommended off unless needed
- Group: ilham
- Auxiliary group: builtin_administrators is okay for main admin
- Shell: /usr/bin/bash is okay for admin
- Sudo Commands: Not Set is fine


### 11. Dataset ACL / Permissions Setup

In TrueNAS:

    Storage → Datasets → select /mnt/seagateNAS/samba → Edit ACL

ACL owner/group shown:

    Owner: ilham
    Owner Group: ilham

ACL entries shown:
- User - ilham: Allow | Full Control
- Group - ilham: Allow | Modify
- Group - builtin_users: Allow | Modify
- Group - builtin_administrators: Allow | Full Control

Important checkboxes:
- Validate effective ACL: checked
- Apply permissions recursively: optional, okay if the dataset is new/empty
- Apply Owner: use if changing owner
- Apply Group: use if changing group

Then click:

    Save Access Control List

This allows the user ilham to access the SMB dataset.


### 12. SMB Share Setup

In TrueNAS:

    Shares → Windows SMB Shares → Add

The SMB share name was:

    samba

Important:
- The share name is what the client mounts.
- Use the exact share name from smbclient output.
- In this case, the correct share name was lowercase:

    samba

Not:

    Samba

The SMB share should point to:

```text
    /mnt/seagateNAS/samba
```

After creating the share:
- Start the SMB service when prompted.
- Enable SMB service to start automatically.

Service path:

```text
    System Settings → Services → SMB → Start
    System Settings → Services → SMB → Start Automatically
```


### 13. Checking SMB Shares from Linux Laptop

From the Linux laptop, install needed tools if necessary.

On Arch Linux:

```text
    sudo pacman -S cifs-utils smbclient
```

List SMB shares:

    smbclient -L //192.168.1.253 -U ilham

The output showed:

    Sharename       Type      Comment
    ---------       ----      -------
    samba           Disk
    IPC$            IPC       IPC Service (TrueNAS Server)
    SMB1 disabled -- no workgroup available

Notes:
- The warning "SMB1 disabled -- no workgroup available" is normal.
- SMB1 is old and should remain disabled.
- The useful part is that the share name is listed as:

    samba

There was also a warning:

    Can't load /etc/samba/smb.conf - run testparm to debug it

This came from the laptop-side smbclient configuration. Since the shares still listed, it was not the main issue.


### 14. Mounting the SMB Share on Linux Laptop

Create a local mount point:

```text
    sudo mkdir -p /mnt/truenas
```

Mount command:

    sudo mount -t cifs //192.168.1.253/samba /mnt/truenas \
      -o username=ilham,vers=3.1.1,sec=ntlmssp,uid=$(id -u),gid=$(id -g),iocharset=utf8,noperm

Important:
- Use the TrueNAS IP:

## 192.168.1.253

- Use the exact share name:

    samba

- Use the exact username:

    ilham

- Do not use the full name.
- Do not use ilham_zaman unless that is the actual TrueNAS username.

If the mount asks for a password, enter the TrueNAS password for user:

    ilham


### 15. Troubleshooting Mount Errors

Error:

```text
    mount error(2): No such file or directory
```

Common causes:
- Local mount folder does not exist
- Wrong SMB share name

Fix:

```text
    sudo mkdir -p /mnt/truenas
```

Then verify the actual share name:

    smbclient -L //192.168.1.253 -U ilham

Error:

```text
    mount error(13): Permission denied
```

Common causes:
- Wrong username
- Wrong password
- SMB Access not enabled for the user
- Dataset ACL does not allow the user
- Wrong share name capitalization
- Share points to the wrong dataset

Fixes checked:
- Confirmed username is ilham, not ilham_zaman
- Confirmed share name is lowercase samba, not Samba
- Confirmed ACL gives User ilham Full Control
- Confirmed dataset path is /mnt/seagateNAS/samba
- Confirmed SMB share exists and is visible with smbclient

Useful test:

    smbclient //192.168.1.253/samba -U ilham

If this drops into:

    smb: \>

then SMB login and share permissions are working.

To leave smbclient:

    exit


### 16. Final Working Design

Final storage design:

```text
    Proxmox VE 9.1
    └── TrueNAS VM
        ├── Boot disk as virtual disk
        ├── VirtIO network adapter
        └── Passed-through LSI/Broadcom SAS2008 HBA
            ├── sdb ST910006640NS
            ├── sdc ST910006640NS
            ├── sdd ST910006640NS
            ├── sde ST910006640NS
            ├── sdf ST910006640NS
            ├── sdg ST910006640NS
            ├── sdh ST910006640NS
            └── sdi ST910006640NS
```

Final ZFS layout:

```text
    Pool: seagateNAS
    └── 1 x RAIDZ3 vdev
        └── 8 x 931.5G drives
```

Usable capacity shown by TrueNAS:

    About 4.01 TiB

Final SMB layout:

    Dataset:

```text
        /mnt/seagateNAS/samba
```

    SMB Share:
        samba

    Main user:
        ilham

    Permissions:
        User ilham: Full Control

Linux mount target:

```text
    /mnt/truenas
```

Linux mount command:

    sudo mount -t cifs //192.168.1.253/samba /mnt/truenas \
      -o username=ilham,vers=3.1.1,sec=ntlmssp,uid=$(id -u),gid=$(id -g),iocharset=utf8,noperm


### 17. Key Lessons

- Do not pass TrueNAS data drives as individual QEMU disks if using ZFS seriously.
- Pass the whole LSI HBA through to TrueNAS.
- TrueNAS should see real disk models, serials, and WWNs.
- For 8 drives with RAIDZ3, use 1 vdev.
- RAIDZ3 with 8 drives gives 3-drive fault tolerance.
- TrueNAS usable capacity can be lower than simple raw math because of ZFS overhead.
- For SMB shares, create a dataset, not a zvol.
- The SMB mount command must use the actual share name, not necessarily the dataset name.
- Linux CIFS can be picky about capitalization.
- The actual username matters. In this setup, it was ilham, not ilham_zaman.
