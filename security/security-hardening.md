# Internet-Facing Services Hardening and Recovery Baseline

**Environment:** Zaman Labs Homelab  
**Primary hardening window:** September 2026  
**Status:** Implemented and validated  
**Scope:** Ubuntu Gateway, Nginx, Cloudflare Tunnel, File Browser, Jellyfin, Immich, TrueNAS, Proxmox, and recovery metadata

> This document records the conservative hardening work applied after an Internet-facing security audit. The goal was to reduce practical attack surface without changing the intended remote-access architecture or introducing unnecessary service risk.
>
> This repository is public. **Do not commit API tokens, `.env` files, private keys, certificate archives, database dumps, TrueNAS configuration exports, raw rollback files, or other secret-bearing backup artifacts.**

---

## 1. Design Goals

The hardening plan intentionally prioritized low-risk, reversible changes.

The following architectural decisions were preserved:

- File Browser remains publicly accessible through the Ubuntu Gateway's Nginx reverse proxy.
- File Browser is **not** placed behind Tailscale, WireGuard, or Cloudflare Access.
- Jellyfin remains publicly accessible through Nginx and is **not** proxied through Cloudflare.
- Normal Jellyfin streaming uses a non-administrator account.
- The TrueNAS administrative web interface remains behind Cloudflare Tunnel.
- TrueNAS SMB/NFS services remain LAN-only.
- Public Nginx services use Cloudflare for DNS only.
- Cloudflare Tunnel is used only for the intentionally configured administrative/internal web services.
- No TrueNAS upgrade was performed during this hardening round.
- No broad container-hardening changes were applied blindly.

The implementation rule throughout the project was:

> **Audit first, back up the current state, make one narrowly scoped change, validate, then continue.**

---

## 2. Initial Security Audit

The initial read-only audit found no evidence of active compromise.

No malicious persistence, successful credential attack, planted PHP payload, suspicious outbound session, or unauthorized SSH activity was confirmed.

The main practical hardening gaps were:

- File Browser was publicly reachable and requires careful boundary protection.
- Nginx lacked application-specific rate limiting.
- Nginx logging did not contain enough incident-response context.
- Nginx logs retained only about two weeks of history.
- The Ubuntu Gateway host firewall allowed SSH and Node Exporter more broadly than necessary.
- Immich's `.env` file was readable more broadly than required.
- Several recovery paths depended on TrueNAS, creating a shared failure domain.
- A stale Cloudflare Tunnel configuration contained routes that no longer matched the intended architecture.

The hardening plan was deliberately narrowed to address these issues without redesigning the homelab.

---

# 3. Implemented Hardening

## 3.1 Immich `.env` Permissions

### Previous state

```text
/opt/immich/.env
owner: izaman
group: docker
mode: 0664
```

The file contains database and deployment secrets, so world-readable access was unnecessary.

### Change

The file was reduced to:

```text
owner: izaman
group: docker
mode: 0600
```

Ownership was not changed.

### Validation

- Deployment account confirmed as `izaman`.
- `docker compose config --quiet` passed.
- `immich_server` remained healthy.
- `immich_machine_learning` remained healthy.
- PostgreSQL remained healthy.
- Redis remained healthy.
- Internal Immich ping succeeded.
- No container was restarted or recreated.

### Result

**PASS**

---

## 3.2 File Browser Nginx Boundary Hardening

File Browser intentionally remains:

```text
Internet
  -> Cloudflare DNS
  -> public Nginx reverse proxy
  -> TrueNAS File Browser
```

No VPN or Cloudflare Access layer was added.

### Sensitive-path protection

The File Browser Nginx virtual host now rejects narrowly selected requests that should never be served accidentally, including:

- `.env`
- `.env.*`
- `.git` paths
- Vim swap files
- trailing `~` editor artifacts
- narrowly named configuration/database backup and dump files

These requests are rejected directly by Nginx and do not reach the File Browser backend.

