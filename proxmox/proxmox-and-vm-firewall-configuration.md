# Proxmox and VM Firewall Configuration

> [!CAUTION]
> This procedure is designed for the current Zaman Labs single-node Proxmox VE 9.2.6 environment. Prepare all rules while firewall enforcement is disabled, enable one scope at a time, and keep a physical or out-of-band console available. Verify live service ports before applying public rules.

## Goal

Configure the Proxmox firewall for the hypervisor, VMs, and LXC containers without losing:

- LAN access to every web interface from any device on `192.168.1.0/24`
- local SSH access
- Cloudflare Tunnel connectivity
- direct Nginx reverse-proxy access
- Minecraft access
- TrueNAS SMB/NFS access
- monitoring, DNS, API, package-update, and other required outbound connectivity

The safe baseline deliberately trusts the entire LAN. More restrictive management-only rules can be introduced after this deployment is tested.

## Important backend decision

Use the established `pve-firewall` backend for this deployment.

On **pve → Firewall → Options**:

```text
Firewall: No                  # while rules are being prepared
nftables (tech preview): No   # keep the production-stable backend
```

If `nftables (tech preview)` was changed to **Yes**, set it back to **No** before continuing. Proxmox still labels its newer nftables firewall backend as a technology preview. The command used by this guide is therefore:

```bash
sudo /usr/sbin/pve-firewall status
```

If `/usr/sbin` is already in the shell PATH, this is equivalent:

```bash
sudo pve-firewall status
```

Do not create a blank inbound `ACCEPT` rule labeled “established and related.” Proxmox handles stateful return traffic internally; a blank rule in the GUI would match all inbound traffic.

## Current IPv4 inventory

| System | IP address | Role |
|---|---:|---|
| Verizon CR1000A | `192.168.1.1` | Router and default gateway |
| Proxmox VE | `192.168.1.242` | Hypervisor, hostname `pve.home.arpa` |
| Alma-MGMT | `192.168.1.252` | Management and automation VM |
| Ubuntu Gateway | `192.168.1.254` | Nginx and Cloudflare Tunnel |
| Jellyfin | `192.168.1.240` | Media server |
| Minecraft Server 1 | `192.168.1.245` | Minecraft server |
| Minecraft Server 2 | `192.168.1.207` | Minecraft server |
| Immich | `192.168.1.208` | Immich server |
| Monitoring LXC | `192.168.1.224` | Prometheus, Grafana, and exporters |
| Fedora Dashboard/Kuma | `192.168.1.226` | Homepage and Uptime Kuma |
| Technitium DNS | `192.168.1.239` | LAN DNS server |
| TrueNAS | `192.168.1.253` | Storage server |
| Hermes Agent | `192.168.1.246` | Hermes Agent VM |

Network:

```text
192.168.1.0/24
```

## Final policy

### Proxmox host and ordinary guests

```text
Input Policy:  DROP
Output Policy: ACCEPT
Allow inbound traffic from 192.168.1.0/24
```

### Ubuntu Gateway additions

```text
Allow TCP 80 from any source, only when the router forwards public HTTP
Allow TCP 443 from any source, only when the router forwards public HTTPS
```

### Minecraft additions

```text
Allow each server's actual internal listening port and protocol from any source
```

No other guest receives an Internet-wide inbound rule.

---

## Phase 1: Prepare emergency access

1. Perform the first activation while physically at home.
2. Keep the current Proxmox browser tab open.
3. Open a second Proxmox session from another LAN computer.
4. Keep an SSH session open to `192.168.1.242`.
5. Have a keyboard and monitor, IPMI, or another out-of-band console available.
6. Do not close the original browser or SSH sessions until the second device confirms access.

Verify current access:

```bash
ping 192.168.1.242
curl -kI https://192.168.1.242:8006
ssh <proxmox-user>@192.168.1.242
```

### Check IPv6 before continuing

```bash
ip -6 address show scope global
```

