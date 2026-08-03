<!--
Organized from: STEP 10 Homepage Dashboard Setup(2).txt
Source wording and technical details were preserved as closely as possible.
-->

> [!CAUTION]
> This is a historical implementation record. Verify versions, addresses, paths, and commands against the live environment before applying changes.

## STEP 10 Homepage Dashboard Setup

This step covers setting up Homepage on the existing Fedora Uptime Kuma VM, customizing the dashboard style, adding service cards, removing default bookmark sections, adding icons, and adding Proxmox/TrueNAS widgets.

---
# 1. What Homepage Is For

Homepage is a self-hosted dashboard for a homelab. It does not replace Uptime Kuma.

Uptime Kuma monitors whether services are online or offline.
Homepage acts as the clean front page where I can organize links and widgets for services like Proxmox, TrueNAS, Jellyfin, Technitium DNS, Uptime Kuma, Cloudflare, and other homelab tools.

Homepage was installed on the same Fedora VM as Uptime Kuma.

Typical ports:

Homepage: http://FEDORA-VM-IP:3000
Uptime Kuma: http://FEDORA-VM-IP:3001

---
## 2. Installing Homepage on Fedora Without Docker

SSH into the Fedora VM:

```text
ssh youruser@YOUR-FEDORA-VM-IP
```

Update Fedora:

```text
sudo dnf update -y
```

Install required packages:

```text
sudo dnf install -y git nodejs npm firewalld
```

Check Node and npm:

```text
node -v
npm -v
```

Install pnpm:

```text
sudo npm install -g pnpm
```

Check pnpm:

```text
pnpm -v
```

Create a dedicated Homepage system user:

```text
sudo useradd --system --create-home --home-dir /opt/homepage --shell /sbin/nologin homepage
```

Download Homepage:

```text
sudo git clone https://github.com/gethomepage/homepage.git /opt/homepage/app
```

Fix ownership:

```text
sudo chown -R homepage:homepage /opt/homepage
```

Install and build Homepage:

```text
cd /opt/homepage/app
sudo -u homepage pnpm install
sudo -u homepage pnpm build
```

Create the config folder:

```text
sudo -u homepage cp -r /opt/homepage/app/src/skeleton /opt/homepage/app/config
```

---
## 3. Creating the systemd Service

Find the Fedora VM IP:

```text
ip a
```

Create the systemd service:

```text
sudo nano /etc/systemd/system/homepage.service
```

Example service file:

```text
[Unit]
Description=Homepage Dashboard
After=network.target
```

```text
[Service]
Type=simple
User=homepage
Group=homepage
WorkingDirectory=/opt/homepage/app
Environment=HOMEPAGE_ALLOWED_HOSTS=YOUR-FEDORA-VM-IP:3000,localhost:3000,127.0.0.1:3000
ExecStart=/usr/local/bin/pnpm start
Restart=always
RestartSec=10
```

```text
[Install]
WantedBy=multi-user.target
```

Replace YOUR-FEDORA-VM-IP with the real IP address of the Fedora VM.

Save the file:

CTRL + O
ENTER
CTRL + X

Enable and start Homepage:

```text
sudo systemctl daemon-reload
sudo systemctl enable --now homepage
```

Check the service:

```text
systemctl status homepage
```

View logs if needed:

```text
journalctl -u homepage -f
```

---
## 4. Opening the Firewall Port

Homepage uses port 3000 by default.

Enable firewalld and open the port:

```text
sudo systemctl enable --now firewalld
sudo firewall-cmd --add-port=3000/tcp --permanent
sudo firewall-cmd --reload
```

Then open Homepage in the browser:

http://YOUR-FEDORA-VM-IP:3000

---
## 5. Main Config Files

Homepage uses separate YAML files for different parts of the dashboard.

Main files used:

```text
/opt/homepage/app/config/settings.yaml
/opt/homepage/app/config/services.yaml
/opt/homepage/app/config/widgets.yaml
/opt/homepage/app/config/bookmarks.yaml
```

What each file does:

```text
settings.yaml  = Theme, background, blur, card style, layout columns
services.yaml  = Main service cards and service widgets
widgets.yaml   = Top widgets like date, weather, search, local resources
bookmarks.yaml = Bookmark sections like Developer, Social, Entertainment
```

Important note:

The Proxmox widget does not go in widgets.yaml.
It goes inside services.yaml under the Proxmox service card.

---
## 6. Setting a Background Image

Create an images folder:

```text
sudo mkdir -p /opt/homepage/app/public/images
```

Place the background image in this folder.

Example image path used:

```text
/opt/homepage/app/public/images/bluehour.jpg
```

