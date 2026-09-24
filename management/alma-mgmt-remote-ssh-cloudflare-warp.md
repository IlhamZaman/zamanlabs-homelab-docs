<!--
Historical implementation record based on the verified September 19-20, 2026 Alma-MGMT remote-access deployment.
Sensitive credentials and API tokens are intentionally omitted.
-->

> [!CAUTION]
> This is a historical implementation record. Verify Cloudflare settings, addresses, routes, client profiles, and service state against the live environment before applying changes.

# Alma-MGMT Remote SSH Through Cloudflare WARP

### Goal

I wanted secure remote SSH access to the AlmaLinux management VM without exposing TCP/22 to the public Internet, adding a router port-forward, or requiring a client-side `cloudflared` SSH proxy.

Alma-MGMT is the central management VM for the homelab:

```text
Alma-MGMT
IP: 192.168.1.252
SSH: TCP/22
User: izaman
```

The final design uses Cloudflare Zero Trust and WARP to route only Alma-MGMT's private IP through a dedicated Cloudflare Tunnel.

Harry assisted with read-only audits, Cloudflare state checks, rollback planning, and validation during the implementation. I reviewed and approved the architecture and each configuration change before it was applied.

## Final Architecture

```text
Authorized remote device
        |
        | Cloudflare One Client / WARP
        v
Cloudflare Zero Trust
        |
        | Gateway network policy
        v
Private route: 192.168.1.252/32
        |
        v
Dedicated Alma-MGMT-SSH Tunnel
        |
        v
Alma-MGMT
192.168.1.252:22
        |
        v
OpenSSH public-key authentication
```

The working path does **not** use:

- public TCP/22 exposure
- router SSH port-forwarding
- `cloudflared access ssh`
- SSH `ProxyCommand`
- browser-based SSH
- a public DNS record for Alma-MGMT
- the existing Ubuntu Gateway tunnel

The existing Ubuntu Gateway and its application routes were kept separate from this management path.

## Why I Used the Private IP

My first approach attempted to use a private hostname:

```text
alma.zaman-labs.dev
```

The Cloudflare private-hostname route existed and the WARP client was receiving DNS through Cloudflare, but the hostname continued returning `NXDOMAIN`.

I did not publish a public DNS record as a workaround.

Instead, I changed the design to route Alma-MGMT's private address directly:

```text
192.168.1.252/32
```

This removed DNS synthesis from the SSH path and made the routing behavior easier to verify.

## Baseline Before Changes

Before changing the Cloudflare configuration, I verified the existing Alma-MGMT state.

The checks confirmed:

- Alma-MGMT was using `192.168.1.252/24`
- `sshd.service` was active
- SSH was listening on TCP/22
- existing SSH key authentication worked internally
- `firewalld` was active
- there was no router SSH port-forward
- the dedicated Alma-MGMT Cloudflare connector was healthy
- no existing Cloudflare private network route conflicted with `192.168.1.252/32`

I also preserved the existing Ubuntu Gateway Cloudflare configuration and treated it as outside the scope of this change.

## Dedicated Cloudflare Tunnel

I used a dedicated Cloudflare Tunnel for Alma-MGMT rather than adding SSH routing to the Ubuntu Gateway tunnel.

```text
Tunnel: Alma-MGMT-SSH
Connector host: Alma-MGMT
```

The tunnel connector runs on Alma-MGMT and establishes an outbound connection to Cloudflare.

Because the connection is outbound, no inbound SSH port needs to be opened on the home router.

## Cloudflare Private Network Route

I added one private network route:

```text
192.168.1.252/32 -> Alma-MGMT-SSH
```

The `/32` is intentional.

I did not route:

```text
192.168.1.0/24
```

or:

```text
192.168.0.0/16
```

through the tunnel.

Only the Alma-MGMT address is part of this Cloudflare private route.

After creating the route, I verified:

- the CIDR was exactly `192.168.1.252/32`
- it pointed to the dedicated Alma-MGMT tunnel
- the route was active
- no duplicate or overlapping private route had been introduced

## Cloudflare Gateway Policy

The private route alone is not the authorization control.

I also protected SSH traffic with Cloudflare Gateway network policies.

The protected traffic is:

```text
Destination IP: 192.168.1.252
Destination port: 22
Protocol: TCP
```

Policy order:

```text
1. Authorized identity -> ALLOW
2. Everyone else      -> BLOCK
```

This means being able to reach Cloudflare is not enough by itself. The enrolled client still has to match the authorized identity policy.

Alma-MGMT continues to require its normal OpenSSH key authentication after Cloudflare permits the network connection.

## WARP Split Tunnel Problem

The Cloudflare One device profile was already using:

```text
Split Tunnel mode: Exclude
```

The existing profile excluded the entire RFC1918 block:

```text
192.168.0.0/16
```

That created a problem.

If `192.168.0.0/16` remained excluded, traffic for Alma-MGMT would bypass WARP and use the client's normal local route instead of entering the Cloudflare private network.

I did not want to remove the exclusion completely because that would pull the rest of the `192.168.0.0/16` space into WARP.

The goal was:

```text
192.168.0.0/16
MINUS
192.168.1.252/32
```

## Split Tunnel Carveout

I replaced the single broad `192.168.0.0/16` exclusion with the following non-overlapping CIDRs:

```text
192.168.128.0/17
192.168.64.0/18
192.168.32.0/19
192.168.16.0/20
192.168.8.0/21
192.168.4.0/22
192.168.2.0/23
192.168.0.0/24
192.168.1.0/25
192.168.1.128/26
192.168.1.192/27
192.168.1.224/28
192.168.1.240/29
192.168.1.248/30
192.168.1.254/31
192.168.1.253/32
```

