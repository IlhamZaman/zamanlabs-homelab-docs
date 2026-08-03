<!--
Organized from: STEP 9 Kuma Monitoring(2).txt
Source wording and technical details were preserved as closely as possible.
-->

> [!CAUTION]
> This is a historical implementation record. Verify versions, addresses, paths, and commands against the live environment before applying changes.

# STEP 9 Kuma Monitoring

Goal:
Install and configure Uptime Kuma on a Fedora Server 44 VM in Proxmox without Docker, then use it to monitor the homelab services. Uptime Kuma was also added behind the existing Ubuntu Cloudflare Tunnel using the public hostname kuma.zaman-labs.dev.

## Step 1: Fedora Server 44 VM Plan

I installed Uptime Kuma on a Fedora Server 44 virtual machine in Proxmox without using Docker.

Recommended VM settings:
- OS: Fedora Server 44
- BIOS: SeaBIOS is fine
- CPU: 2 cores
- RAM: 2 GB minimum, 4 GB preferred
- Disk: 20 GB minimum
- Network: VirtIO bridge
- IP address: static IP recommended

The Uptime Kuma VM ended up using this local IP:
- 192.168.1.226

The local Uptime Kuma web interface is:
- http://192.168.1.226:3001


## Step 2: Update Fedora

After Fedora was installed, I updated the server:

```text
sudo dnf upgrade --refresh -y
sudo reboot
```

After the reboot, I logged back into the Fedora VM over SSH.


## Step 3: Install Required Packages

I installed the packages needed for a source-based Uptime Kuma install:

```text
sudo dnf install -y git nodejs npm sqlite gcc gcc-c++ make python3
```

Then I checked the versions:

```text
node -v
npm -v
git --version
```

Uptime Kuma needs a modern Node.js version. My logs later showed Node.js 22.22.2, which was working.


## Step 4: Install Uptime Kuma Without Docker

I installed Uptime Kuma from source instead of using Docker.

The general process was:

```text
git clone https://github.com/louislam/uptime-kuma.git
cd uptime-kuma
npm run setup
```

Then I tested it manually:

```text
node server/server.js
```

The web UI should load at:

http://192.168.1.226:3001


## Step 5: Open Fedora Firewall Port

Since Uptime Kuma runs on port 3001, I opened that port in Fedora's firewall:

```text
sudo firewall-cmd --add-port=3001/tcp --permanent
sudo firewall-cmd --reload
```

Then I verified the open ports:

```text
sudo firewall-cmd --list-ports
```

Expected result:

3001/tcp


## Step 6: Run Uptime Kuma with PM2

I installed and used PM2 so Uptime Kuma can stay running in the background:

```text
npm install pm2 -g
pm2 install pm2-logrotate
```

Then I started Uptime Kuma:

```text
pm2 start server/server.js --name uptime-kuma
```

Then I saved the PM2 process list:

```text
pm2 save
```

To check the status:

```text
pm2 status
```


## Step 7: PM2 Startup Issue and Fix

When trying to enable PM2 startup, I originally ran:

```text
sudo env PATH=$PATH:/usr/bin pm2 startup systemd -u izaman --hp /home/izaman
```

But it failed with:

env: ‘pm2’: No such file or directory

That meant root could not find the pm2 command. The fix was to find where pm2 was installed:

```text
which pm2
```

If pm2 is installed at /usr/local/bin/pm2, the startup command should include /usr/local/bin in the PATH:

```text
sudo env PATH=$PATH:/usr/bin:/usr/local/bin pm2 startup systemd -u izaman --hp /home/izaman
```

If pm2 is installed inside the user's npm global path, then the full PM2 path needs to be used.

After fixing the PATH issue, I saved PM2 again:

```text
pm2 save
```

Then I checked the systemd service:

```text
sudo systemctl status pm2-izaman
```


## Step 8: Uptime Kuma Database Choice

When Uptime Kuma first opened, it asked which database to use.

I chose:

SQLite

Reason:
- This is a homelab monitoring setup, not a huge enterprise deployment.
- SQLite is simpler.
- No separate MariaDB or MySQL server is needed.
- Backups are easier.
- Less stuff can break.

MariaDB/MySQL was not needed for this setup.


## Step 9: First Uptime Kuma Login

After choosing SQLite, I created the Uptime Kuma admin account and logged in.

