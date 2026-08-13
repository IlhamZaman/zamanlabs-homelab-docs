<!--
Organized from: STEP 10 Homepage Dashboard Setup(2).txt
Source wording and technical details were preserved as closely as possible.
Updated 2026-08-12 after resolving Homepage 1.13.1 / TrueNAS SCALE 25.10.4 API authentication.
-->

> [!CAUTION]
> This is a historical implementation record. Verify versions, addresses, paths, and commands against the live environment before applying changes.

## STEP 10 Homepage Dashboard Setup

This step covers setting up Homepage on the existing Fedora Uptime Kuma VM, customizing the dashboard style, adding service cards, removing default bookmark sections, adding icons, and adding Proxmox/TrueNAS widgets. It also documents the current secure TrueNAS API configuration using HTTPS and a protected systemd environment file.

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

Create a protected directory for Homepage secrets:

```text
sudo install -d -m 700 -o root -g root /etc/homepage
```

Create the Homepage environment file:

```text
sudo install -m 600 -o root -g root /dev/null /etc/homepage/homepage.env
```

Verify the permissions:

```text
sudo stat -c '%A %a %U:%G %n' /etc/homepage/homepage.env
```

Expected:

```text
-rw------- 600 root:root /etc/homepage/homepage.env
```

Create the systemd service:

```text
sudo nano /etc/systemd/system/homepage.service
```

Example service file:

```ini
[Unit]
Description=Homepage Dashboard
After=network.target

[Service]
Type=simple
User=homepage
Group=homepage
WorkingDirectory=/opt/homepage/app
Environment=HOMEPAGE_ALLOWED_HOSTS=YOUR-FEDORA-VM-IP:3000,localhost:3000,127.0.0.1:3000
EnvironmentFile=/etc/homepage/homepage.env
ExecStart=/usr/local/bin/pnpm start
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Replace `YOUR-FEDORA-VM-IP` with the real IP address of the Fedora VM.

The `EnvironmentFile` line allows Homepage secrets to be kept outside the normal YAML configuration. The environment file is root-owned and mode `0600` so it is not readable by normal local users.

Save the file:

```text
CTRL + O
ENTER
CTRL + X
```

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
/etc/homepage/homepage.env
```

What each file does:

```text
settings.yaml              = Theme, background, blur, card style, layout columns
services.yaml              = Main service cards and service widgets
widgets.yaml               = Top widgets like date, weather, search, local resources
bookmarks.yaml             = Bookmark sections like Developer, Social, Entertainment
/etc/homepage/homepage.env = Protected Homepage environment secrets
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

The current TrueNAS integration uses the Homepage TrueNAS widget with:

```yaml
version: 2
```

For TrueNAS 25.04 and later, Homepage uses the newer WebSocket API when `version: 2` is configured.

### Create the key in TrueNAS SCALE

1. Open TrueNAS over HTTPS:

```text
https://YOUR-TRUENAS-IP
```

2. Open the user/profile menu in the top-right.
3. Select:

```text
My API Keys
```

Alternative path:

```text
Credentials > Users > select user > View API Keys
```

4. Click **Add API Key**.
5. Give the key a descriptive name such as:

```text
homepage
```

6. Select the user account the key belongs to.
7. Leave **Non-expiring** enabled unless an expiration is specifically wanted.
8. Save.
9. Copy the generated API key immediately. TrueNAS does not allow the same key string to be viewed again after the dialog is closed.

> [!IMPORTANT]
> A TrueNAS user-linked API key provides password-equivalent API access as its associated user and is not protected by that user's 2FA configuration. Treat the key as a secret.

> [!WARNING]
> TrueNAS 25.10 requires HTTPS for API-key authentication. If a user-linked API key is submitted during an authentication attempt over insecure HTTP, TrueNAS revokes it. A revoked key must be reset, which generates a new key string.

Do not configure the Homepage TrueNAS widget with:

```yaml
url: http://YOUR-TRUENAS-IP
```

Use HTTPS:

```yaml
url: https://YOUR-TRUENAS-IP
```

---
## 17. Storing the TrueNAS API Key Securely

Do not store the live TrueNAS API key directly in:

```text
/opt/homepage/app/config/services.yaml
```

The Homepage configuration file can be readable by users who should not have access to the API key. The current setup stores the key in the protected environment file created earlier:

```text
/etc/homepage/homepage.env
```

Stop Homepage before rotating or replacing the TrueNAS key:

```text
sudo systemctl stop homepage.service
```

Open the environment file:

```text
sudo nano /etc/homepage/homepage.env
```

Add the key:

```ini
HOMEPAGE_VAR_TRUENAS_KEY=REDACTED
```

Do not put the real key in this repository.

Verify the environment file permissions:

```text
sudo stat -c '%A %a %U:%G %n' /etc/homepage/homepage.env
```

Expected:

```text
-rw------- 600 root:root /etc/homepage/homepage.env
```

The systemd unit must contain:

```ini
EnvironmentFile=/etc/homepage/homepage.env
```

Reload systemd after changing the service unit:

```text
sudo systemctl daemon-reload
```

Homepage supports environment-secret substitution for variables beginning with `HOMEPAGE_VAR_`. The TrueNAS widget references the environment variable instead of containing the API key itself.

---
## 18. Adding the TrueNAS Widget

Open `services.yaml`:

```text
sudo nano /opt/homepage/app/config/services.yaml
```

Example TrueNAS card:

```yaml
- Infrastructure:
    - TrueNAS:
        icon: truenas.png
        href: https://YOUR-TRUENAS-IP
        description: NAS storage
        widget:
          type: truenas
          url: https://YOUR-TRUENAS-IP
          version: 2
          key: "{{HOMEPAGE_VAR_TRUENAS_KEY}}"
          enablePools: true
```

Replace:

```text
YOUR-TRUENAS-IP
```

Do **not** replace `{{HOMEPAGE_VAR_TRUENAS_KEY}}` with the literal API key. Homepage replaces that reference with the value loaded from the protected environment file.

Start or restart Homepage:

```text
sudo systemctl restart homepage.service
```

Check the service:

```text
systemctl status homepage.service --no-pager
```

---
## 19. TrueNAS Widget Authentication Troubleshooting

### Resolved 2026-08-12

Affected systems during the incident:

```text
Homepage: 1.13.1 on kuma-homepage / 192.168.1.226
TrueNAS SCALE: 25.10.4 on 192.168.1.253
```

The TrueNAS service card showed a green status indicator, but the widget could not authenticate.

Homepage logs showed:

```text
TrueNAS API key [REDACTED] failed
Websocket call for TrueNAS failed: TrueNAS authentication failed
```

The green service status did not prove that widget authentication was working. It only showed that the separate service/ping check could reach TrueNAS.

### Root cause 1: HTTP was used for API-key authentication

The widget originally used:

```yaml
url: http://192.168.1.253
```

while also using a user-linked API key.

For TrueNAS 25.10, API-key authentication must use HTTPS. An API key submitted over HTTP can be revoked automatically.

The widget URL was corrected to:

```yaml
url: https://192.168.1.253
```

and the v2 widget configuration was retained:

```yaml
version: 2
```

### Root cause 2: the API key was stored directly in `services.yaml`

The key was originally stored directly in:

```text
/opt/homepage/app/config/services.yaml
```

The file was mode `0644`, so the key was treated as exposed and replaced.

The replacement key was moved to:

```text
/etc/homepage/homepage.env
```

with:

```text
root:root
0600
```

### Intermediate failure: Homepage loaded the secret but the widget did not reference it

A read-only verification showed:

```text
HTTPS: correct
version: 2: correct
HOMEPAGE_VAR_TRUENAS_KEY: loaded and non-empty
homepage.service: active
```

but the TrueNAS widget was still missing:

```yaml
key: "{{HOMEPAGE_VAR_TRUENAS_KEY}}"
```

Without that `key:` line, Homepage had the secret in its process environment but the TrueNAS widget was not told to use it.

After adding the line, the configuration structure was correct.

### Final failure: TrueNAS still rejected the old/revoked key

After HTTPS, `version: 2`, the environment file, and the `key:` reference were all correct, Homepage still logged:

```text
Websocket call for TrueNAS failed: TrueNAS authentication failed
```

The TrueNAS API key itself was then reset/replaced and the new value was stored in:

```text
/etc/homepage/homepage.env
```

Homepage was restarted.

The final read-only check reported:

```text
PASS — fixed.
homepage.service: active
No TrueNAS widget errors in the last two minutes
No recent authentication failures detected
```

### Key-length troubleshooting note

A diagnostic check of the value loaded into the running Homepage process reported:

```text
Loaded key length: 66
```

The widget was nevertheless authenticating successfully and the logs were clean.

Do not trim, rewrite, or reset a working API key solely because a diagnostic character count differs from an expected value. Successful authentication and clean current logs are the operational validation that matters for this setup.

### Safe validation commands

Check Homepage:

```text
systemctl is-active homepage.service
```

Check recent logs:

```text
sudo journalctl -u homepage.service --since "2 minutes ago" --no-pager
```

Filter for likely TrueNAS/authentication errors:

```text
sudo journalctl -u homepage.service --since "2 minutes ago" --no-pager \
  | grep -iE 'truenas|websocket|auth|error'