Together, these exclusions cover every address that was previously covered by `192.168.0.0/16` except:

```text
192.168.1.252
```

The resulting behavior is:

```text
192.168.1.252 -> WARP
Everything else in 192.168.0.0/16 -> normal local routing
```

I verified the CIDRs were pairwise non-overlapping and represented 65,535 addresses, leaving exactly one address from the original 65,536-address `/16` outside the exclusion list.

That one address is Alma-MGMT.

## Client Routing Validation

After the updated device profile propagated, I refreshed the WARP connection on an enrolled Linux client.

WARP reported connected and healthy.

The route to Alma-MGMT was:

```text
192.168.1.252 dev CloudflareWARP table 65743 src 100.96.0.3
```

I also checked an unrelated LAN address.

Example:

```text
192.168.1.1 dev eno1 src 192.168.1.x
```

This confirmed the intended behavior:

- Alma-MGMT was entering WARP
- unrelated LAN traffic was still using the normal local interface

This client-side route verification was important because the existence of a Cloudflare route object by itself does not prove that the endpoint is actually sending traffic through WARP.

## SSH Validation

After confirming routing, I tested TCP/22 to:

```text
192.168.1.252
```

The connection succeeded through WARP.

I then used normal OpenSSH with the existing client identity key:

```text
ssh -i "/path/to/existing/private-key" izaman@192.168.1.252
```

No SSH authentication settings had to be weakened or replaced.

I did **not** change:

- `/etc/ssh/sshd_config`
- `authorized_keys`
- SSH host keys
- SSH authentication methods
- Alma-MGMT `firewalld` rules

The Cloudflare layer controls whether the remote client can reach SSH. OpenSSH still performs the actual host and user authentication.

## Android / Termius

I also validated the setup from an Android device using Termius while connected through a different external network.

Connection settings:

```text
Host: 192.168.1.252
Port: 22
User: izaman
Authentication: existing SSH private key
```

The device must first be enrolled in the same Cloudflare Zero Trust organization and show WARP as connected.

## Remote Client Requirements

A remote device must meet all of these conditions:

1. Be enrolled in the correct Cloudflare Zero Trust organization.
2. Receive the intended Cloudflare One device profile.
3. Show WARP as connected.
4. Match the authorized Cloudflare Gateway identity policy.
5. Have the correct SSH private key.
6. Connect directly to `192.168.1.252`.

Example:

```text
ssh -i "/path/to/existing/private-key" izaman@192.168.1.252
```

A new client should also verify Alma-MGMT's existing SSH host-key fingerprint before accepting a new trust entry.

## What I Intentionally Did Not Use

### Public SSH

There is no public TCP/22 listener created by this design.

No router WAN port-forward is required.

### Public DNS

There is no public `A`, `AAAA`, or `CNAME` record required for Alma-MGMT remote SSH.

### Client-Side cloudflared

The remote client does not need:

```text
cloudflared access ssh
```

and does not need an SSH `ProxyCommand`.

The client only needs the Cloudflare One/WARP client and the existing SSH key.

### Ubuntu Gateway Tunnel

The existing Ubuntu Gateway Cloudflare Tunnel was not modified for this project.

The Alma-MGMT SSH path remains isolated in its own dedicated tunnel and private route.

## Gateway Analytics Validation

After testing the remote connection, I checked Cloudflare Gateway Layer 4 analytics.

The observed SSH sessions matched the authorized-identity `ALLOW` policy.

The following `BLOCK` policy remained immediately after the allow rule for clients that do not match the authorized identity.

This is the correct place to verify the Cloudflare identity decision.

The source IP visible to Alma-MGMT's `sshd` is not, by itself, proof of the original remote user's Cloudflare identity.

## Failed Hostname Route Cleanup

After the private-IP design was working, I removed the obsolete private-hostname route for:

```text
alma.zaman-labs.dev
```

I preserved:

- the `192.168.1.252/32` private network route
- the dedicated Alma-MGMT tunnel
- the active connector
- the Gateway allow/block policies
- the WARP split-tunnel configuration

I did not create a public DNS replacement for the failed hostname.

## Recovery

The main rollback boundary is intentionally narrow.

If the `/32` route or split-tunnel change ever causes unrelated LAN traffic to enter WARP, the targeted recovery procedure is:

1. Remove only the Alma-MGMT `192.168.1.252/32` private route.
2. Restore the previous `192.168.0.0/16` WARP exclusion.
3. Refresh the client device profile.
4. Verify normal LAN routing.
5. Stop and investigate before making broader changes.

If only SSH fails while unrelated routing remains correct, I would first check:

- dedicated tunnel and connector health
- exact `/32` private route
- WARP client status
- client route selection
- Gateway policy decisions
- TCP/22 reachability
- Alma-MGMT `sshd` state

I would not weaken the Gateway policies or SSH authentication simply because a connection test failed.

## Final State

The verified remote-management path is:

```text
Enrolled WARP client
        |
        v
Cloudflare Gateway identity policy
        |
        v
192.168.1.252/32 private route
        |
        v
Alma-MGMT-SSH tunnel
        |
        v
Alma-MGMT TCP/22
        |
        v
Existing SSH key authentication
```

This gives me remote SSH access to the central management VM without exposing SSH publicly and without routing the rest of the home LAN through Cloudflare.

The Alma-MGMT management model remains unchanged after login: Alma acts as the central SSH, Ansible, and Terraform control system for the rest of the homelab.