If copying from another computer:

```text
scp bluehour.jpg youruser@YOUR-FEDORA-VM-IP:/tmp/bluehour.jpg
```

Then move it into place:

```text
sudo mv /tmp/bluehour.jpg /opt/homepage/app/public/images/bluehour.jpg
sudo chown homepage:homepage /opt/homepage/app/public/images/bluehour.jpg
```

---
## 7. Final settings.yaml Used

Open settings.yaml:

```text
sudo nano /opt/homepage/app/config/settings.yaml
```

Final settings used:

---
```text
title: Zaman Labs
theme: dark
color: slate
headerStyle: clean
statusStyle: dot
target: _blank
```

background:

```text
  image: /images/bluehour.jpg
  blur: lg
  opacity: 65
```

cardBlur: sm

hideVersion: true

layout:

```text
  System:
    style: row
    columns: 1
```

  Monitoring:

```text
    style: row
    columns: 2
```

  Entertainment:

```text
    style: row
    columns: 2
```

  Infrastructure:

```text
    style: row
    columns: 2
```

  Network:

```text
    style: row
    columns: 3
```

  Links:

```text
    style: row
    columns: 2
```

Save:

CTRL + O
ENTER
CTRL + X

Restart Homepage:

```text
sudo systemctl restart homepage
```

Note:

The section names under layout must match the group names in services.yaml exactly.
For example, if settings.yaml has "Infrastructure", then services.yaml also needs a group named "Infrastructure".

---
## 8. Background Blur

The background blur is controlled inside settings.yaml under the background section.

Example:

background:

```text
  image: /images/bluehour.jpg
  blur: lg
  opacity: 65
```

Blur options can include:

sm
md
lg

If the background is too blurry or washed out, try:

background:

```text
  image: /images/bluehour.jpg
  blur: md
  opacity: 50
```

Restart after changes:

```text
sudo systemctl restart homepage
```

---
## 9. Removing the Default Bookmark Section

The default Developer, Social, and Entertainment bookmark row comes from bookmarks.yaml.

Open it:

```text
sudo nano /opt/homepage/app/config/bookmarks.yaml
```

To remove all default bookmarks, replace the whole file with:

---

Save and restart:

```text
sudo systemctl restart homepage
```

---
## 10. Adding Icons to Bookmarks

Bookmarks can use icons too.

Open bookmarks.yaml:

```text
sudo nano /opt/homepage/app/config/bookmarks.yaml
```

Example:

---
- Developer:
    - GitHub:
        - icon: github.png
          href: https://github.com

    - ChatGPT:
        - icon: openai.png
          href: https://chatgpt.com

- Social:
    - Reddit:
        - icon: reddit.png
          href: https://reddit.com

- Entertainment:
    - YouTube:
        - icon: youtube.png
          href: https://youtube.com

Restart:

```text
sudo systemctl restart homepage
```

Common icon names:

github.png
reddit.png
youtube.png
cloudflare.png
proxmox.png
truenas.png
jellyfin.png
uptime-kuma.png

---
## 11. Top widgets.yaml Example

widgets.yaml is only for top info widgets like date, weather, and local resources.

Open:

```text
sudo nano /opt/homepage/app/config/widgets.yaml
```

Example:

---
- datetime:

```text
    text_size: xl
    format:
      dateStyle: short
      timeStyle: short
```

- openmeteo:

```text
    label: Philadelphia
    latitude: 39.9526
    longitude: -75.1652
    timezone: America/New_York
    units: imperial
    cache: 5
```

Restart:

```text
sudo systemctl restart homepage
```

Note:

The resources widget in widgets.yaml only monitors the Fedora VM where Homepage is installed.
It does not monitor Proxmox.

Example Fedora-only resources widget:

- resources:

```text
    cpu: true
    memory: true
    disk: /
```

This was removed because I wanted Proxmox resource usage instead of Fedora VM usage.

---
## 12. Proxmox Widget Permissions

For Homepage to read Proxmox resource usage safely, create a read-only API token.

Recommended permission:

Role: PVEAuditor
Path: /
Propagate: Checked
Privilege Separation: Checked

Best practice:

Do not use root@pam directly.
Create a separate user such as:

api@pam

Then create a token:

api@pam!homepage

Proxmox GUI steps:

Datacenter > Permissions > Groups
Create group:

api-ro-users

Then:

Datacenter > Permissions
Add > Group Permission

```text
Path: /
Group: api-ro-users
Role: PVEAuditor
Propagate: Checked
```

Then:

Datacenter > Permissions > Users
Add user:

User name: api
Realm: Linux PAM standard authentication
Group: api-ro-users

