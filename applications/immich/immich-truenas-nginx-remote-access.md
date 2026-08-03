<!--
Organized from: STEP 13 Immich TrueNAS Nginx Remote Access.txt
Source wording and technical details were preserved as closely as possible.
-->

> [!CAUTION]
> This is a historical implementation record. Verify versions, addresses, paths, and commands against the live environment before applying changes.

# STEP 13 - Immich Home Lab Deployment With TrueNAS Storage

### Goal

Deploy Immich in the home lab using:

- A dedicated Debian VM for Immich
- TrueNAS as the destination storage for Immich photos and videos
- PostgreSQL stored locally on the Immich VM
- Existing ubuntu-cloudflare VM as the Nginx reverse proxy
- Cloudflare DNS only
- Verizon CR1000A port forwarding
- Certbot for HTTPS
- No Caddy
- No VPN
- No Cloudflare Tunnel for Immich
- No Technitium DNS steps


### Final Architecture

Internet
    |
    | HTTPS: TCP 443
    v
Cloudflare DNS-only record
immich.zaman-labs.dev -> Verizon public IPv4
    |
    v
Verizon CR1000A
TCP 80 and TCP 443 port forwarding
    |
    v
ubuntu-cloudflare VM
192.168.1.254
Nginx + Certbot
    |
    | HTTP 2283 over LAN
    v
Debian Immich VM
`<IMMICH_VM_IP>`
Docker Compose
    |
    |-- PostgreSQL database -> /opt/immich/postgres on local VM disk
    |
    `-- Immich media -> /media/immich over NFS
                              |
                              v
                    TrueNAS: 192.168.1.253
                    /mnt/SeagateNAS/Immich


### Important Rule

The Immich media library can live on TrueNAS through NFS.

The Immich PostgreSQL database must stay on the Immich VM's local disk.
Do not place PostgreSQL on TrueNAS, NFS, SMB, or any other network share.


### Address Plan

TrueNAS:

```text
192.168.1.253
```

Nginx reverse proxy VM:

```text
192.168.1.254
```

Immich VM:
`<IMMICH_VM_IP>`

Public hostname:
immich.zaman-labs.dev

TrueNAS dataset:

```text
 /mnt/SeagateNAS/Immich
```

Debian NFS mount:

```text
 /media/immich
```

Immich configuration directory:

```text
 /opt/immich
```

Immich PostgreSQL directory:

```text
 /opt/immich/postgres
