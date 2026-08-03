<!--
Organized from: STEP 14 File Browser TrueNAS Nginx Remote Access.txt
Source wording and technical details were preserved as closely as possible.
-->

> [!CAUTION]
> This is a historical implementation record. Verify versions, addresses, paths, and commands against the live environment before applying changes.

# STEP 14 - File Browser TrueNAS Nginx Remote Access

Corrected File Browser setup for TrueNAS.

This setup uses:

files.zaman-labs.dev

Backend path:

Public browser -> HTTPS -> Nginx -> HTTPS -> TrueNAS File Browser app

No Caddy.
No VPN.
No Cloudflare Tunnel for File Browser.
No extra Nginx password prompt.

### Cloudflare Tunnel Note

Cloudflare Tunnel is fine for lightweight dashboards and admin pages, but it is not ideal for a NAS-style file manager because large uploads and downloads can hit Cloudflare limits. For File Browser, direct HTTPS through Nginx is the better setup.

### Final Architecture

Internet
   |
   v
Cloudflare DNS-only record
files.zaman-labs.dev -> Verizon public IPv4
   |
   v
Verizon CR1000A
TCP 80/443 forwarded to ubuntu-cloudflare
   |
   v
ubuntu-cloudflare VM
192.168.1.254
Nginx + Certbot
   |
   v
HTTPS backend
TrueNAS File Browser app
192.168.1.253:30051
   |
   v
TrueNAS datasets
/mnt/SeagateNAS/Files
/mnt/SeagateNAS/FileBrowserConfig


## PART 1 - CHOOSE WHAT FILE BROWSER CAN ACCESS

Do not expose the entire pool.

Use a dedicated dataset:

```text
/mnt/SeagateNAS/Files
```

Use a separate dataset for File Browser config:

```text
/mnt/SeagateNAS/FileBrowserConfig
```

Avoid exposing:

```text
/mnt/SeagateNAS
/mnt/SeagateNAS/Immich
/mnt/SeagateNAS/ProxmoxBackups
/mnt/SeagateNAS/.system
```

File Browser is a real file manager. Anything exposed to it can potentially be uploaded to, deleted, renamed, or modified through the web UI.


## PART 2 - CREATE THE TRUENAS DATASETS

In TrueNAS, go to:

Datasets -> SeagateNAS -> Add Dataset

Create:

Files

Then create:

FileBrowserConfig

Final paths:

```text
/mnt/SeagateNAS/Files
/mnt/SeagateNAS/FileBrowserConfig
```

Recommended dataset settings:

Dataset preset:
Generic

ACL type:
NFSv4 or POSIX

Atime:
Off

Compression:
Inherit/default

If an existing dataset is the one you want to browse, use that instead of creating /mnt/SeagateNAS/Files.


## PART 3 - GIVE THE APP PERMISSION

The TrueNAS File Browser app runs as UID/GID 568 by default.

For the config dataset:

Go to:

Datasets -> SeagateNAS/FileBrowserConfig -> Permissions -> Edit

Grant access to:

User ID:
568

Give it:

Full Control

If the ACL screen works better with groups, grant:

Group ID:
568

Access:
Full Control

Apply recursively only if this dataset is new and empty.

For the files dataset:

Go to:

Datasets -> SeagateNAS/Files -> Permissions -> Edit

Grant access to:

User ID:
568

Give it:

Modify

If Modify is not available, use:

Full Control

Apply recursively only if you are comfortable changing permissions on everything already inside that dataset.


## PART 4 - INSTALL THE FILE BROWSER APP

In TrueNAS:

Apps -> Discover Apps

Search:

File Browser

Select the File Browser Community app.

Click:

Install


## PART 5 - CONFIGURE THE APP

Application name:

filebrowser

User and group:

User ID:
568

Group ID:
568

Do not run it as root.

Network settings:

WebUI Port:
30051

Host Network:
Off

Bind mode:
Published

Host IP:
Default/all, or 192.168.1.253 if available

Do not forward port 30051 on the Verizon router.

App config storage:

Type:
Host Path

Host Path:

```text
/mnt/SeagateNAS/FileBrowserConfig
```

This corresponds to the app's internal /config location, where File Browser stores its config and database.

Additional storage:

Add additional storage for the files you want to browse:

Type:
Host Path

Host Path:

```text
/mnt/SeagateNAS/Files
```

Mount Path:

```text
/data/files
```

Read Only:
Off

Use /data/files for this TrueNAS app. The TrueNAS File Browser app sets its root to /data, and additional storage should live under that root.

Click:

Install

Wait until the app status becomes:

Running


## PART 6 - TEST FILE BROWSER LOCALLY

From a browser on your LAN, open:

https://192.168.1.253:30051

A certificate warning is expected here because the backend certificate is not valid for the raw IP address.

For command-line testing from ubuntu-cloudflare, use:

```text
curl -kL https://192.168.1.253:30051 | head -40
```

Do not rely on this as the main backend test:

```text
curl -kI https://192.168.1.253:30051
```

-I sends a HEAD request, and the app may return 404 even when the browser UI works normally.