Then:

Datacenter > Permissions > API Tokens
Add token:

User: api@pam
Token ID: homepage
Privilege Separation: Checked

Copy the token secret. It is shown only once.

Then add API token permission:

Datacenter > Permissions
Add > API Token Permission

Path: /
API Token: api@pam!homepage
Role: PVEAuditor
Propagate: Checked

---
## 13. Adding the Proxmox Widget

The Proxmox widget belongs inside services.yaml, not widgets.yaml.

Open:

```text
sudo nano /opt/homepage/app/config/services.yaml
```

Example Proxmox card:

---
- System:
    - Proxmox VE:
        icon: proxmox.png
        href: https://YOUR-PROXMOX-IP:8006
        description: Main virtualization host
        widget:
          type: proxmox
          url: https://YOUR-PROXMOX-IP:8006
          username: api@pam!homepage
          password: YOUR_TOKEN_SECRET
          node: pve
          fields:
            - vms
            - lxc
            - resources.cpu
            - resources.mem

Replace:

YOUR-PROXMOX-IP
YOUR_TOKEN_SECRET
pve

The node name should match the Proxmox node name. It is usually visible in the Proxmox left sidebar. It can also be checked on Proxmox with:

hostname

Restart:

```text
sudo systemctl restart homepage
```

Check logs:

```text
journalctl -u homepage -n 80 --no-pager
```

---
## 14. Making the Proxmox Widget Appear Near the Top

Homepage does not allow the Proxmox service widget inside widgets.yaml next to the weather/date widgets.

To make it appear near the top, put Proxmox as the first group in services.yaml.

Example:

---
- System:
    - Proxmox VE:
        icon: proxmox.png
        href: https://YOUR-PROXMOX-IP:8006
        description: Main virtualization host
        widget:
          type: proxmox
          url: https://YOUR-PROXMOX-IP:8006
          username: api@pam!homepage
          password: YOUR_TOKEN_SECRET
          node: pve
          fields:
            - vms
            - lxc
            - resources.cpu
            - resources.mem

Then make sure System is first in settings.yaml:

layout:

```text
  System:
    style: row
    columns: 1
```

  Monitoring:

```text
    style: row
    columns: 2
```

  Entertainment:

```text
    style: row
    columns: 2
```

  Infrastructure:

```text
    style: row
    columns: 2
```

  Network:

```text
    style: row
    columns: 3
```

  Links:

```text
    style: row
    columns: 2
```

---
## 15. Proxmox Storage Resources

The Homepage Proxmox widget can show:

VM count
LXC count
CPU usage
Memory usage

It does not show Proxmox storage/disk usage directly.

Supported fields used:

fields:
  - vms
  - lxc
  - resources.cpu
  - resources.mem

For storage monitoring, the better option is to use the TrueNAS widget, since the storage pool is managed inside TrueNAS.

---
## 16. Creating a TrueNAS API Key

In TrueNAS SCALE:

## 1. Open TrueNAS in the browser:

http://YOUR-TRUENAS-IP

## 2. Click the user/profile icon in the top-right.

## 3. Click:

My API Keys

Alternative path:

Credentials > Users > select user > View API Keys

## 4. Click:

Add API Key

## 5. Name it:

homepage

## 6. Pick the user account the key belongs to.

## 7. Leave Non-expiring enabled unless an expiration is wanted.

## 8. Save.

## 9. Copy the API key immediately. TrueNAS only shows it one time.

Important:

A TrueNAS API key acts like password-level access for that user and is not protected by the user's 2FA. Keep it private.

---
## 17. Adding the TrueNAS Widget

Open services.yaml:

```text
sudo nano /opt/homepage/app/config/services.yaml
```

Example TrueNAS card:

- Infrastructure:
    - TrueNAS:
        icon: truenas.png
        href: http://YOUR-TRUENAS-IP
        description: NAS storage
        widget:
          type: truenas
          url: http://YOUR-TRUENAS-IP
          key: YOUR_TRUENAS_API_KEY
          enablePools: true

Replace:

YOUR-TRUENAS-IP
YOUR_TRUENAS_API_KEY

Restart:

```text
sudo systemctl restart homepage
```

---
## 18. Example Full services.yaml Structure

This structure matches the final settings.yaml layout groups:

---
- System:
    - Proxmox VE:
        icon: proxmox.png
        href: https://YOUR-PROXMOX-IP:8006
        description: Main virtualization host
        widget:
          type: proxmox
          url: https://YOUR-PROXMOX-IP:8006
          username: api@pam!homepage
          password: YOUR_TOKEN_SECRET
          node: pve
          fields:
            - vms
            - lxc
            - resources.cpu
            - resources.mem