```

Replace every instance of `<IMMICH_VM_IP>` in this file with the actual static IP
address of the Debian Immich VM.


## PART 1 - CREATE THE IMMICH VM

### 1. Create a Debian VM in Proxmox

Create a full VM, not an LXC container.

Recommended settings:

Operating system:
Debian 13 or Debian 12

CPU:
4 to 6 cores

CPU type:
host

RAM:
8 GB minimum, 12 GB preferred

Disk:
80 GB to 100 GB

Disk location:
Local SSD-backed Proxmox storage

Network adapter:
VirtIO

QEMU Guest Agent:
Enabled

Start at boot:
Enabled

Give the VM a static DHCP reservation in the Verizon CR1000A.


### 2. Prepare Debian

Run this inside the Debian Immich VM:

```text
sudo apt update
sudo apt full-upgrade -y
```

sudo apt install -y \
  qemu-guest-agent \
  nfs-common \
  ca-certificates \
  curl \
  wget \
  openssl \
  nano

Enable QEMU Guest Agent:

```text
sudo systemctl enable --now qemu-guest-agent
```

Check the VM address:

```text
ip -br address
```

Reboot:

```text
sudo reboot
```


## PART 2 - CONFIGURE TRUENAS STORAGE

### 3. Create the immich-nfs service account

In TrueNAS, go to:

Credentials -> Users -> Add

Create the user with these settings:

Username:
immich-nfs

Full Name:
Immich NFS Service

SMB Access:
Off

TrueNAS Access:
Off

Shell Access:
Off

SSH Access:
Off

Disable Password:
On

Primary group:
Create new primary group named immich-nfs

Auxiliary groups:
None

Home Directory:

```text
/var/empty
```

Sudo Commands:
None

This account exists only so TrueNAS has a local identity to map NFS writes to.
It should not have login, shell, SSH, SMB, sudo, or admin access.


### 4. Create the Immich dataset

In TrueNAS, go to:

Datasets -> SeagateNAS -> Add Dataset

Use:

Name:
Immich

Preset:
Generic

ACL Type:
POSIX

Case Sensitivity:
Sensitive

Atime:
Off

Sync:
Standard

Compression:
Inherit/default

The final path should be:

```text
/mnt/SeagateNAS/Immich
```


### 5. Configure the dataset ACL

Go to:

Datasets -> SeagateNAS/Immich -> Permissions -> Edit

Leave ownership as:

Owner:
root

Owner Group:
root

Create or confirm these POSIX ACL entries:

User Obj - root:
Read, Write, Execute

Group Obj - root:
Read, Write, Execute

User - immich-nfs:
Read, Write, Execute

Mask:
Read, Write, Execute

Other:
No permissions

For the immich-nfs entry:

1. Click Add Item.
2. Set Who to User.
3. Select immich-nfs.
4. Enable Read.
5. Enable Write.
6. Enable Execute.
7. Leave Default unchecked.

For the Mask entry:

1. Click Add Item.
2. Set Who to Mask.
3. Enable Read.
4. Enable Write.
5. Enable Execute.
6. Leave Default unchecked.

For the Other entry:

1. Click Add Item.
2. Set Who to Other.
3. Leave Read disabled.
4. Leave Write disabled.
5. Leave Execute disabled.
6. Leave Default unchecked.

If this dataset is brand new and empty, enable Apply permissions recursively.

Save the ACL.

It is fine if the dataset still displays as root:root. The immich-nfs ACL entry
is what gives the mapped NFS user access.


### 6. Create the NFS share

Go to:

Shares -> Unix Shares (NFS) -> Add

Configure:

Path:

```text
/mnt/SeagateNAS/Immich
```

Description:
Immich Media Storage

Enabled:
On

Read Only:
Off

Hosts:
`<IMMICH_VM_IP>`

Mapall User:
immich-nfs

Mapall Group:
immich-nfs

Security:
SYS

Leave Maproot User and Maproot Group empty.

Save the share.

Then go to:

System -> Services -> NFS

Set:

Running:
Yes

Start Automatically:
Yes

Do not create an SMB share for this Immich dataset.


## PART 3 - MOUNT TRUENAS ON THE IMMICH VM

### 7. Create the mount point

On the Debian Immich VM:

```text
sudo mkdir -p /media/immich
```


### 8. Manually test the NFS mount

```text
sudo mount -t nfs4 \
  192.168.1.253:/mnt/SeagateNAS/Immich \
  /media/immich
```

Verify the mount:

findmnt /media/immich

Check that it is mounted read-write:

findmnt -no SOURCE,FSTYPE,OPTIONS /media/immich

The output must include:

rw


### 9. Test write access

Run:

```text
sudo touch /media/immich/immich-write-test
sudo ls -l /media/immich/immich-write-test
sudo rm /media/immich/immich-write-test
```

All three commands must succeed.

If the directory appears as root:root, that is acceptable. What matters is that
the write test succeeds.


### 10. Make the NFS mount persistent

Edit fstab:

```text
sudo nano /etc/fstab
```

Add this line:

192.168.1.253:/mnt/SeagateNAS/Immich /media/immich nfs4 rw,hard,noatime,_netdev,x-systemd.automount,x-systemd.mount-timeout=60 0 0

Test it:

sudo umount /media/immich
sudo systemctl daemon-reload
sudo mount -a
findmnt /media/immich

Repeat the write test:

```text
sudo touch /media/immich/immich-write-test
sudo rm /media/immich/immich-write-test
```

Do not continue until this succeeds.


## PART 4 - INSTALL DOCKER ON THE IMMICH VM

### 11. Remove conflicting packages

Run on the Debian Immich VM:

sudo apt remove -y \
  docker.io \
  docker-compose \
  docker-doc \
  podman-docker \
  containerd \
  runc || true


### 12. Add Docker's official repository

```text
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
```

Add Docker's signing key:

sudo curl -fsSL https://download.docker.com/linux/debian/gpg \
  -o /etc/apt/keyrings/docker.asc

```text
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add Docker's repository:

sudo tee /etc/apt/sources.list.d/docker.sources >/dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

Update APT:

```text
sudo apt update
```


### 13. Install Docker Engine and Docker Compose plugin

sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin

Enable Docker:

```text
sudo systemctl enable --now docker
```

Verify Docker:

```text
sudo systemctl status docker --no-pager
sudo docker run --rm hello-world
sudo docker compose version
```


### 14. Allow your admin user to run Docker

```text
sudo usermod -aG docker "$USER"
```

Disconnect from SSH completely:

exit

Reconnect and verify:

id
docker ps

The output of id should include:

docker

Do not run:

```text
chmod 666 /var/run/docker.sock
```

If docker ps still shows a permission error, log out and reconnect again, or
reboot the Immich VM.


## PART 5 - MAKE DOCKER WAIT FOR THE TRUENAS MOUNT

### 15. Create the systemd override

This prevents Immich from writing media to the VM's local disk if the NFS mount
is missing.

Run:

```text
sudo mkdir -p /etc/systemd/system/docker.service.d
```

Create the override file:

sudo tee /etc/systemd/system/docker.service.d/override.conf >/dev/null <<'EOF'
[Unit]
Wants=network-online.target
After=network-online.target
RequiresMountsFor=/media/immich
EOF

Reload systemd:

```text
sudo systemctl daemon-reload
```

Verify the override:

```text
sudo systemctl cat docker.service
```

Near the bottom, this should appear:

[Unit]
Wants=network-online.target
After=network-online.target
RequiresMountsFor=/media/immich

Restart Docker:

```text
sudo systemctl restart docker
```

Verify:

findmnt /media/immich
systemctl status docker --no-pager


## PART 6 - INSTALL IMMICH

### 16. Download the official Immich Compose files

On the Debian Immich VM:

```text
sudo mkdir -p /opt/immich
sudo chown "$USER":"$USER" /opt/immich
```

```text
cd /opt/immich
```

Download:

```text
wget -O docker-compose.yml \
  https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
```

```text
wget -O .env \
  https://github.com/immich-app/immich/releases/latest/download/example.env
```


### 17. Generate a database password

```text
openssl rand -hex 32
```

Copy the generated value.


### 18. Configure Immich

Open:

```text
nano /opt/immich/.env
```

Set these values:

```text
UPLOAD_LOCATION=/media/immich
DB_DATA_LOCATION=/opt/immich/postgres
```

TZ=America/New_York

```text
DB_PASSWORD=PASTE_THE_GENERATED_PASSWORD_HERE
```

Leave the supplied IMMICH_VERSION, DB username, DB name, and image values as
they are unless you intentionally need to change them.

Protect the .env file:

```text
chmod 600 /opt/immich/.env
```

The intended storage layout is:

/media/immich
|-- backups
|-- encoded-video
|-- library
|-- profile
|-- thumbs
`-- upload

/opt/immich/postgres
`-- Live PostgreSQL database


### 19. Validate the Compose file

Confirm the NFS mount exists first:

findmnt /media/immich

Then validate Compose:

```text
cd /opt/immich
docker compose config >/dev/null
```


### 20. Start Immich

```text
docker compose up -d
```

Check the containers:

```text
docker compose ps
```

Watch logs:

```text
docker compose logs -f --tail=100
```

Press Ctrl+C after startup settles.

Verify the Immich port:

```text
sudo ss -lntp | grep 2283
```

Open Immich locally:

http://`<IMMICH_VM_IP>`:2283

Create the first account. The first account becomes the Immich administrator.


### 21. Confirm media is writing to TrueNAS

Upload one test image.

Then run:

```text
sudo find /media/immich -maxdepth 3 -type f | head -20
```

You should see Immich-created files under the mounted TrueNAS dataset.


## PART 7 - CONFIGURE NGINX ON UBUNTU-CLOUDFLARE

The reverse proxy VM is:

ubuntu-cloudflare
192.168.1.254

This VM already runs Nginx. Use Nginx for Immich instead of Caddy.


### 22. Confirm Nginx can reach Immich

On ubuntu-cloudflare:

```text
curl -I http://<IMMICH_VM_IP>:2283
```

A valid HTTP response means the reverse proxy can reach Immich.


### 23. Back up Nginx

