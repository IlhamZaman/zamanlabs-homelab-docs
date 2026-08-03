<!--
Organized from: STEP 7 Jellyfin Debian 13.4 VM Setup(2).txt
Source wording and technical details were preserved as closely as possible.
-->

> [!CAUTION]
> This is a historical implementation record. Verify versions, addresses, paths, and commands against the live environment before applying changes.

# STEP 7 - Jellyfin Debian 13.4 VM Setup

### Purpose
This guide documents the Jellyfin VM setup conversation for a Proxmox home lab.
The goal is to install Jellyfin on Debian 13.4 in a Proxmox VM, give it enough CPU/RAM, mount media from TrueNAS, and create/manage Jellyfin users.

### Final VM Plan
Recommended Jellyfin VM settings:

CPU: 8 vCPU / host CPU cores
RAM: 8 GB
Disk: 64 GB or larger
OS: Debian 13.4 minimal
Network: VirtIO
Storage for media: SMB or NFS mount from TrueNAS
GPU passthrough: not required at first
Preferred playback: Direct Play when possible

Notes:
- 8 GB RAM and 8 vCPU is good for Jellyfin.
- This is more than enough for Direct Play, multiple light streams, metadata scanning, thumbnails, and some 1080p CPU transcoding.
- Heavy 4K transcoding is still better with GPU hardware acceleration.
- Jellyfin does not need the LSI card passed through.
- TrueNAS owns the disks through the LSI/HBA passthrough.
- Jellyfin should access media through SMB or NFS from TrueNAS.

### Minimum Requirements Summary
Direct Play only:

```text
CPU: 1-2 vCPU
RAM: 2 GB minimum, 4 GB better
Disk: 20-32 GB
GPU: Not needed
```

Better home lab setup:

```text
CPU: 2-4 vCPU
RAM: 4 GB
Disk: 32-64 GB
Media: mounted from TrueNAS
```

Chosen setup:

CPU: 8 vCPU
RAM: 8 GB
Disk: 64 GB
OS: Debian 13.4 minimal

### Step 1 - Create the Proxmox VM
In Proxmox, create a new VM.

Suggested settings:

OS: Debian 13.4 ISO
BIOS: SeaBIOS is fine
Machine: q35 or default i440fx is fine
SCSI Controller: VirtIO SCSI single
Disk: SCSI, 64 GB or larger
CPU: 8 vCPU
RAM: 8192 MB
Network: VirtIO

Do not pass the LSI card to this VM.
The LSI card should stay passed through to TrueNAS.

### Step 2 - Install Debian 13.4
Boot the Debian ISO.

During installation:

```text
Hostname: jellyfin
Domain: leave blank
Root password: your choice
User: create your normal admin user
Partitioning: guided, use entire disk
```

Software selection:

[x] SSH server
[x] standard system utilities
[ ] Debian desktop environment
[ ] GNOME
[ ] KDE

After install:

1. Remove the Debian ISO from the VM CD/DVD drive.
2. Reboot the VM.

### Step 3 - Log Into Debian
SSH into the VM:

```text
ssh youruser@VM-IP
```

Become root:

```text
su -
```

Or, if sudo is available:

```text
sudo -i
```

### Step 4 - Update Debian
Run:

```text
apt update
apt upgrade -y
```

Install useful tools:

```text
apt install -y curl gnupg ca-certificates sudo nano
```

Optional: give your user sudo access:

```text
usermod -aG sudo youruser
```

Then log out and back in.

### Step 5 - Set a Static IP
Check the network interface name:

```text
ip a
```

The interface may look like:

enp6s18

Edit the network config:

```text
nano /etc/network/interfaces
```

Example static config:

```text
auto lo
iface lo inet loopback
```

auto enp6s18
iface enp6s18 inet static
    address 192.168.1.50/24
    gateway 192.168.1.1
    dns-nameservers 192.168.1.1 1.1.1.1

Change these values for your network:

```text
192.168.1.50 = Jellyfin VM IP
192.168.1.1 = router IP
```

Restart networking:

```text
systemctl restart networking
```

Or reboot:

```text
reboot
```

Then SSH back in using the new IP:

```text
ssh youruser@192.168.1.50
```

### Step 6 - Add the Jellyfin Repository Key
Become root:

```text
sudo -i
```

Create the APT keyring directory:

```text
mkdir -p /etc/apt/keyrings
```

Download and save the Jellyfin signing key:

```text
curl -fsSL https://repo.jellyfin.org/jellyfin_team.gpg.key | gpg --dearmor -o /etc/apt/keyrings/jellyfin.gpg
```

### Step 7 - Add the Jellyfin Debian 13 Repository
Debian 13 is Trixie.

Create the Jellyfin source file:

cat > /etc/apt/sources.list.d/jellyfin.sources <<EOF2
Types: deb
URIs: https://repo.jellyfin.org/debian
Suites: trixie
Components: main
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/jellyfin.gpg
EOF2

Update package lists:

```text
apt update
```

### Step 8 - Install Jellyfin
Install Jellyfin:

```text
apt install -y jellyfin
```

This installs Jellyfin server, Jellyfin web UI, and the FFmpeg package Jellyfin uses.

### Step 9 - Enable Jellyfin on Boot
Check status:

```text
systemctl status jellyfin
```

Enable Jellyfin at boot:

```text
systemctl enable jellyfin
```

Start Jellyfin now:

```text
systemctl start jellyfin
```

Check status again:

```text
systemctl status jellyfin
```

You want to see:

active (running)

After this, Jellyfin starts automatically when Debian starts.

### Step 10 - Open Jellyfin in Browser
From your main PC, open:

http://JELLYFIN-IP:8096

Example:

http://192.168.1.50:8096

Jellyfin's default web port is 8096.

### Step 11 - First-Time Jellyfin Setup
In the Jellyfin web setup:

1. Choose language.
2. Create the admin username and password.
3. Skip media libraries for now if the TrueNAS share is not mounted yet.
4. Disable remote access unless you know you want it.
5. Finish setup.

For now, keep Jellyfin LAN-only.

### Step 12 - Mount TrueNAS Media Storage
Since the LSI card is passed through to TrueNAS, Jellyfin should access media through a TrueNAS SMB or NFS share.

Recommended options:

- NFS: clean for Linux-to-Linux setups.
- SMB: works fine and may already be configured in TrueNAS.

### Option A - Mount TrueNAS SMB Share
Install SMB tools:

```text
apt install -y cifs-utils
```

Create a mount folder:

```text
mkdir -p /mnt/media
```

Create a credentials file:

```text
nano /root/.smbcredentials
```

Put this inside:

```text
username=YOUR_TRUENAS_SMB_USER
password=YOUR_TRUENAS_SMB_PASSWORD
```

Lock down the credentials file:

```text
chmod 600 /root/.smbcredentials
```

Test the mount manually:

```text
mount -t cifs //TRUENAS-IP/SHARENAME /mnt/media -o credentials=/root/.smbcredentials,iocharset=utf8,uid=jellyfin,gid=jellyfin,file_mode=0664,dir_mode=0775
```

Example:

```text
mount -t cifs //192.168.1.30/Media /mnt/media -o credentials=/root/.smbcredentials,iocharset=utf8,uid=jellyfin,gid=jellyfin,file_mode=0664,dir_mode=0775
```

Check it:

```text
ls /mnt/media
```

If you see your movies/shows, the mount worked.

Make the SMB mount permanent:

```text
nano /etc/fstab
```

Add this line:

```text
//TRUENAS-IP/SHARENAME /mnt/media cifs credentials=/root/.smbcredentials,iocharset=utf8,uid=jellyfin,gid=jellyfin,file_mode=0664,dir_mode=0775,nofail,x-systemd.automount,_netdev 0 0
```

Example:

```text
//192.168.1.30/Media /mnt/media cifs credentials=/root/.smbcredentials,iocharset=utf8,uid=jellyfin,gid=jellyfin,file_mode=0664,dir_mode=0775,nofail,x-systemd.automount,_netdev 0 0
```

Test fstab:

```text
systemctl daemon-reload
mount -a
ls /mnt/media
```

### Option B - Mount TrueNAS NFS Share
Install NFS tools:

```text
apt install -y nfs-common
```

Create mount folder:

```text
mkdir -p /mnt/media
```

Test mount:

```text
mount -t nfs TRUENAS-IP:/mnt/POOL/DATASET /mnt/media
```

Example:

```text
mount -t nfs 192.168.1.30:/mnt/tank/media /mnt/media
```

Check it:

```text
ls /mnt/media
```

Make NFS mount permanent:

```text
nano /etc/fstab
```

Add this line:

TRUENAS-IP:/mnt/POOL/DATASET /mnt/media nfs defaults,nofail,x-systemd.automount,_netdev 0 0

Example:

192.168.1.30:/mnt/tank/media /mnt/media nfs defaults,nofail,x-systemd.automount,_netdev 0 0

Test it:

```text
systemctl daemon-reload
mount -a
ls /mnt/media
```

### Step 13 - Add Media Libraries in Jellyfin
Go to:

http://JELLYFIN-IP:8096

Then go to:

Dashboard -> Libraries -> Add Media Library

Example libraries:

Content type: Movies
Folder: /mnt/media/Movies

Content type: Shows
Folder: /mnt/media/TV Shows

Content type: Music
Folder: /mnt/media/Music

Save and let Jellyfin scan the library.

### Step 14 - Fix Permissions If Jellyfin Cannot See Files
Check the Jellyfin user:

```text
id jellyfin
```

Check the media folder:

```text
ls -lah /mnt/media
```

If using SMB, the uid=jellyfin,gid=jellyfin mount option usually fixes access.

If using NFS, adjust permissions on the TrueNAS dataset/share so the Jellyfin VM can read the files.

Jellyfin needs at least read access to the media files.

### Step 15 - Set Transcoding Path
Create a transcode folder:

```text
mkdir -p /var/cache/jellyfin/transcodes
chown -R jellyfin:jellyfin /var/cache/jellyfin
```

In Jellyfin, go to:

Dashboard -> Playback -> Transcoding

Set:

Transcode path: /var/cache/jellyfin/transcodes

Leave hardware acceleration disabled unless a GPU is passed through later.

### Step 16 - Restart Jellyfin
Restart Jellyfin:

```text
systemctl restart jellyfin
```

Check logs if needed:

```text
journalctl -u jellyfin -f
```

### Step 17 - Firewall Notes
If Proxmox firewall is disabled, no extra rule is needed.

If Proxmox firewall is enabled, allow:

## Tcp 8096

If using UFW inside Debian:

```text
apt install -y ufw
ufw allow ssh
ufw allow 8096/tcp
ufw enable
```

Check UFW:

```text
ufw status
```

### Step 18 - Add Another Jellyfin User
After creating the admin user, add more users from the Jellyfin web interface.

Steps:

1. Log into Jellyfin with the admin account.
2. Click the profile icon in the top-right.
3. Go to Dashboard -> Users.
4. Click + or Add User.
5. Enter the new user's username and password.
6. Choose what libraries they can access.
7. Save.

The new user can log in at:

http://JELLYFIN-IP:8096

Example:

http://192.168.1.50:8096

Recommended normal user settings:

- Can access server: enabled
- Manage Server: disabled
- Delete Media: disabled
- Download Media: optional
- Library access: only the libraries you want them to see

### Step 19 - Useful Jellyfin Commands
Check status:

```text
systemctl status jellyfin
```

Restart:

```text
systemctl restart jellyfin
```

Stop:

```text
systemctl stop jellyfin
```

Start:

```text
systemctl start jellyfin
```

View logs live:

```text
journalctl -u jellyfin -f
```

Update Jellyfin:

```text
apt update
apt upgrade -y
```

Check listening port:

```text
ss -tulpn | grep 8096
```

### Final Notes
The clean setup is:

- Debian 13.4 VM for Jellyfin
- 8 vCPU
- 8 GB RAM
- 64 GB disk
- Static IP
- Jellyfin installed from the official repo
- Media mounted from TrueNAS using SMB or NFS
- No LSI passthrough to Jellyfin
- Direct Play preferred
- Avoid heavy 4K transcoding unless GPU passthrough is added later