Then I started adding monitors for my homelab services.


## Step 10: Monitors Added

I added monitors for services like:

- Jellyfin
- Minecraft Server
- Proxmox
- Technitium DNS
- Technitium Web UI
- TrueNAS
- Ubuntu Cloudflare Tunnel VM

The dashboard showed several services online with green percentages, including:
- Jellyfin: 100%
- Minecraft Server: 100%
- Proxmox: 100%
- Technitium DNS: 100%
- Technitium Web UI: 100%
- TrueNAS: around 90% at the time of the screenshot


## Step 11: Proxmox Monitor Settings

For Proxmox, the correct monitor setup is:

Monitor Type:
HTTP(s)

Friendly Name:
Proxmox

URL:

```text
https://192.168.1.242:8006
```

Important:
Proxmox uses a self-signed certificate, so I needed to enable:

Ignore TLS/SSL error

One issue I saw in the logs was that Proxmox tried connecting to port 443 at one point:

connect ECONNREFUSED 192.168.1.242:443

That means the monitor was pointed at the wrong port. Proxmox should use port 8006, not 443.


## Step 12: TrueNAS Monitor Settings

For TrueNAS, the monitor can be:

Monitor Type:
HTTP(s)

URL:

```text
http://192.168.1.xxx
```

or:

https://192.168.1.xxx

If using HTTPS, I need to enable:

Ignore TLS/SSL error

The logs showed a self-signed certificate warning for TrueNAS, which is normal unless the monitor is configured to ignore TLS errors.


## Step 13: Technitium DNS Monitor Settings

For Technitium DNS itself, I used a TCP Port monitor.

Monitor Type:
TCP Port

Friendly Name:
Technitium DNS

Hostname:
192.168.1.xxx

Port:
53

SSL/TLS:
Disabled / None

Important:
For TCP Port monitors, do not use http:// or https:// in the hostname. Only use the IP address.


## Step 14: Technitium Web UI Monitor Settings

For the Technitium web dashboard, I used:

Monitor Type:
HTTP(s)

Friendly Name:
Technitium Web UI

URL:

```text
http://192.168.1.xxx:5380
```


## Step 15: Jellyfin Monitor Settings

For Jellyfin, I used:

Monitor Type:
HTTP(s)

Friendly Name:
Jellyfin

URL:

```text
http://192.168.1.xxx:8096
```


## Step 16: Minecraft Server Monitor Settings

For the Minecraft server, I used a TCP Port monitor.

Monitor Type:
TCP Port

Friendly Name:
Minecraft Server

Hostname:
192.168.1.xxx

Port:
25565

If the Minecraft server uses a custom port, then I should use that custom port instead.


## Step 17: Ubuntu Cloudflare Tunnel VM Monitor

I also added a monitor for the Ubuntu Cloudflare Tunnel VM.

Monitor Type:
Ping

Friendly Name:
Ubuntu Cloudflare Tunnel VM

Hostname:
The local IP address of the Ubuntu Cloudflare Tunnel VM

Important mistake:
I accidentally typed:

## 192.168.1.294

That is not a valid IP address because IP address octets only go from 0 to 255. So the last number cannot be 294.

I need to replace it with the real local IP of the Ubuntu Cloudflare Tunnel VM.


## Step 18: Adding Uptime Kuma to the Existing Ubuntu Cloudflare Tunnel

I added Uptime Kuma to my existing Ubuntu Cloudflare Tunnel setup.

The public hostname I used was:

kuma.zaman-labs.dev

The tunnel service should point to the local Uptime Kuma service:

http://192.168.1.226:3001

or, if the tunnel connector is running directly on the same VM as Uptime Kuma:

http://localhost:3001

In my setup, I was using the Ubuntu Cloudflare Tunnel VM, so the tunnel public hostname should route to:

http://192.168.1.226:3001

Important:
The Cloudflare Tunnel service should use HTTP to the origin unless HTTPS is actually configured on the Uptime Kuma server itself.

Correct:

```text
http://192.168.1.226:3001
```

Do not use this unless Uptime Kuma itself has HTTPS configured:

```text
https://192.168.1.226:3001
```

The public Uptime Kuma domain is:

https://kuma.zaman-labs.dev


## Step 19: WebSocket Error I Saw

I kept seeing this red error at the top of Uptime Kuma:

Cannot connect to the socket server. [Error: xhr poll error] Reconnecting...
Using a Reverse Proxy? Check how to config it for WebSocket

At first, I thought I was using the local IP, but the browser was still showing:

kuma.zaman-labs.dev/add

That meant I was still going through the Cloudflare domain.

When using the actual local IP, the browser should show:

http://192.168.1.226:3001/add

After switching to the local IP, the red WebSocket error still appeared, so I checked the PM2 logs.


## Step 20: PM2 Logs Result

I checked the logs with:

```text
pm2 logs uptime-kuma --lines 80
```

The logs showed that Uptime Kuma was running correctly.

Important log lines:

Welcome to Uptime Kuma
Your Node.js version: 22.22.2
Uptime Kuma Version: 2.3.2
Database Type: sqlite
Connected to the database
Listening on:
- http://localhost:3001
- http://192.168.1.226:3001

The logs also showed that my browser connected successfully:

[SOCKET] INFO: New polling connection, IP = 192.168.1.231
[AUTH] INFO: Login by token. IP=192.168.1.231
[AUTH] INFO: Username from JWT: admin
[AUTH] INFO: Successfully logged in user admin. IP=192.168.1.231

Conclusion:
The Uptime Kuma backend was healthy. It was listening on the local IP, using SQLite correctly, and accepting my login.


## Step 21: WebSocket Error Troubleshooting

Since the backend was healthy, the WebSocket error was probably caused by one of these:

- Browser VPN
- Browser proxy
- Firefox extension
- Privacy/adblock extension
- Cloudflare Tunnel/reverse proxy WebSocket issue
- Old cached site data
- Using the Cloudflare domain instead of the local IP

The browser toolbar showed VPN enabled, so the first thing to do was turn off the VPN and test again locally.

Local test URL:

http://192.168.1.226:3001

Hard refresh on Mac:

Cmd + Shift + R


## Step 22: Browser Fixes to Try

To fix the red socket error, I should try:

1. Turn off the browser VPN.
2. Open Uptime Kuma in a private Firefox window.
3. Disable extensions temporarily, especially:
   - VPN extensions
   - adblockers
   - privacy blockers
   - HTTPS-only extensions
   - tracking blockers
4. Clear site data for:
   - 192.168.1.226
   - kuma.zaman-labs.dev

Firefox path:

Settings > Privacy & Security > Cookies and Site Data > Manage Data

Then search for and remove:

192.168.1.226
kuma.zaman-labs.dev

Then reopen:

http://192.168.1.226:3001


## Step 23: Useful Uptime Kuma Commands

Check PM2 status:

```text
pm2 status
```

Restart Uptime Kuma:

```text
pm2 restart uptime-kuma
```

View logs:

```text
pm2 logs uptime-kuma --lines 80
```

Save PM2 process list:

```text
pm2 save
```

Check what is listening on port 3001:

```text
ss -tulpn | grep 3001
```

Expected result should include:

## 0.0.0.0:3001

or show the server listening on:

## 192.168.1.226:3001


## Step 24: Backing Up Uptime Kuma

The most important folder to back up is the data folder inside the Uptime Kuma install directory.

Common path if installed in the user home:

```text
/home/izaman/uptime-kuma/data
```

or if installed in /opt:

```text
/opt/uptime-kuma/data
```

Backup command example:

```text
tar -czvf uptime-kuma-backup-$(date +%F).tar.gz /home/izaman/uptime-kuma/data
```

This backup should be copied to TrueNAS or another safe location.


## Step 25: Final Notes

Uptime Kuma is installed and running without Docker on Fedora Server 44.

Local Uptime Kuma URL:

http://192.168.1.226:3001

Cloudflare Tunnel public URL:

https://kuma.zaman-labs.dev

The backend is healthy based on the PM2 logs. The socket error is not from Uptime Kuma failing to start. It is most likely related to browser VPN/extensions, cached site data, or the Cloudflare Tunnel/reverse proxy path.

The main things to fix next are:
- Turn off VPN when testing local LAN
- Clear browser site data
- Make sure kuma.zaman-labs.dev points to http://192.168.1.226:3001 in the Ubuntu Cloudflare Tunnel
- Fix the invalid Ubuntu Tunnel monitor IP, because 192.168.1.294 is not valid
- Use Ignore TLS/SSL error for Proxmox and TrueNAS HTTPS monitors