If globally routed IPv6 addresses are in use for server access, add the appropriate IPv6 LAN prefix to a separate IPSet before enabling an IPv4-only default-deny policy.

---

## Phase 2: Verify the firewall backend

Run:

```bash
sudo /usr/sbin/pve-firewall status
sudo systemctl status pve-firewall --no-pager
```

Compile the current configuration without activating new rules:

```bash
sudo /usr/sbin/pve-firewall compile >/tmp/pve-firewall-compile.txt
less /tmp/pve-firewall-compile.txt
```

The compile command should complete without configuration errors.

Do not use `proxmox-firewall` or `nft list ruleset` for this documented deployment because those belong to the optional nftables technology-preview backend.

---

## Phase 3: Back up the current configuration

Open **pve → Shell**, then run:

```bash
sudo -i
```

Create the backup:

```bash
BACKUP="/root/proxmox-firewall-backup-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP"

cp -a /etc/pve/firewall "$BACKUP/" 2>/dev/null || true
cp -a /etc/pve/datacenter.cfg "$BACKUP/" 2>/dev/null || true

ip address > "$BACKUP/ip-address.txt"
ip route > "$BACKUP/ip-route.txt"
qm list > "$BACKUP/qm-list.txt"
pct list > "$BACKUP/pct-list.txt"
/usr/sbin/pve-firewall compile > "$BACKUP/pve-firewall-compile-before.txt" 2>&1 || true

printf 'Backup location: %s\n' "$BACKUP"
find "$BACKUP" -maxdepth 3 -type f -ls
```

Write down the backup directory and exit the root shell:

```bash
exit
```

---

## Phase 4: Disable enforcement while preparing rules

### Datacenter

Open **Datacenter → Firewall → Options** and set or confirm:

```text
Firewall:       No
Input Policy:   DROP
Output Policy:  ACCEPT
Forward Policy: ACCEPT
ebtables:       Yes
Log rate limit: Default
```

Leave other Datacenter options unchanged.

### Proxmox node

Open **pve → Firewall → Options** and set or confirm:

```text
Firewall:                 No
nftables (tech preview):  No
SMURFS filter:            Yes
NDP:                      Yes
```

Leave connection-tracking values and log levels at their current defaults.

### Every VM and LXC

For each guest:

1. Open **guest → Firewall → Options**.
2. Keep **Firewall: No**.
3. Open the guest network-device settings.
4. Keep the network-device **Firewall** checkbox unchecked.

This prevents any old guest rule set from becoming active unexpectedly when the Datacenter master switch is enabled later.

---

## Phase 5: Create the LAN IPSet

Open **Datacenter → Firewall → IPSet**.

Create:

```text
Name: LAN
Comment: Hogwarts local IPv4 network
```

Add this entry:

```text
IP/CIDR: 192.168.1.0/24
Nomatch:  unchecked
Comment:  Household and homelab LAN
```

`Nomatch` must remain unchecked. It is only used to exclude an address from a larger IPSet.

The optional `MANAGEMENT` and `INFRASTRUCTURE` IPSets can remain, but they are not required for this safe baseline.

---

## Phase 6: Add the Datacenter rule

Open **Datacenter → Firewall** and click **Add**.

Enter exactly:

```text
Direction: in
Action:    ACCEPT
Enable:    checked
Source:    +LAN
Comment:   Allow all Hogwarts LAN traffic
```

Leave these blank:

```text
Interface
Macro
Protocol
Source port
Destination
Destination port
```

Do not add:

- a blank `ACCEPT` rule
- an “established and related” rule with no state selector
- a manual catch-all `DROP` rule

The Input Policy supplies the final drop behavior.

Keep **Datacenter → Firewall → Options → Firewall: No**.

---

## Phase 7: Add the explicit Proxmox node rule

Open **pve → Firewall → Rules** and add:

```text
Direction: in
Action:    ACCEPT
Enable:    checked
Source:    +LAN
Comment:   Allow all LAN access to Proxmox
```