sudo cp -a /etc/nginx \
  "/etc/nginx.backup-$(date +%F-%H%M%S)"


### 24. Install required packages

```text
sudo apt update
```

sudo apt install -y \
  nginx \
  certbot \
  python3-certbot-nginx

Enable Nginx:

```text
sudo systemctl enable --now nginx
```


### 25. Create the Immich Nginx site

Create:

```text
sudo nano /etc/nginx/sites-available/immich
```

Paste this configuration and replace `<IMMICH_VM_IP>`:

server {
    listen 80;
    listen [::]:80;

    server_name immich.zaman-labs.dev;

    client_max_body_size 50000M;

    proxy_request_buffering off;
    client_body_buffer_size 1024k;

    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_http_version 1.1;
    proxy_redirect off;

    proxy_read_timeout 600s;
    proxy_send_timeout 600s;
    send_timeout 600s;

    location / {
        proxy_pass http://`<IMMICH_VM_IP>`:2283;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}

Example:

proxy_pass http://192.168.1.251:2283;

Enable the site:

```text
sudo ln -sfn \
  /etc/nginx/sites-available/immich \
  /etc/nginx/sites-enabled/immich
```

Validate Nginx:

```text
sudo nginx -t
```

Expected:

syntax is ok
test is successful

Reload Nginx:

```text
sudo systemctl reload nginx
```


### 26. Test Nginx hostname routing

Run on ubuntu-cloudflare:

curl -I \
  -H 'Host: immich.zaman-labs.dev' \
  http://127.0.0.1

Confirm the site is loaded:

sudo nginx -T 2>/dev/null | grep -A30 -B5 \
  'server_name immich.zaman-labs.dev'


### 27. Allow Nginx through UFW if UFW is active

Check:

```text
sudo ufw status
```

If active:

```text
sudo ufw allow 'Nginx Full'
sudo ufw reload
```


## PART 8 - CONFIGURE CLOUDFLARE DNS

### 28. Find the public IPv4

From a system on the home network:

curl -4 https://api.ipify.org
echo

Compare this with the WAN IPv4 address shown in the Verizon CR1000A.

The two addresses should match. If they do not match, there may be double NAT
or CGNAT.


### 29. Create the DNS record

In Cloudflare, go to:

zaman-labs.dev
-> DNS
-> Records
-> Add record

Use:

Type:
A

Name:
immich

IPv4 address:
Your Verizon public IPv4

Proxy status:
DNS only

TTL:
Auto

The cloud icon must be gray.

Do not create an AAAA record unless public IPv6 is intentionally configured.

Verify:

```text
dig +short immich.zaman-labs.dev @1.1.1.1
```

If dig is not installed:

```text
sudo apt install -y dnsutils
```

Compare the result with:

curl -4 https://api.ipify.org
echo

They should match.


## PART 9 - CONFIGURE VERIZON CR1000A PORT FORWARDING

### 30. Open the router

Open:

https://192.168.1.1

Sign in.

Go to:

Advanced
-> Security & Firewall
-> Port Forwarding


### 31. Add the HTTP rule

Application:
Immich HTTP

Original Port:
80

Protocol:
TCP

Fwd to Addr:
192.168.1.254 - ubuntu-server

Fwd to Port:
80

Schedule:
Always

Click Add to list.


### 32. Add the HTTPS rule

Application:
Immich HTTPS

Original Port:
443

Protocol:
TCP

Fwd to Addr:
192.168.1.254 - ubuntu-server

Fwd to Port:
443

Schedule:
Always

Click Add to list.

Make sure both rules are enabled, then click Apply Changes.

Final rules:

```text
TCP 80  -> 192.168.1.254:80
TCP 443 -> 192.168.1.254:443
```

Do not forward:

2283  Immich backend
2049  NFS
445   SMB
5432  PostgreSQL
8006  Proxmox
22    SSH

Do not configure a DMZ host.


## PART 10 - TEST PUBLIC HTTP

### 33. Test from cellular data

On a phone:

1. Disable Wi-Fi.
2. Disable any VPN.
3. Use cellular data.
4. Open:

http://immich.zaman-labs.dev

At the same time, watch Nginx on ubuntu-cloudflare:

```text
sudo tail -f \
  /var/log/nginx/access.log \
  /var/log/nginx/error.log
```

A new access-log entry means DNS and Verizon port forwarding are reaching
Nginx.

Do not request the HTTPS certificate until public HTTP reaches the server.


## PART 11 - ENABLE HTTPS WITH CERTBOT

### 34. Request the certificate

Run on ubuntu-cloudflare:

sudo certbot --nginx --redirect \
  -d immich.zaman-labs.dev

During setup:

1. Enter an email address.
2. Accept the terms.
3. Permit the HTTPS redirect.


### 35. Validate HTTPS

```text
sudo nginx -t
sudo systemctl reload nginx
```

Check listening ports:

```text
sudo ss -lntup | grep -E ':(80|443)\b'
```

Nginx should listen on both 80 and 443.

Test the HTTP redirect:

curl -I \
  -H 'Host: immich.zaman-labs.dev' \
  http://127.0.0.1

Expected:

HTTP/1.1 301 Moved Permanently
Location: https://immich.zaman-labs.dev/

Test local HTTPS:

curl -I \
  --resolve immich.zaman-labs.dev:443:127.0.0.1 \
  https://immich.zaman-labs.dev/

Test Immich discovery:

curl -I \
  --resolve immich.zaman-labs.dev:443:127.0.0.1 \
  https://immich.zaman-labs.dev/.well-known/immich


### 36. Test certificate renewal

```text
sudo certbot renew --dry-run
```

Check the timer:

```text
systemctl status certbot.timer --no-pager
```


### 37. Test public HTTPS

On cellular data, open:

https://immich.zaman-labs.dev

Expected result:

- Valid HTTPS certificate
- No browser warning
- Immich setup or login page


## PART 12 - FINISH IMMICH CONFIGURATION

### 38. Set the external domain

In Immich, go to:

Administration
-> Settings
-> Server Settings
-> External Domain

Set:

https://immich.zaman-labs.dev

Do not include a trailing slash.


### 39. Configure the mobile app

In the Immich mobile app, use:

https://immich.zaman-labs.dev

Then:

1. Sign in.
2. Open backup settings.
3. Select one small album.
4. Enable backup.
5. Confirm successful uploads.
6. Then select the full phone library.

Confirm files are reaching TrueNAS:

```text
sudo find /media/immich -maxdepth 3 -type f | head -20
```


## PART 13 - BACKUPS AND SNAPSHOTS

### 40. Configure Immich database backups

In Immich, go to:

Administration
-> Settings
-> Backup

Recommended:

Enabled:
Yes

Schedule:
Daily at 2:00 AM

Retention:
30 backups

Immich stores database dumps in:

```text
/media/immich/backups
```

Create a manual test backup from:

Administration
-> Job Queues
-> Create job
-> Create Database Dump

Confirm it exists:

```text
ls -lh /media/immich/backups
```


### 41. Configure TrueNAS snapshots

In TrueNAS, go to:

Data Protection
-> Periodic Snapshot Tasks
-> Add

Use:

Dataset:
SeagateNAS/Immich

Recursive:
On

Schedule:
Every 6 hours

Lifetime:
30 days

Snapshots are not a full backup by themselves because they remain on the same
TrueNAS pool. Keep another copy of important media and database backups on a
separate physical system or offsite location.


### 42. Continue Proxmox VM backups

The Proxmox backup of the Immich VM should protect:

```text
/opt/immich
/opt/immich/postgres
/etc/fstab
/etc/systemd/system/docker.service.d
```

The TrueNAS dataset protects:

```text
/media/immich
```

Complete recovery requires:

1. Immich database
2. Immich media dataset
3. Compose file and .env configuration


## PART 14 - STARTUP ORDER

If TrueNAS and Immich run on the same Proxmox host, configure startup ordering.

TrueNAS VM:

Start at boot:
Yes

Order:
1

Startup delay:
120 seconds

Immich VM:

Start at boot:
Yes

Order:
2

Startup delay:
30 seconds

Docker also has the systemd mount dependency for /media/immich.


## PART 15 - REBOOT VALIDATION

### 43. Test the complete sequence

Reboot TrueNAS first and wait until it is fully online.

Then reboot Immich:

```text
sudo reboot
```

After reconnecting to Immich:

findmnt /media/immich
df -h /media/immich

```text
cd /opt/immich
docker compose ps
```

Verify:

- /media/immich is mounted from 192.168.1.253
- The mount is read-write
- All Immich containers are running
- Existing images load
- New test upload succeeds
- Public HTTPS works over cellular

If /media/immich is missing, do not start Immich manually. Fix NFS first.


## PART 16 - UPDATING IMMICH

Before updating:

1. Confirm a recent database dump exists.
2. Create a manual TrueNAS snapshot.
3. Confirm the NFS mount:

findmnt /media/immich

Update Immich:

```text
cd /opt/immich
docker compose pull
docker compose up -d
```

Check:

```text
docker compose ps
docker compose logs --tail=100
```

Review Immich release notes before major upgrades.


## TROUBLESHOOTING

### NFS write test says permission denied

Confirm the TrueNAS POSIX ACL contains:

User Obj root:       rwx
Group Obj root:      rwx
User immich-nfs:     rwx
Mask:                rwx
Other:               ---

Confirm the NFS share has:

Mapall User:
immich-nfs

Mapall Group:
immich-nfs

Read Only:
Off

Then restart NFS on TrueNAS and remount on Debian:

```text
sudo umount /media/immich
sudo mount -a
```


### NFS mount shows root:root

This is fine if the write test succeeds.


### Docker socket permission denied

Run:

sudo usermod -aG docker "$USER"
exit

Reconnect and verify:

id
docker ps


### Nginx shows the default page

Test with the correct hostname:

curl -I \
  -H 'Host: immich.zaman-labs.dev' \
  http://127.0.0.1

Check enabled sites:

```text
ls -l /etc/nginx/sites-enabled/
```

Check loaded config:

sudo nginx -T | grep -A20 \
  'server_name immich.zaman-labs.dev'


### Nginx returns 502 Bad Gateway

From ubuntu-cloudflare:

```text
curl -I http://<IMMICH_VM_IP>:2283
```

On the Immich VM:

```text
cd /opt/immich
docker compose ps
sudo ss -lntp | grep 2283
```


### Public connection times out

Check DNS:

```text
dig +short immich.zaman-labs.dev @1.1.1.1
```

Check public IP:

curl -4 https://api.ipify.org
echo

They must match.

Confirm CR1000A forwards:

```text
TCP 80  -> 192.168.1.254:80
TCP 443 -> 192.168.1.254:443
```

Check Nginx:

```text
sudo ss -lntup | grep -E ':(80|443)\b'
sudo tail -f /var/log/nginx/access.log
```


### HTTP returns 301 redirect

After Certbot is configured, this is correct:

HTTP/1.1 301 Moved Permanently
Location: https://immich.zaman-labs.dev/


### Large uploads return 413

Make sure Cloudflare is set to DNS only for the immich record.
The cloud icon should be gray, not orange.


## FINAL CHECKLIST

[ ] Immich VM has a static IP
[ ] PostgreSQL is stored locally on the Immich VM
[ ] TrueNAS dataset is /mnt/SeagateNAS/Immich
[ ] Dataset uses POSIX ACL
[ ] immich-nfs ACL entry has rwx
[ ] POSIX Mask has rwx
[ ] POSIX Other entry exists with no access
[ ] NFS Mapall User is immich-nfs
[ ] NFS Mapall Group is immich-nfs
[ ] NFS share permits only the Immich VM
[ ] /media/immich passes the write test
[ ] /media/immich is configured in /etc/fstab
[ ] Docker requires /media/immich
[ ] UPLOAD_LOCATION is /media/immich
[ ] DB_DATA_LOCATION is /opt/immich/postgres
[ ] Immich works internally on port 2283
[ ] Nginx runs on 192.168.1.254
[ ] Nginx proxies to the Immich VM
[ ] Cloudflare record is immich.zaman-labs.dev
[ ] Cloudflare record is DNS only
[ ] CR1000A forwards TCP 80 and 443 to 192.168.1.254
[ ] Port 2283 is not forwarded publicly
[ ] Certbot certificate works
[ ] HTTP redirects to HTTPS
[ ] /.well-known/immich reaches Immich
[ ] External Domain is set in Immich
[ ] Database backups are enabled
[ ] TrueNAS snapshots are enabled
[ ] Full reboot validation succeeds
