<!--
Organized from: STEP 4 cloudflare_gateway_vm_setup_notes(2).txt
Source wording and technical details were preserved as closely as possible.
-->

> [!CAUTION]
> This is a historical implementation record. Verify versions, addresses, paths, and commands against the live environment before applying changes.

# Cloudflare Tunnel Gateway VM Setup Notes

### Context

Cousin asked:
"Where did you put Cloudflare Tunnel?"

He also said:
"U should do one gateway VM. Everything gets routed through it."

Ubuntu Server gateway VM IP:

```text
192.168.1.254
```


### What He Means

He is asking which machine or VM is running cloudflared.

The recommended setup is to avoid installing Cloudflare Tunnel on every VM. Instead, create one Ubuntu Server VM that acts as the gateway/reverse-proxy VM.

That gateway VM runs:

- cloudflared
- optionally Nginx
- firewall rules
- routing/reverse proxy rules

Cloudflare sends traffic to that one VM, and that VM forwards the traffic internally to other services like:

- Proxmox
- TrueNAS
- other VMs
- web apps
- admin panels


### Network Layout

Internet
   |
Cloudflare
   |
Cloudflare Tunnel
   |
Ubuntu Gateway VM
192.168.1.254
   |
Internal LAN services
Proxmox: 192.168.1.x:8006
TrueNAS: 192.168.1.x
Other VMs: 192.168.1.x


Example:

proxmox.yourdomain.com

goes through Cloudflare Tunnel to:

Ubuntu Gateway VM
192.168.1.254

Then the gateway VM forwards it internally to:

https://192.168.1.242:8006


### Main Goal

Use one clean gateway VM instead of running a tunnel on every machine.

This makes the setup:

- easier to manage
- more secure
- cleaner
- easier to firewall
- better for Cloudflare Access


## Ubuntu Gateway VM Setup

### Assumptions

Gateway VM IP:

## 192.168.1.254

Example internal service IPs:

Proxmox:

```text
192.168.1.242:8006
```

TrueNAS:

```text
192.168.1.250:443
```

Change these IPs if your actual Proxmox or TrueNAS IPs are different.


### Step 1: SSH Into the Ubuntu Gateway VM

```text
ssh youruser@192.168.1.254
```


### Step 2: Update Ubuntu

```text
sudo apt update
sudo apt upgrade -y
sudo apt install curl gpg nginx ufw -y
```


### Step 3: Confirm or Set Static IP

Check the current IP:

```text
ip a
```

Check Netplan files:

```text
ls /etc/netplan/
```

Edit the config:

```text
sudo nano /etc/netplan/00-installer-config.yaml
```

Example Netplan config:

network:
  version: 2
  ethernets:
    ens18:
      dhcp4: no
      addresses:
        - 192.168.1.254/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8

Apply it:

```text
sudo netplan apply
```

Test internet:

```text
ping 1.1.1.1
ping google.com
```


## Error Encountered: Netplan Permissions Too Open

Warning shown:

** (configure:3547): WARNING **: Permissions for /etc/netplan/00-installer-config.yaml are too open.
Netplan configuration should NOT be accessible by others.

** (process:3545): WARNING **: Permissions for /etc/netplan/00-installer-config.yaml are too open.
Netplan configuration should NOT be accessible by others.

### Meaning

The Netplan YAML file permissions were too loose.

Ubuntu was warning that normal users could potentially read the network configuration file.

This was not a broken-network issue. It was just a permissions warning.


### Fix

Run:

```text
sudo chmod 600 /etc/netplan/00-installer-config.yaml
```

Then check:

```text
ls -l /etc/netplan/00-installer-config.yaml
```

Expected output should look like:

-rw------- 1 root root ...

Then re-apply Netplan:

```text
sudo netplan apply
```

Optional safer full fix:

```text
sudo chmod 600 /etc/netplan/*.yaml
sudo chown root:root /etc/netplan/*.yaml
sudo netplan apply
```

Meaning:

Only root can read and write the Netplan network configuration.


## Step 4: Install cloudflared

Run:

```text
sudo mkdir -p --mode=0755 /usr/share/keyrings
```

```text
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
```

```text
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main" | sudo tee /etc/apt/sources.list.d/cloudflared.list
```

```text
sudo apt update
sudo apt install cloudflared -y
```

Check version:

cloudflared --version


## Step 5: Create the Cloudflare Tunnel

Go to Cloudflare Dashboard:

```text
Cloudflare Dashboard
→ Zero Trust
→ Networks
→ Tunnels
→ Create a tunnel
→ Cloudflared
```

Name the tunnel something like:

ubuntu-gateway

Cloudflare will give an install command that looks similar to:

```text
sudo cloudflared service install eyJhIjoi...
```

Copy that command and run it on the Ubuntu Gateway VM.

Then check the service:

```text
sudo systemctl status cloudflared
```

Enable it on boot:

```text
sudo systemctl enable cloudflared
```


## Step 6: Add Public Hostnames in Cloudflare

In the tunnel settings:

Tunnel
→ Routes
→ Add route
→ Published application

Example route for Proxmox:

Subdomain:
proxmox

Domain:
yourdomain.com

Service:

```text
https://192.168.1.242:8006
```

Result:

proxmox.yourdomain.com → https://192.168.1.242:8006


Example route for TrueNAS:

Subdomain:
truenas

Domain:
yourdomain.com

Service:

```text
https://192.168.1.250
```

Result:

truenas.yourdomain.com → https://192.168.1.250


Example test route for the gateway VM:

Subdomain:
gateway

Domain:
yourdomain.com

Service:

```text
http://localhost:80
```


## Step 7: Test Nginx Locally

On the Ubuntu Gateway VM, run:

```text
curl localhost
```

You should see the default Nginx page.

Then visit:

https://gateway.yourdomain.com

If that loads, the Cloudflare Tunnel is working.


## Step 8: Secure the Ubuntu Gateway Firewall

Set default firewall rules:

```text
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

Allow SSH only from the LAN:

```text
sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp
```

Allow HTTP for local Nginx/Cloudflare Tunnel routing:

```text
sudo ufw allow 80/tcp
```

Enable UFW:

```text
sudo ufw enable
```

Check status:

```text
sudo ufw status verbose
```

Important note:

Since Cloudflare Tunnel uses outbound connections, you do not need to port forward ports like:

- 80
- 443
- 8006
- TrueNAS ports
- Proxmox ports

on your router.


## Step 9: Lock Down Proxmox to Only Allow the Gateway VM

Once the tunnel and Cloudflare Access work properly, Proxmox can be restricted so only the gateway VM can reach the web UI.

Gateway VM:

## 192.168.1.254

Proxmox firewall idea:

ALLOW TCP from 192.168.1.254 to port 8006
DROP TCP to port 8006 from everything else

Do not do this until you confirm the tunnel and Cloudflare Access are working, or you could lock yourself out.


## Step 10: Add Cloudflare Access

For important admin panels like Proxmox and TrueNAS, do not just expose the login page through Cloudflare Tunnel.

Add Cloudflare Access in front of them.

Go to:

```text
Cloudflare Zero Trust
→ Access
→ Applications
→ Add application
→ Self-hosted
```

Example for Proxmox:

Application name:
Proxmox

Domain:
proxmox.yourdomain.com

Policy:

Allow only your email.
Require identity login or OTP.

This makes users authenticate with Cloudflare before they even see the Proxmox login page.


## Optional: Use Nginx as a Reverse Proxy

There are two approaches.


### Option 1: Easier Way

Let Cloudflare Tunnel point directly to internal services:

```text
proxmox.yourdomain.com → https://192.168.1.242:8006
truenas.yourdomain.com → https://192.168.1.250
```

This is simpler and fine for most home lab setups.


### Option 2: Cleaner Advanced Way

Cloudflare Tunnel points to local Nginx on the gateway VM:

```text
proxmox.yourdomain.com → http://localhost:8081
truenas.yourdomain.com → http://localhost:8082
```

Then Nginx forwards internally.


### Example Proxmox Nginx Config

Create file:

```text
sudo nano /etc/nginx/sites-available/proxmox
```

Paste:

server {
    listen 8081;
    server_name proxmox.yourdomain.com;

    location / {
        proxy_pass https://192.168.1.242:8006;
        proxy_ssl_verify off;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

Enable config:

```text
sudo ln -s /etc/nginx/sites-available/proxmox /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

Then in Cloudflare Tunnel route:

proxmox.yourdomain.com → http://localhost:8081


## Recommended Setup

Use:

Ubuntu Gateway VM
IP: 192.168.1.254

Runs:

- cloudflared
- Nginx optional
- UFW firewall

Cloudflare routes:

```text
proxmox.yourdomain.com → https://192.168.1.242:8006
truenas.yourdomain.com → https://your-truenas-ip
```

Protect both with:

Cloudflare Access

Later, tighten internal firewalls so Proxmox and TrueNAS only accept admin web traffic from:

## 192.168.1.254


## Final Notes

The phrase "where did you put Cloudflare Tunnel?" means:

Which VM or machine is running cloudflared?

The phrase "one gateway VM, everything gets routed through it" means:

Use one Ubuntu Server VM as the central entry point for Cloudflare Tunnel and reverse proxy routing.

The best setup is:

Cloudflare Tunnel runs only on Ubuntu Server at 192.168.1.254.
All public hostnames point to that tunnel.
The gateway forwards traffic to internal LAN services.
Cloudflare Access protects sensitive apps before their login pages are visible.