Leave every other match field blank.

This broad LAN rule preserves:

- Proxmox GUI on TCP `8006`
- local SSH on TCP `22`
- browser-based guest consoles
- any current LAN-only monitoring access to the host

Keep **pve → Firewall → Options → Firewall: No** until the final activation phase.

---

## Phase 8: Add the baseline rule to every guest

For each VM and LXC:

1. Open **guest → Firewall → Options**.
2. Set or confirm:

   ```text
   Firewall:      No
   Input Policy:  DROP
   Output Policy: ACCEPT
   ```

3. Open **guest → Firewall → Rules**.
4. Add:

   ```text
   Direction: in
   Action:    ACCEPT
   Enable:    checked
   Source:    +LAN
   Comment:   Allow all Hogwarts LAN traffic
   ```

5. Leave every other match field blank.
6. Keep the guest firewall disabled.
7. Keep the guest NIC firewall checkbox unchecked.

Repeat for:

- Alma-MGMT, `192.168.1.252`
- Ubuntu Gateway, `192.168.1.254`
- Jellyfin, `192.168.1.240`
- Minecraft Server 1, `192.168.1.245`
- Minecraft Server 2, `192.168.1.207`
- Immich, `192.168.1.208`
- Monitoring LXC, `192.168.1.224`
- Fedora Dashboard/Kuma, `192.168.1.226`
- Technitium DNS, `192.168.1.239`
- TrueNAS, `192.168.1.253`
- Hermes Agent, `192.168.1.246`

This single LAN rule preserves all existing local ports without requiring a fragile port-by-port inventory during the first deployment.

---

## Phase 9: Add public rules only where required

### 9.1 Ubuntu Gateway: Nginx reverse proxy

Guest:

```text
192.168.1.254
```

First verify Nginx listeners inside the VM:

```bash
sudo ss -lntp | grep -E ':(80|443)\b'
sudo nginx -t
```

If the Verizon router directly forwards public TCP `80` and `443` to this VM, add these two rules.

#### Public HTTP

```text
Direction:  in
Action:     ACCEPT
Enable:     checked
Protocol:   TCP
Dest. port: 80
Source:     blank
Comment:    Public HTTP to Nginx
```

#### Public HTTPS

```text
Direction:  in
Action:     ACCEPT
Enable:     checked
Protocol:   TCP
Dest. port: 443
Source:     blank
Comment:    Public HTTPS to Nginx
```

A blank source means any source. That is intentional only for services directly exposed through router port forwarding.

Do not add public rules to Immich, TrueNAS, Jellyfin, or another Nginx upstream. The gateway reaches those services from `192.168.1.254`, which is already accepted by their `+LAN` rules.

If the reverse proxy is reachable only through Cloudflare Tunnel and the router does not forward `80` or `443`, omit these two Internet-wide rules.

### 9.2 Cloudflare Tunnel

Do not add an inbound Cloudflare rule.

`cloudflared` initiates outbound connections. The guest Output Policy is `ACCEPT`, so tunnel establishment, API access, DNS, updates, and return traffic remain allowed.

Do not add inbound TCP or UDP `7844`.

### 9.3 Minecraft Server 1 and Server 2

Inside each Minecraft VM, identify the actual listeners:

```bash
sudo ss -lntup
sudo ss -lntup | grep -Ei 'java|minecraft|bedrock'
```

Check the Java server port where applicable:

```bash
grep '^server-port=' /path/to/server.properties
```

For a Java server listening internally on TCP `25565`, add:

```text
Direction:  in
Action:     ACCEPT
Enable:     checked
Protocol:   TCP
Dest. port: 25565
Source:     blank
Comment:    Public Minecraft server
```

Use the internal destination port after router NAT, not necessarily the public-facing port.

Example:

```text
Public TCP 25566 → 192.168.1.207:25565
```

The Proxmox guest rule for `192.168.1.207` must allow internal TCP `25565`.