- Monitoring:
    - Uptime Kuma:
        icon: uptime-kuma.png
        href: http://YOUR-FEDORA-VM-IP:3001
        description: Monitoring dashboard
        ping: http://YOUR-FEDORA-VM-IP:3001

    - Homepage:

```text
        icon: homepage.png
        href: http://YOUR-FEDORA-VM-IP:3000
        description: Homelab dashboard
        ping: http://YOUR-FEDORA-VM-IP:3000
```

- Entertainment:
    - Jellyfin:
        icon: jellyfin.png
        href: http://YOUR-JELLYFIN-IP:8096
        description: Movies and shows
        ping: http://YOUR-JELLYFIN-IP:8096

    - Minecraft Server:

```text
        icon: minecraft.png
        href: http://YOUR-MC-SERVER-IP
        description: Game server
```

- Infrastructure:
    - TrueNAS:
        icon: truenas.png
        href: http://YOUR-TRUENAS-IP
        description: NAS storage
        widget:
          type: truenas
          url: http://YOUR-TRUENAS-IP
          key: YOUR_TRUENAS_API_KEY
          enablePools: true

    - Alma Management:

```text
        icon: almalinux.png
        href: http://YOUR-ALMA-IP
        description: Terraform and Ansible VM
```

- Network:
    - Technitium DNS:
        icon: technitium.png
        href: http://YOUR-TECHNITIUM-IP:5380
        description: DNS and ad blocking
        ping: http://YOUR-TECHNITIUM-IP:5380

    - Cloudflare Zero Trust:
        icon: cloudflare.png
        href: https://one.dash.cloudflare.com
        description: Tunnel and Access policies

    - Router:

```text
        icon: router.png
        href: http://YOUR-ROUTER-IP
        description: Home router
```

- Links:
    - GitHub:
        icon: github.png
        href: https://github.com
        description: Code and projects

    - Cloudflare Dashboard:
        icon: cloudflare.png
        href: https://dash.cloudflare.com
        description: DNS and domain dashboard

---
## 19. Optional Cloudflare Tunnel Setup

If exposing Homepage through the existing Ubuntu Cloudflare tunnel, add a public hostname:

homepage.zaman-labs.dev

Point it to:

http://YOUR-FEDORA-VM-IP:3000

Then update Homepage allowed hosts in the Fedora systemd service:

```text
sudo nano /etc/systemd/system/homepage.service
```

Update the Environment line:

```text
Environment=HOMEPAGE_ALLOWED_HOSTS=YOUR-FEDORA-VM-IP:3000,homepage.zaman-labs.dev,localhost:3000,127.0.0.1:3000
```

Reload and restart:

```text
sudo systemctl daemon-reload
sudo systemctl restart homepage
```

Security note:

Homepage should be protected if exposed publicly. Use Cloudflare Access policy or another authentication layer.

---
## 20. Useful Commands

Restart Homepage:

```text
sudo systemctl restart homepage
```

Start Homepage:

```text
sudo systemctl start homepage
```

Stop Homepage:

```text
sudo systemctl stop homepage
```

Check status:

```text
systemctl status homepage
```

Follow logs:

```text
journalctl -u homepage -f
```

Show recent logs:

```text
journalctl -u homepage -n 80 --no-pager
```

Update Homepage later:

```text
cd /opt/homepage/app
sudo -u homepage git pull
sudo -u homepage pnpm install
sudo -u homepage pnpm build
sudo systemctl restart homepage
```

---
## 21. Final Notes

Homepage is the dashboard/front page.
Uptime Kuma is still the monitoring tool.

The dashboard now has:

- Dark theme
- Blue hour background image
- Background blur
- Frosted card blur
- Clean layout sections
- Removed default bookmarks
- Optional bookmark icons
- Proxmox widget for CPU/RAM/VM/LXC info
- TrueNAS widget for storage/pool info
- Weather and date widgets at the top

The final settings.yaml used:

---
```text
title: Zaman Labs
theme: dark
color: slate
headerStyle: clean
statusStyle: dot
target: _blank
```

background:

```text
  image: /images/bluehour.jpg
  blur: lg
  opacity: 65
```

cardBlur: sm

hideVersion: true

layout:

```text
  System:
    style: row
    columns: 1
```

  Monitoring:

```text
    style: row
    columns: 2
```

  Entertainment:

```text
    style: row
    columns: 2
```

  Infrastructure:

```text
    style: row
    columns: 2
```

  Network:

```text
    style: row
    columns: 3
```

  Links:

```text
    style: row
    columns: 2
```