```

Test HTTPS connectivity to TrueNAS:

```text
curl -vkI https://YOUR-TRUENAS-IP/
```

`-k` is only for this manual connectivity test when the TrueNAS certificate is self-signed or otherwise not trusted by the local certificate store. Do not switch the Homepage widget back to HTTP to work around a certificate problem.

Verify the protected environment file without displaying the secret:

```text
sudo stat -c '%A %a %U:%G %n' /etc/homepage/homepage.env
```

Check whether the running Homepage process has the variable without printing the key:

```text
pid=$(systemctl show -p MainPID --value homepage.service)

sudo tr '\0' '
' < /proc/$pid/environ \
  | awk '/^HOMEPAGE_VAR_TRUENAS_KEY=/{sub(/^[^=]*=/,""); print "Loaded key length:", length($0)}'
```

Do not paste the API key into shell output, logs, screenshots, GitHub issues, or this repository.

---
## 20. Example Full services.yaml Structure

This structure matches the final `settings.yaml` layout groups and uses the secure TrueNAS API configuration:

```yaml
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
        icon: homepage.png
        href: http://YOUR-FEDORA-VM-IP:3000
        description: Homelab dashboard
        ping: http://YOUR-FEDORA-VM-IP:3000

- Entertainment:
    - Jellyfin:
        icon: jellyfin.png
        href: http://YOUR-JELLYFIN-IP:8096
        description: Movies and shows
        ping: http://YOUR-JELLYFIN-IP:8096

    - Minecraft Server:
        icon: minecraft.png
        href: http://YOUR-MC-SERVER-IP
        description: Game server

- Infrastructure:
    - TrueNAS:
        icon: truenas.png
        href: https://YOUR-TRUENAS-IP
        description: NAS storage
        widget:
          type: truenas
          url: https://YOUR-TRUENAS-IP
          version: 2
          key: "{{HOMEPAGE_VAR_TRUENAS_KEY}}"
          enablePools: true

    - Alma Management:
        icon: almalinux.png
        href: http://YOUR-ALMA-IP
        description: Terraform and Ansible VM

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
        icon: router.png
        href: http://YOUR-ROUTER-IP
        description: Home router

- Links:
    - GitHub:
        icon: github.png
        href: https://github.com
        description: Code and projects

    - Cloudflare Dashboard:
        icon: cloudflare.png
        href: https://dash.cloudflare.com
        description: DNS and domain dashboard