### Deliberately not blocked

No blanket extension deny was added for:

```text
.php
.cgi
.pl
.sh
```

File Browser is a file-serving application, and legitimate files may use those extensions.

The active Nginx configuration has no PHP/FastCGI execution path, so such files are not executed by Nginx.

### Unknown-host rejection

Default Nginx handling was hardened so unconfigured hostnames are rejected instead of receiving the generic Nginx page.

Behavior:

- unknown HTTP Host -> rejected with Nginx `444`
- unknown HTTPS SNI -> rejected during TLS negotiation
- configured domains continue to route normally

### Validation

- `nginx -t`: PASS
- graceful reload: PASS
- File Browser: healthy
- Jellyfin: healthy
- Immich: healthy
- HTTP -> HTTPS redirects: healthy
- TLS certificates: valid
- Certbot timer: active

### Result

**PASS**

---

## 3.3 Nginx Incident Logging

The original access log remains in place for compatibility.

A second structured incident-response log was added in parallel.

### Incident log fields

The JSON-format incident log records:

```text
timestamp
request_id
client_ip
host
server_name
method
uri
status
bytes_sent
referrer
user_agent
upstream_addr
upstream_status
request_time
upstream_response_time
```

### Privacy safeguards

The incident log does **not** record:

- request bodies
- authorization headers
- cookies
- passwords
- API tokens
- query arguments

The logged URI excludes query strings.

Referrers containing query strings are suppressed.

### Retention

Nginx log rotation was extended from approximately 14 daily rotations to:

```text
daily
rotate 30
compress
delaycompress
notifempty
```

The original access log continues to receive entries.

### Validation

- JSON parsing tested successfully.
- No parse failures observed.
- Existing tooling remained unaffected.
- Disk-space impact was negligible.
- File Browser, Jellyfin, Immich, Nginx, Certbot, and cloudflared remained healthy.

### Result

**PASS**

---

## 3.4 Authentication Endpoint Rate Limiting

No global request limiter is used.

Only verified authentication endpoints are protected.

### File Browser

```text
POST /api/login
rate: 6 requests/minute/client IP
burst: 12
nodelay
excess response: 429
```

### Jellyfin

```text
POST /Users/AuthenticateByName
POST /Users/AuthenticateWithQuickConnect
rate: 12 requests/minute/client IP
burst: 20
nodelay
excess response: 429
```

Jellyfin Quick Connect polling remains unrestricted.

Streaming, playback, WebSockets, sessions, and ordinary APIs remain outside the limiter.

### Immich

```text
POST /api/auth/login
rate: 10 requests/minute/client IP
burst: 20
nodelay
excess response: 429
```

Immich uploads, asset delivery, synchronization, thumbnails, and media routes remain unrestricted.

### Design rule

The limiter exists to slow sustained password spraying and brute-force behavior without affecting normal application usage.

There is deliberately:

- no global limiter
- no generic upload limiter
- no Jellyfin streaming limiter
- no Immich sync limiter
- no File Browser transfer limiter
- no connection limiter
- no CrowdSec/Fail2ban dependency

### Validation

- `nginx -t`: PASS
- graceful reload: PASS
- normal service traffic unaffected
- no unexpected `429` responses observed during validation

### Result

**PASS**

---

## 3.5 Cloudflare Tunnel Cleanup

The Ubuntu Gateway also hosts the remotely managed Cloudflare Tunnel connector.

The intended design separates:

### DNS-only public Nginx services

Examples include:

- File Browser
- Jellyfin
- Immich

These resolve using Cloudflare DNS but traffic reaches the public Nginx reverse proxy directly.

### Cloudflare Tunnel services

The Tunnel remains responsible for the intentionally configured management/internal web applications such as:

- Proxmox
- TrueNAS GUI
- Uptime Kuma
- Homepage
- Grafana
- Prometheus

### Cleanup

Two stale Tunnel ingress entries were removed:

- an obsolete/nonexistent Ubuntu hostname
- the File Browser Tunnel route, because File Browser intentionally uses the DNS-only Nginx path

The File Browser DNS record was **not** changed.

The legitimate Tunnel routes, catch-all `http_status:404`, and WARP routing were preserved.

### Cloudflare API access

A dedicated least-privilege API token was created for Tunnel management.

The token:

- is scoped to Cloudflare Tunnel management only
- does not have DNS write permission
- is stored outside Git
- is protected with mode `0600`
- is never printed into documentation, logs, or automation output

### Validation

- exactly one Tunnel configuration update was performed
- live configuration was fetched immediately before the write
- a sanitized rollback snapshot was retained privately
- all remaining Tunnel routes were verified afterward
- cloudflared remained healthy with its HA connections
- File Browser remained available through Nginx
- DNS was not modified

### Result

**PASS**

---

## 3.6 Ubuntu Gateway Firewall Narrowing

The Gateway performs both Nginx reverse proxy and Cloudflare Tunnel duties.

The firewall change therefore used strict:

```text
ADD -> VERIFY -> REMOVE -> VERIFY
```

sequencing.

### SSH

Previous rules allowed SSH from the LAN and/or broadly.

The final rule limits SSH to the management host:

```text
Alma-MGMT -> Ubuntu Gateway TCP/22
```

A new SSH session from Alma-MGMT was tested both before and after broad-rule removal.

### Node Exporter

Node Exporter is now reachable only from the Monitoring LXC:

```text
Monitoring LXC -> Ubuntu Gateway TCP/9100
```

Prometheus scraping was verified before and after removing the broad rule.

### UDP 9100

The previous unqualified firewall rule also permitted UDP.

UDP access was removed because:

- Node Exporter has no UDP listener
- no UDP 9100 traffic was observed
- no legitimate dependency was identified

### Public web traffic

The intended public Nginx listeners remain open:

```text
TCP/80
TCP/443
```

### Default policy

```text
incoming: DENY
outgoing: ALLOW
routed: disabled
```

The outgoing policy was deliberately preserved so Cloudflare Tunnel connectivity was not affected.

### Validation

- fresh SSH from Alma-MGMT: PASS
- Prometheus target: UP
- Node Exporter `/metrics`: HTTP 200
- cloudflared: healthy
- Nginx: healthy
- File Browser: healthy
- Jellyfin: healthy
- Immich: healthy
- legitimate Cloudflare Tunnel applications: healthy

### Result

**PASS**

---

# 4. Architecture Decisions Preserved

## Jellyfin

Jellyfin remains a standard Nginx reverse-proxied service.

It is not moved behind Cloudflare Tunnel.

Normal streaming uses a dedicated non-administrator user account rather than the Jellyfin administrative account.

No unnecessary authentication or network architecture changes were introduced.

## File Browser

File Browser remains reverse proxied through Nginx.

It was not moved behind:

- Tailscale
- WireGuard
- Cloudflare Access

Security is enforced through:

- File Browser authentication
- Nginx sensitive-path protection
- authentication rate limiting
- logging
- normal HTTPS
- backend/LAN boundaries

## TrueNAS

The TrueNAS GUI remains behind Cloudflare Tunnel.

SMB and NFS remain private LAN services.

No manual `runc` replacement or unsupported appliance modification was performed.

A TrueNAS platform update was intentionally deferred.

---

# 5. Recovery and Backup Improvements

The original audit found that the main Proxmox backup repository depended on TrueNAS itself.

That design still provides useful protection against individual VM failure, but it is not independent protection against complete TrueNAS/storage loss.

A set of small, high-value recovery artifacts was therefore copied to Alma-MGMT local storage.

> **Important:** Alma-MGMT is a VM on the same physical Proxmox host. These copies are independent of TrueNAS, but they are not yet independent of the entire Proxmox machine.

---

## 5.1 TrueNAS Configuration

