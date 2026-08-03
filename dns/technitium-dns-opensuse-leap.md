<!--
Organized from: STEP 6 Technitium DNS on openSUSE Leap(2).txt
Source wording and technical details were preserved as closely as possible.
-->

> [!CAUTION]
> This is a historical implementation record. Verify versions, addresses, paths, and commands against the live environment before applying changes.

# STEP 6 - Technitium DNS on openSUSE Leap Server in Proxmox

Chosen distro: openSUSE Leap Server
Network manager: NetworkManager
VM network interface: enp6s18
Purpose: Technitium DNS adblocker DNS server for the home network

Important router note:
My router did not have a LAN DNS / DHCP DNS option available, so I set the router's WAN DNS instead. This means the router itself forwards DNS queries to the Technitium VM. Ideally, the router's LAN DHCP DNS should point clients directly to Technitium, but if that option does not exist, setting WAN DNS to the Technitium server is the available workaround.

Example network used in this guide:
Router/Gateway: 192.168.1.1
Technitium VM IP: 192.168.1.53
Subnet: /24

Replace 192.168.1.53 with the static IP you actually want to use.

---
## 1. Update openSUSE Leap

Run:

```text
sudo zypper refresh
sudo zypper update -y
sudo reboot
```

After reboot, check the network interface:

```text
ip a
```

Confirm the interface is:

enp6s18

---
### 2. Set a static IP using NetworkManager

Check the active connection name:

```text
nmcli device status
nmcli connection show --active
```

You may see something like:

Wired connection 1

Set the static IP. Replace the IP, gateway, and DNS values if your network is different:

sudo nmcli connection modify "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses 192.168.1.53/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "127.0.0.1 1.1.1.1" \
  ipv6.method disabled

Restart the connection:

```text
sudo nmcli connection down "Wired connection 1"
sudo nmcli connection up "Wired connection 1"
```

Verify the network settings:

```text
ip a show enp6s18
ip route
cat /etc/resolv.conf
ping -c 3 1.1.1.1
ping -c 3 google.com
```

The VM should now be using the static IP, for example:

## 192.168.1.53

---
### 3. Install required packages

Install basic tools:

```text
sudo zypper install -y curl wget tar gzip ca-certificates
```

Install Technitium DNS Server:

```text
curl -sSL https://download.technitium.com/dns/install.sh | sudo bash
```

Check the Technitium service:

```text
sudo systemctl status dns
```

Enable Technitium at boot:

```text
sudo systemctl enable --now dns
```

View logs if needed:

```text
journalctl -u dns -f
```

---
### 4. Make sure port 53 is available

Technitium needs DNS port 53 over both TCP and UDP.

Check if anything is already using port 53:

```text
sudo ss -tulpn | grep ':53'
```

Technitium should be the service listening on port 53.

If another service is using port 53, fix that before continuing.

---
### 5. Open firewall ports on openSUSE Leap

Check if firewalld is running:

```text
sudo systemctl status firewalld
```

Open DNS and the Technitium web panel:

```text
sudo firewall-cmd --permanent --add-port=53/udp
sudo firewall-cmd --permanent --add-port=53/tcp
sudo firewall-cmd --permanent --add-port=5380/tcp
sudo firewall-cmd --reload
```

Verify the ports:

```text
sudo firewall-cmd --list-ports
```

Expected ports:

53/udp 53/tcp 5380/tcp

---
### 6. Open the Technitium web console

From a browser on the same network, go to:

http://192.168.1.53:5380

Create the admin username and password when prompted.

Do not expose this web console to the public internet. Keep it LAN-only.

---
### 7. Configure Technitium forwarders

Inside the Technitium web UI, go to:

Settings > Proxy & Forwarders

Simple setup:

```text
Forwarder Protocol: UDP
Forwarders:
1.1.1.1
1.0.0.1
9.9.9.9
149.112.112.112
```

Better privacy setup:

Forwarder Protocol: DNS-over-HTTPS or DNS-over-TLS

Common DNS-over-HTTPS examples:

Cloudflare:

```text
https://cloudflare-dns.com/dns-query
```

Quad9:

```text
https://dns.quad9.net/dns-query
```

Save the settings after adding the forwarders.

---
### 8. Enable ad blocking