```

> [!CAUTION]
> `YOUR_TOKEN_SECRET` is only a placeholder in this documentation. Never commit a real Proxmox token secret or TrueNAS API key to the repository.

---
## 21. Optional Cloudflare Tunnel Setup

If exposing Homepage through the existing Ubuntu Cloudflare tunnel, add a public hostname:

```text
homepage.zaman-labs.dev
```

Point it to:

```text
http://YOUR-FEDORA-VM-IP:3000
```

Then update Homepage allowed hosts in the Fedora systemd service:

```text
sudo nano /etc/systemd/system/homepage.service
```

Update the `Environment` line:

```ini
Environment=HOMEPAGE_ALLOWED_HOSTS=YOUR-FEDORA-VM-IP:3000,homepage.zaman-labs.dev,localhost:3000,127.0.0.1:3000
```

Keep the protected environment file line:

```ini
EnvironmentFile=/etc/homepage/homepage.env
```

Reload and restart:

```text
sudo systemctl daemon-reload
sudo systemctl restart homepage.service
```

Security note:

Homepage should be protected if exposed publicly. Use Cloudflare Access policy or another authentication layer.

---
## 22. Useful Commands

Restart Homepage:

```text
sudo systemctl restart homepage.service
```

Start Homepage:

```text
sudo systemctl start homepage.service
```

Stop Homepage:

```text
sudo systemctl stop homepage.service
```

Check status:

```text
systemctl status homepage.service --no-pager
```

Check whether Homepage is active:

```text
systemctl is-active homepage.service
```

Follow logs:

```text
journalctl -u homepage.service -f
```

Show recent logs:

```text
journalctl -u homepage.service -n 80 --no-pager
```

Check recent TrueNAS/authentication-related messages:

```text
sudo journalctl -u homepage.service --since "5 minutes ago" --no-pager \
  | grep -iE 'truenas|websocket|auth|error'
```

Verify the environment file permissions:

```text
sudo stat -c '%A %a %U:%G %n' /etc/homepage/homepage.env
```

Show the effective systemd unit:

```text
sudo systemctl cat homepage.service
```

Update Homepage later:

```text
cd /opt/homepage/app
sudo -u homepage git pull
sudo -u homepage pnpm install
sudo -u homepage pnpm build
sudo systemctl restart homepage.service
```

---
## 23. Final Notes

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
- TrueNAS v2 widget for storage/pool information
- TrueNAS API authentication over HTTPS
- TrueNAS API secret stored outside `services.yaml`
- Protected `/etc/homepage/homepage.env` with mode `0600`
- Weather and date widgets at the top
- Optional Cloudflare Tunnel access

The current TrueNAS widget security pattern is:

```text
services.yaml
    |
    | key: "{{HOMEPAGE_VAR_TRUENAS_KEY}}"
    v
homepage.service
    |
    | EnvironmentFile=/etc/homepage/homepage.env
    v
HOMEPAGE_VAR_TRUENAS_KEY
    |
    v
Homepage TrueNAS v2 Widget
    |
    | HTTPS
    v
TrueNAS SCALE
```

The final `settings.yaml` used:

```yaml
title: Zaman Labs
theme: dark
color: slate
headerStyle: clean
statusStyle: dot
target: _blank

background:
  image: /images/bluehour.jpg
  blur: lg
  opacity: 65

cardBlur: sm
hideVersion: true

layout:
  System:
    style: row
    columns: 1

  Monitoring:
    style: row
    columns: 2

  Entertainment:
    style: row
    columns: 2

  Infrastructure:
    style: row
    columns: 2

  Network:
    style: row
    columns: 3

  Links:
    style: row
    columns: 2
```

### Current resolved TrueNAS state

```text
Homepage service:            ACTIVE
TrueNAS widget transport:    HTTPS
TrueNAS widget version:      2
Environment secret loading:  WORKING
TrueNAS authentication:      WORKING
Recent widget errors:        NONE
API key in services.yaml:    NO
Secret file permissions:     0600 root:root
```

**Result: RESOLVED**