Once the browser shows the File Browser login page, the backend is good.


## PART 7 - SECURE THE FILE BROWSER ACCOUNT

In the File Browser web UI:

1. Log in with the initial admin account.
2. Change the admin password immediately.
3. Create your normal daily-use account.
4. Use a long, unique password.
5. Confirm uploads work.
6. Confirm downloads work.
7. Confirm rename/delete behavior only works where expected.

Create one test file through the web UI, then verify it exists on TrueNAS:

```text
ls -l /mnt/SeagateNAS/Files
```


## PART 8 - CONFIGURE NGINX ON UBUNTU-CLOUDFLARE

Your Nginx VM is:

ubuntu-cloudflare
192.168.1.254

Confirm Nginx can reach File Browser:

Run on ubuntu-cloudflare:

```text
curl -kL https://192.168.1.253:30051 | head -40
```

If you see HTML or login-page content, continue.

Back up Nginx:

```text
sudo cp -a /etc/nginx "/etc/nginx.backup-$(date +%F-%H%M%S)"
```

Install required packages:

```text
sudo apt update
sudo apt install -y nginx certbot python3-certbot-nginx
```

Enable Nginx:

```text
sudo systemctl enable --now nginx
```

Create the Nginx site:

```text
sudo nano /etc/nginx/sites-available/files
```

Paste this:

server {
    listen 80;
    listen [::]:80;

    server_name files.zaman-labs.dev;

    client_max_body_size 50000M;

    proxy_request_buffering off;
    client_body_buffer_size 1024k;

    proxy_http_version 1.1;
    proxy_redirect off;

    proxy_read_timeout 600s;
    proxy_send_timeout 600s;
    send_timeout 600s;

    location / {
        proxy_pass https://192.168.1.253:30051;

        # TrueNAS File Browser is HTTPS-only on this port.
        # Its backend certificate does not match 192.168.1.253,
        # so Nginx should not verify that internal certificate.
        proxy_ssl_verify off;
        proxy_ssl_server_name on;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}

Enable the site:

```text
sudo ln -sfn /etc/nginx/sites-available/files /etc/nginx/sites-enabled/files
```

Validate:

```text
sudo nginx -t
```

Expected:

syntax is ok
test is successful

Reload:

```text
sudo systemctl reload nginx
```

Test hostname routing:

curl -L \
  -H 'Host: files.zaman-labs.dev' \
  http://127.0.0.1 | head -40

You should see File Browser content, not the default Nginx page and not Immich.


## PART 9 - CONFIGURE CLOUDFLARE DNS

If files.zaman-labs.dev currently exists as a Cloudflare Tunnel public hostname, remove that tunnel route first. The same hostname should not be used by both the tunnel and this direct DNS-only setup.

Find your public IPv4:

curl -4 https://api.ipify.org
echo

In Cloudflare:

zaman-labs.dev -> DNS -> Records -> Add record

Create:

Type:
A

Name:
files

IPv4 address:
Your Verizon public IPv4

Proxy status:
DNS only

TTL:
Auto

The cloud icon must be gray, not orange.

Use DNS-only here because proxying file uploads through Cloudflare can hit plan-based upload limits.

Verify DNS:

```text
dig +short files.zaman-labs.dev @1.1.1.1
```

Compare it with:

curl -4 https://api.ipify.org
echo

They should match.


## PART 10 - VERIZON CR1000A PORT FORWARDING

You do not need a new port forward if the Immich rules already exist.

The router should forward:

```text
TCP 80  -> 192.168.1.254:80
TCP 443 -> 192.168.1.254:443
```

That same pair supports both:

immich.zaman-labs.dev
files.zaman-labs.dev

Nginx decides where to send traffic based on the hostname.

Do not forward:

30051  File Browser backend
2049   NFS
445    SMB
8006   Proxmox
22     SSH

Do not use DMZ host.


## PART 11 - TEST PUBLIC HTTP

On your phone:

1. Turn off Wi-Fi.
2. Turn off any VPN.
3. Use cellular data.
4. Open:

http://files.zaman-labs.dev

On ubuntu-cloudflare, watch logs:

```text
sudo tail -f /var/log/nginx/access.log /var/log/nginx/error.log
```

A new access-log entry means traffic is reaching Nginx.

Do not run Certbot until public HTTP reaches Nginx.


## PART 12 - ENABLE HTTPS WITH CERTBOT

Run on ubuntu-cloudflare:

```text
sudo certbot --nginx --redirect -d files.zaman-labs.dev
```

Then validate:

```text
sudo nginx -t
sudo systemctl reload nginx
```

Check listening ports:

```text
sudo ss -lntup | grep -E ':(80|443)\b'
```

Test HTTP redirect:

curl -I \
  -H 'Host: files.zaman-labs.dev' \
  http://127.0.0.1

Expected:

HTTP/1.1 301 Moved Permanently
Location: https://files.zaman-labs.dev/

Test the certificate being served for the correct hostname:

echo | openssl s_client \
  -connect 127.0.0.1:443 \
  -servername files.zaman-labs.dev \
  2>/dev/null | openssl x509 -noout -subject -issuer -ext subjectAltName

You should see files.zaman-labs.dev in the certificate subject alternative names.

Test HTTPS through Nginx:

curl -kL \
  --resolve files.zaman-labs.dev:443:127.0.0.1 \
  https://files.zaman-labs.dev/ | head -40

Then test from cellular data:

https://files.zaman-labs.dev

You should get a valid public certificate for files.zaman-labs.dev.


## PART 13 - FIX WRONG-CERTIFICATE BEHAVIOR

If the browser says the certificate is only valid for:

immich.zaman-labs.dev

then Nginx is falling back to the wrong HTTPS server block.

Run:

```text
sudo nginx -T 2>/dev/null | grep -A35 -B5 'server_name files.zaman-labs.dev'
```

Confirm there is a server_name files.zaman-labs.dev block.

Then issue or reissue the certificate:

```text
sudo certbot --nginx --redirect -d files.zaman-labs.dev
```

Reload Nginx:

```text
sudo nginx -t
sudo systemctl reload nginx
```

Verify the certificate:

echo | openssl s_client \
  -connect 127.0.0.1:443 \
  -servername files.zaman-labs.dev \
  2>/dev/null | openssl x509 -noout -subject -issuer -ext subjectAltName

Only continue when files.zaman-labs.dev appears in the certificate SAN list.


## PART 14 - SNAPSHOTS

In TrueNAS:

Data Protection -> Periodic Snapshot Tasks -> Add

Create a snapshot task for:

SeagateNAS/Files

Recommended:

Recursive:
On

Schedule:
Every 6 hours

Lifetime:
30 days

Snapshots are not a full backup because they remain on the same TrueNAS pool, but they are useful for recovering from accidental deletion or bad edits.


## PART 15 - UPDATES

Update the File Browser app from:

TrueNAS -> Apps -> Installed Applications -> filebrowser

Before updating:

1. Confirm snapshots exist.
2. Confirm you can log in.
3. Update the app.
4. Test login.
5. Test upload/download.

Update Nginx and Certbot on ubuntu-cloudflare:

```text
sudo apt update
sudo apt upgrade -y
```

Test certificate renewal:

```text
sudo certbot renew --dry-run
```


## TROUBLESHOOTING

### Local IP shows certificate warning

This is normal:

https://192.168.1.253:30051

The internal certificate is not issued for the raw IP.

For command-line testing, use:

```text
curl -kL https://192.168.1.253:30051 | head -40
```


### curl -kI returns 404

Use GET instead:

```text
curl -kL https://192.168.1.253:30051 | head -40
```

-I sends a HEAD request, which may not behave like a normal browser page load.


### Nginx returns 502

From ubuntu-cloudflare:

```text
curl -kL https://192.168.1.253:30051 | head -40
```

If that fails, check the TrueNAS app logs:

TrueNAS -> Apps -> Installed Applications -> filebrowser -> Logs


### Nginx shows the default page

Test with the hostname:

curl -L \
  -H 'Host: files.zaman-labs.dev' \
  http://127.0.0.1 | head -40

Check enabled configs:

```text
ls -l /etc/nginx/sites-enabled/
sudo nginx -T | grep -A25 'server_name files.zaman-labs.dev'
```


### Browser shows Immich certificate

Reissue the files certificate:

```text
sudo certbot --nginx --redirect -d files.zaman-labs.dev
sudo nginx -t
sudo systemctl reload nginx
```

Then verify:

echo | openssl s_client \
  -connect 127.0.0.1:443 \
  -servername files.zaman-labs.dev \
  2>/dev/null | openssl x509 -noout -subject -issuer -ext subjectAltName


### Public access times out

Check DNS:

dig +short files.zaman-labs.dev @1.1.1.1
curl -4 https://api.ipify.org
echo

They should match.

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


### Large uploads fail

Make sure the Cloudflare files record is DNS only, not proxied.
The cloud icon should be gray.


## FINAL CHECKLIST

[ ] File Browser app is installed on TrueNAS
[ ] File Browser uses port 30051
[ ] File Browser backend is HTTPS-only
[ ] /mnt/SeagateNAS/Files exists
[ ] /mnt/SeagateNAS/FileBrowserConfig exists
[ ] UID/GID 568 has access to both datasets
[ ] File Browser works locally at https://192.168.1.253:30051
[ ] Nginx uses server_name files.zaman-labs.dev
[ ] Nginx uses proxy_pass https://192.168.1.253:30051
[ ] Nginx includes proxy_ssl_verify off
[ ] Cloudflare record is files.zaman-labs.dev
[ ] Cloudflare record is DNS only
[ ] CR1000A forwards TCP 80 and 443 to 192.168.1.254
[ ] Port 30051 is not forwarded publicly
[ ] Certbot certificate is issued for files.zaman-labs.dev
[ ] Browser no longer shows the Immich certificate
[ ] HTTPS works at https://files.zaman-labs.dev
[ ] File Browser admin password is changed
[ ] TrueNAS snapshots are enabled for SeagateNAS/Files