A native TrueNAS configuration export was created using the supported middleware configuration-save mechanism.

The recovery copy includes the password secret seed required to restore encrypted configuration fields.

It does not include ZFS pool keys.

The artifact is secret-bearing and is stored privately outside Git.

Status: **Protected independently of TrueNAS**

---

## 5.2 Proxmox Host Configuration

A reconstruction archive was created containing recovery-relevant configuration including:

- `/etc/pve` guest definitions
- storage configuration
- datacenter configuration
- firewall configuration
- user/ACL metadata
- backup-job definitions
- network configuration
- host reconstruction configuration
- IOMMU/passthrough configuration
- TrueNAS HBA mapping

Especially sensitive material such as cluster private keys, SSH private keys, TLS private keys, password hashes, and secret token material was intentionally excluded.

Status: **Protected independently of TrueNAS**

---

## 5.3 Immich

The Immich recovery set contains:

- native PostgreSQL custom-format dump
- `docker-compose.yml`
- protected `.env` recovery copy
- non-secret recovery manifest

The PostgreSQL archive passed `pg_restore --list`.

Immich remained online during the backup.

The TrueNAS-hosted photo/video library was **not** copied.

Status: **Application state protected; media still depends on TrueNAS**

---

## 5.4 Ubuntu Gateway

The Gateway recovery set contains:

- complete `/etc/nginx`
- Nginx logrotate configuration
- complete `/etc/letsencrypt`

The Certbot archive contains private keys and ACME account credentials and must never be committed to Git.

Status: **Nginx/Certbot state protected independently of TrueNAS**

---

## 5.5 File Browser

The File Browser recovery set contains:

- `filebrowser.db`
- `settings.json`
- recovery metadata

File Browser uses BoltDB/bbolt rather than SQLite.

A stable byte-copy method was used while the service remained online, and the copy passed bbolt/application-level validation using a disposable duplicate.

The SMB user-data tree was deliberately not traversed or copied.

Status: **Application state protected; user files still depend on TrueNAS**

---

## 5.6 Jellyfin

The Jellyfin recovery set contains:

- SQLite database created using SQLite's online backup API
- persistent configuration
- plugin state
- application metadata
- user/account state

The database passed `PRAGMA integrity_check`.

The TrueNAS-hosted media library was not traversed or copied.

Status: **Application state protected; media still depends on TrueNAS**

---

# 6. Secret-Bearing Recovery Artifacts

The recovery artifacts created during this project are operational backups, **not repository content**.

Never commit any of the following:

```text
Cloudflare API tokens
TrueNAS configuration exports
Immich .env recovery copies
Immich database dumps
Certbot /etc/letsencrypt archives
File Browser databases
Jellyfin databases/application archives
private SSH keys
API credentials
password hashes
raw firewall rollback records containing sensitive metadata
```

Documentation should contain only sanitized descriptions, procedures, and validation results.

---

# 7. Deliberately Deferred Changes

The following were intentionally **not** performed during this hardening round because they introduce greater compatibility or lockout risk and were not necessary to achieve the current security objective:

- TrueNAS upgrade solely for the bundled container runtime
- manual replacement of TrueNAS appliance packages
- Immich rootless conversion
- arbitrary Docker capability dropping
- blanket `read_only` container filesystems
- aggressive Content-Security-Policy deployment
- broad file-extension blocking in File Browser
- changing File Browser upstream TLS verification
- Cloudflare Access in front of File Browser
- moving Jellyfin behind Cloudflare
- TrueNAS MFA changes
- broad CrowdSec deployment
- Fail2ban for LAN-only SSH
- global Nginx request limiting
- broad scanner/404 blocking without evidence-based tuning

These may be revisited individually if future evidence justifies them.

---

# 8. Ongoing Monitoring

A weekly security investigation is already scheduled for:

```text
Monday 10:00 AM
```

The weekly review should pay particular attention to:

- unexpected `429` responses
- repeated login failures
- exploit/scanner request patterns
- abnormal HTTP `4xx`/`5xx` bursts
- suspicious requests in the structured incident log
- Nginx errors
- File Browser health
- Jellyfin health
- Immich health
- Cloudflare Tunnel health
- Prometheus Gateway target health
- unexpected SSH access
- unexpected Node Exporter exposure
- new persistence or outbound-connection indicators

The current policy is to prefer **evidence-driven remediation** over continuously adding new security controls.

---

# 9. Current Security Baseline

At the end of this hardening round:

| Control | State |
|---|---|
| Immich `.env` permission hardening | Complete |
| File Browser sensitive-path protection | Complete |
| Unknown-host Nginx rejection | Complete |
| Structured incident logging | Complete |
| 30-day Nginx log retention | Complete |
| Authentication endpoint rate limiting | Complete |
| Gateway SSH source restriction | Complete |
| Gateway Node Exporter source restriction | Complete |
| Unused UDP/9100 exposure | Removed |
| Cloudflare Tunnel stale-route cleanup | Complete |
| File Browser Nginx architecture | Preserved |
| Jellyfin Nginx architecture | Preserved |
| TrueNAS GUI Cloudflare Tunnel architecture | Preserved |
| TrueNAS configuration recovery copy | Complete |
| Proxmox configuration recovery copy | Complete |
| Immich application recovery set | Complete |
| Gateway Nginx/Certbot recovery set | Complete |
| File Browser application recovery set | Complete |
| Jellyfin application recovery set | Complete |
| TrueNAS-independent bulk-data backup | Not yet implemented |
| Physically independent/off-site recovery tier | Not yet implemented |
| Isolated restoration drill | Not yet performed |

---

# 10. Remaining Recovery Limitations

The hardening round is complete, but several disaster-recovery limitations remain intentionally outside its scope.

### Same-host dependency

The independent application/configuration recovery sets currently live on Alma-MGMT, which is hosted on the same physical Proxmox machine.

They protect against loss of TrueNAS, but not against complete physical loss of the Proxmox host.

### Bulk data

The following bulk datasets still require a physically independent backup strategy:

- Immich photos/videos
- File Browser/SMB user data
- Jellyfin media where the content cannot be easily reacquired

### Full VM backups

The existing VM/LXC backup repository remains dependent on TrueNAS.

A second physically independent target can be added later without replacing the existing working backup job.

### Restore testing

Backup presence and structural validation do not prove complete recovery.

An isolated restore drill should eventually be performed under a separate, controlled change window.

---

# 11. Operating Principles Going Forward

1. Keep public exposure limited to intentionally published services.
2. Keep SSH management centralized through Alma-MGMT.
3. Keep monitoring ports source-restricted.
4. Keep administrative interfaces private or behind their intended Cloudflare Tunnel/Access boundary.
5. Apply rate limits only where endpoint behavior is understood.
6. Never apply one generic security policy blindly across Jellyfin, Immich, and File Browser.
7. Prefer add-before-remove firewall changes.
8. Back up current configuration before high-impact changes.
9. Validate new controls using real service behavior.
10. Keep secret-bearing backup artifacts outside Git.
11. Treat weekly security scans and incident logs as the evidence source for future hardening.
12. Avoid security changes solely for the sake of adding more controls.

---

## Final State

The September 2026 hardening round materially reduced the attack surface of Zaman Labs without changing its intended remote-access model or causing service loss.

The environment now has:

- narrower management exposure
- application-specific authentication throttling
- better incident evidence
- safer Nginx boundary behavior
- cleaned Cloudflare Tunnel routing
- tighter local secret permissions
- independent configuration/application recovery artifacts

No compromise was identified during the original investigation, and every implemented hardening change was validated with application-health and rollback checks.

Further changes should be driven by weekly security-review evidence or by a separately planned disaster-recovery project rather than by broad hardening for its own sake.