Inside Technitium, look for the blocking/adblock section.

Depending on the UI version, it may be under:

Apps / DNS Apps
or
Settings > Blocking

Enable blocking.

Start with a small number of trusted lists. Good starting options:

OISD Basic
Hagezi Multi NORMAL
StevenBlack hosts

Do not add too many lists right away. Start light, test the network, then add more if needed.

After adding blocklists:

Save the settings
Run Update Now / Refresh if available

---
### 9. Test DNS from another device

From another Linux/macOS machine:

```text
nslookup google.com 192.168.1.53
nslookup doubleclick.net 192.168.1.53
```

Or with dig:

```text
dig @192.168.1.53 google.com
dig @192.168.1.53 doubleclick.net
```

From Windows PowerShell or Command Prompt:

```text
nslookup google.com 192.168.1.53
```

For blocked domains, Technitium may return 0.0.0.0, NXDOMAIN, or another blocked response depending on the blocking settings.

---
### 10. Point the router to Technitium

Ideal method:

Router DHCP / LAN DNS settings:
Primary DNS: 192.168.1.53
Secondary DNS: blank if possible

Do not put 8.8.8.8, 1.1.1.1, or another public DNS as secondary unless bypassing Technitium sometimes is acceptable.

My actual router situation:

The router did not provide a LAN DNS or DHCP DNS option, so I used the WAN DNS setting instead.

WAN DNS:
Primary DNS: 192.168.1.53
Secondary DNS: blank if possible

This should make the router forward DNS requests to the Technitium VM.

After changing DNS settings, renew client leases or reboot devices.

Linux NetworkManager:

```text
nmcli connection show
sudo nmcli connection down "your connection name"
sudo nmcli connection up "your connection name"
```

Windows:

ipconfig /release
ipconfig /renew
ipconfig /flushdns

macOS:

```text
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

---
### 11. Make the openSUSE VM use Technitium for DNS

Once Technitium is working, point the VM to itself for DNS:

```text
sudo nmcli connection modify "Wired connection 1" ipv4.dns "127.0.0.1"
sudo nmcli connection up "Wired connection 1"
```

Test DNS:

```text
dig google.com
```

If dig is missing, install it:

```text
sudo zypper install -y bind-utils
```

Then test again:

```text
dig google.com
dig @127.0.0.1 google.com
```

---
### 12. Recommended Proxmox VM settings

For the Technitium VM:

CPU: 1 to 2 cores
RAM: 1 to 2 GB
Disk: 16 to 32 GB
Network Device: VirtIO
Bridge: vmbr0
BIOS: SeaBIOS is fine
Start at boot: Yes

Technitium is lightweight, so this is enough for a normal home network.

In Proxmox:

VM > Options > Start at boot > Yes

This makes the DNS VM start automatically when the Proxmox host boots.

---
### 13. Backup Technitium configuration

After everything works, download a config backup from the Technitium web UI.

Look for:

Settings > Backup Settings

Also consider backing up these paths from the VM:

```text
/opt/technitium/dns
/etc/dns
/var/log/technitium/dns
```

This makes it easier to restore or migrate Technitium later.

---
### 14. Troubleshooting commands

Check Technitium service:

```text
sudo systemctl status dns
```

Follow logs:

```text
journalctl -u dns -f
```

Check DNS/web ports:

```text
sudo ss -tulpn | grep -E ':53|:5380'
```

Check firewall:

```text
sudo firewall-cmd --list-all
```

Test DNS locally:

```text
dig @127.0.0.1 google.com
dig @192.168.1.53 google.com
```

Test DNS from another device:

```text
nslookup google.com 192.168.1.53
```

---
### Final setup summary

Proxmox VM:
openSUSE Leap Server
NetworkManager
Interface: enp6s18
Static IP: 192.168.1.53
Technitium Web UI: http://192.168.1.53:5380

openSUSE firewall:
53/udp open
53/tcp open
5380/tcp open

Router:
Preferred option: LAN DHCP DNS points to 192.168.1.53
Actual used option: WAN DNS points to 192.168.1.53 because LAN DNS was not available
Secondary DNS: blank if possible

Technitium:
Forwarders configured
Ad blocking enabled
Blocklists added
Service enabled at boot