Add UDP only when `ss`, the server configuration, or a required plugin confirms an actual UDP listener.

---

## Phase 10: Validate the prepared configuration

Before enabling anything, run:

```bash
sudo /usr/sbin/pve-firewall compile >/tmp/pve-firewall-compile.txt
less /tmp/pve-firewall-compile.txt
sudo /usr/sbin/pve-firewall status
```

Resolve every syntax or reference error before continuing.

Confirm this staging state:

```text
Datacenter Firewall:       No
pve node Firewall:         No
pve nftables tech preview: No
Every guest Firewall:      No
Every guest NIC Firewall:  unchecked
```

---

## Phase 11: Enable the Datacenter master firewall

Open **Datacenter → Firewall → Options**.

Change only:

```text
Firewall: Yes
```

Keep:

```text
Input Policy:   DROP
Output Policy:  ACCEPT
Forward Policy: ACCEPT
```

Immediately check:

```bash
sudo /usr/sbin/pve-firewall status
sudo systemctl status pve-firewall --no-pager
```

The Proxmox node firewall and all guest firewalls remain disabled at this point.

---

## Phase 12: Test one low-risk guest

Start with Fedora Dashboard/Kuma:

```text
192.168.1.226
```

1. Open **Kuma/Homepage → Firewall → Options**.
2. Set **Firewall: Yes**.
3. Open **Hardware → Network Device → Edit**.
4. Check **Firewall**.
5. Save.

From another LAN computer, test:

```bash
ping 192.168.1.226
ssh <guest-user>@192.168.1.226
```

Open both Homepage and Uptime Kuma.

Inside the VM, test outbound traffic:

```bash
ping -c 3 192.168.1.1
ping -c 3 1.1.1.1
getent hosts proxmox.com
curl -I https://www.google.com
```

Review **guest → Firewall → Log** and confirm no required traffic is being dropped.

---

## Phase 13: Enable the remaining guests one at a time

Recommended order:

1. Hermes
2. Alma-MGMT
3. Monitoring LXC
4. Jellyfin
5. Immich
6. TrueNAS
7. Technitium DNS
8. Minecraft Server 1
9. Minecraft Server 2
10. Ubuntu Gateway

For each guest:

1. Confirm the `IN ACCEPT from +LAN` rule exists.
2. Confirm required public exceptions exist only where needed.
3. Set **guest → Firewall → Options → Firewall: Yes**.
4. Enable **Firewall** on the guest network device.
5. Test LAN GUI access.
6. Test local SSH.
7. Test DNS and outbound Internet access.
8. Test the actual service.
9. Review the firewall log.
10. Continue only after all tests pass.

For guests with multiple NICs, enable filtering on one interface at a time and confirm which network each interface serves.

---

## Phase 14: Service-specific validation

### Technitium DNS

From at least two LAN clients:

```bash
nslookup example.com 192.168.1.239
```

Or:

```bash
dig @192.168.1.239 example.com
```

Also open the Technitium web interface.

### TrueNAS

Test:

- the TrueNAS web interface
- an existing SMB share
- an existing NFS share, if used
- File Browser through the reverse proxy

From Jellyfin:

```bash
mount | grep -i Network
ls -la /media/Network
```

Do not remount or modify storage merely for the firewall test.

### Monitoring

Open Grafana and Prometheus. In Prometheus, check **Status → Targets** and confirm expected targets remain `UP`.

### Cloudflare Tunnel

On the Ubuntu Gateway:

```bash
sudo systemctl status cloudflared --no-pager
sudo journalctl -u cloudflared -n 100 --no-pager
```

Test every Tunnel-hosted application from cellular data rather than home Wi-Fi.

### Nginx reverse proxy

On the Ubuntu Gateway:

```bash
sudo nginx -t
sudo systemctl status nginx --no-pager
sudo nginx -T 2>/dev/null | grep -E 'server_name|listen|proxy_pass'
```

Test each upstream from the gateway with `curl`, using the protocol and port shown by `proxy_pass`. Then test public domains from cellular data.

### Minecraft

Connect to both servers from outside the home network with the Minecraft client. A generic port checker does not fully validate the game protocol or plugins.

---

## Phase 15: Enable the Proxmox node firewall last

Before this step, confirm all critical services still work.

Keep the original Proxmox browser and SSH sessions open.

Open **pve → Firewall → Options** and set:

```text
Firewall:                Yes
nftables (tech preview): No
```

From a second LAN computer, immediately test:

```bash
ping 192.168.1.242
curl -kI https://192.168.1.242:8006
ssh <proxmox-user>@192.168.1.242
```

Open:

```text
https://192.168.1.242:8006
```

Open at least one guest console through the Proxmox GUI. Do not close the original sessions until every second-device test succeeds.

---

## Emergency rollback

### One guest stopped working

1. Select the guest.
2. Open **Firewall → Options**.
3. Set **Firewall: No**.
4. Open the guest network-device settings.
5. Uncheck **Firewall**.

### Proxmox access still works in an existing session

1. Open **pve → Firewall → Options**.
2. Set **Firewall: No**.
3. If necessary, open **Datacenter → Firewall → Options**.
4. Set **Firewall: No**.

### Proxmox web access is completely lost

Use the physical or out-of-band console.

Check the current files:

```bash
ls -la /etc/pve/firewall
```

Back them up before editing:

```bash
RECOVERY="/root/firewall-emergency-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$RECOVERY"
cp -a /etc/pve/firewall "$RECOVERY/"
```

Edit the cluster configuration:

```bash
nano /etc/pve/firewall/cluster.fw
```

Under `[OPTIONS]`, change or add:

```text
enable: 0
```

Inspect the node file, normally `/etc/pve/firewall/pve.fw`:

```bash
nano /etc/pve/firewall/pve.fw
```

Under `[OPTIONS]`, change or add:

```text
enable: 0
```

Then reload the established firewall backend:

```bash
systemctl restart pve-firewall
/usr/sbin/pve-firewall status
```

Do not delete firewall files as the first recovery action.

---

## Final rule matrix

| Scope | Rule |
|---|---|
| Datacenter | `IN ACCEPT` from `+LAN` |
| Proxmox node | `IN ACCEPT` from `+LAN` |
| Every ordinary guest | `IN ACCEPT` from `+LAN` |
| Ubuntu Gateway | Baseline LAN rule, plus TCP `80`/`443` from any source only for direct router forwarding |
| Minecraft 1 | Baseline LAN rule, plus actual public game listener from any source |
| Minecraft 2 | Baseline LAN rule, plus actual public game listener from any source |
| Cloudflare Tunnel | No inbound exception; outbound remains accepted |
| Immich, TrueNAS, Jellyfin, Technitium, monitoring, dashboards, Hermes, Alma-MGMT | No Internet-wide inbound rules |

## Rules that must not be created

```text
Blank inbound ACCEPT
Fake established/related rule with every match field blank
Manual catch-all DROP when Input Policy is already DROP
Inbound Cloudflare port 7844
Public Immich application port
Public TrueNAS management, SMB, or NFS ports
Public Technitium recursive DNS
Public Proxmox GUI or SSH port forwards
```

## Post-deployment tightening

After the baseline has operated successfully, SSH can be narrowed to Alma-MGMT (`192.168.1.252/32`) and selected administrator devices. Do that as a separate change with its own testing and rollback plan.

## References

- Proxmox VE Firewall documentation: <https://pve.proxmox.com/pve-docs/chapter-pve-firewall.html>
- `pve-firewall(8)` command reference: <https://pve.proxmox.com/pve-docs/pve-firewall.8.html>
- Proxmox cluster filesystem layout: <https://pve.proxmox.com/pve-docs/chapter-pmxcfs.html>
- Cloudflare Tunnel firewall requirements: <https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/configure-tunnels/tunnel-with-firewall/>
