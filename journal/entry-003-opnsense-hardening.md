# Entry 003 — OPNsense Hardening & Service Configuration

**Date:** 2026-03-08
**Status:** ✅ Complete
**Phase:** Phase 1 — Foundation

---

## What Was Accomplished

- Set hostname and domain on OPNsense
- Configured Kea DHCP pools for all three active subnets
- Added static MAC reservations for all five VMs
- Changed root password from default
- Enabled SSH restricted to LAN_Admin interface only
- Verified NTP configuration with four pool servers
- Downloaded OPNsense configuration backup

---

## Hostname Configuration

```
System → Settings → General

Hostname:  opnsense
Domain:    lab.local
Full FQDN: opnsense.lab.local
DNS:       8.8.8.8 (temporary — will point to WinServ-DNS after AD setup)
```

> DNS will be updated to 192.168.10.10 (WinServ-DNS) once Active Directory
> and DNS are configured on the LAN_DMZ Windows Server.

---

## Kea DHCP Configuration

OPNsense 26.x ships with Kea DHCP as the default server.
Pool format uses dash notation: `x.x.x.x-x.x.x.x`

### Subnets Configured

| Subnet | Description | DHCP Pool |
|---|---|---|
| 192.168.10.0/29 | LAN_Admin | 192.168.10.2 – 192.168.10.6 |
| 192.168.10.8/29 | LAN_DMZ | 192.168.10.10 – 192.168.10.14 |
| 192.168.10.16/29 | LAN_Attack | 192.168.10.18 – 192.168.10.22 |

### Static MAC Reservations

| Description | MAC Address | IP Address | Subnet |
|---|---|---|---|
| WinServ-AD | 08:00:27:FC:A3:D4 | 192.168.10.2 | LAN_Admin |
| Ubuntu-Monitor | 08:00:27:2C:AD:09 | 192.168.10.3 | LAN_Admin |
| WinServ-DNS | 08:00:27:0A:38:A5 | 192.168.10.10 | LAN_DMZ |
| Kali-Attack | 08:00:27:9A:94:17 | 192.168.10.18 | LAN_Attack |
| Metasploitable2 | 08:00:27:96:BE:B1 | 192.168.10.19 | LAN_Attack |

> Static reservations guarantee each VM always receives the same IP regardless
> of DHCP lease order — critical for firewall rules and attack targeting.

---

## SSH Configuration

```
System → Settings → Administration → SSH

☑ Enable Secure Shell
☑ Permit root user login     (lab only — never in production)
☑ Permit password login      (lab only — use key pairs in production)
Listen interface: LAN_Admin  (management plane restricted to admin network)
Port: 22
```

### Why SSH is Restricted to LAN_Admin

```
LAN_Admin  → SSH reachable    ✅  (admin machines only)
LAN_DMZ    → SSH unreachable  ❌  (DMZ servers cannot manage firewall)
LAN_Attack → SSH unreachable  ❌  (attack lab cannot reach management)
WAN        → SSH unreachable  ❌  (internet cannot reach management)
```

> Production standard: always use SSH key pairs, never password login.
> AWS EC2 instances enforce key pair authentication by default.
> Management plane security = restrict admin access to dedicated admin networks only.

---

## NTP Configuration

```
Services → Network Time → General

Time servers:
  0.opnsense.pool.ntp.org  (Preferred)
  1.opnsense.pool.ntp.org
  2.opnsense.pool.ntp.org
  3.opnsense.pool.ntp.org

Interface: LAN_Admin
```

> Four NTP servers provide redundancy — if the first is unreachable,
> OPNsense falls back to the next automatically.
>
> Critical for Active Directory: Kerberos authentication fails if
> clocks are more than 5 minutes apart across domain members.
> All VMs must sync time from OPNsense or the same NTP source.

---

## Password Change — Security Hardening

Root password changed from default `opnsense` immediately after permanent install.

**Industry rule:** Default credentials on any network device must be changed
before the device is considered operational. Default credentials are publicly
known and are the first thing attackers check.

```
Default credentials left unchanged = critical vulnerability
Documented in: CVE databases, penetration testing checklists,
               CIS Benchmarks, NIST SP 800-53
```

---

## Configuration Backup

OPNsense config exported via:
```
System → Configuration → Backups → Download configuration
☑ Do not backup RRD data
☐ Encrypt (not needed for local storage)
```

Saved as: `configs/opnsense-backup-2026-03-08.xml`

> The entire OPNsense configuration lives in a single XML file.
> This backup can restore all interfaces, firewall rules, DHCP pools,
> reservations, and settings in seconds after a fresh install.
> In production, config backups are automated and stored offsite.

> ⚠️ configs/*.xml is listed in .gitignore — never push firewall
> configs to public repositories as they contain network topology details.

---

## Key Concepts Reinforced

- Kea DHCP is the modern replacement for ISC DHCP in OPNsense 24.x+
- Static DHCP reservations bind MAC addresses to IPs — servers always get the same address
- Management plane security — restrict SSH/HTTPS admin access to dedicated admin networks only
- NTP synchronization is mandatory for Active Directory / Kerberos authentication
- Default credentials must be changed before any device is considered operational
- FQDN format: hostname.domain = opnsense.lab.local
- SSH key pairs are production standard — password login is lab convenience only
- Config backups should never be pushed to public repos — add to .gitignore

---

## OPNsense Configuration — Final State

```
Hostname:        opnsense.lab.local
Version:         OPNsense 26.1.2_5 (amd64)
Install type:    Permanent (VDI — ZFS Stripe)

Interfaces:
  WAN        em0   10.0.2.15/24    DHCP
  LAN_Admin  em1   192.168.10.1/29 Static
  LAN_DMZ    em2   192.168.10.9/29 Static
  LAN_Attack em3   192.168.10.17/29 Static

DHCP:
  LAN_Admin  → 192.168.10.2–.6
  LAN_DMZ    → 192.168.10.10–.14
  LAN_Attack → 192.168.10.18–.22

Firewall:   8 rules across 3 interfaces + implicit deny
SSH:        Enabled on LAN_Admin only, port 22
NTP:        4x opnsense.pool.ntp.org
DNS:        8.8.8.8 (temporary)
```

---

## Evidence

| Screenshot | Description |
|---|---|
| ![NTP Config](evidence/entry-003/ntp-configuration.png) | NTP servers and LAN_Admin interface binding |
| ![Kea DHCP Subnets](evidence/entry-003/kea-dhcp-subnets.png) | Three DHCP subnets configured |
| ![Kea Reservations](evidence/entry-003/kea-dhcp-reservations.png) | Five static MAC reservations |
| ![SSH Config](evidence/entry-003/ssh-configuration.png) | SSH enabled on LAN_Admin only |
| ![Password Change](evidence/entry-003/password-change.png) | Root password change evidence |

---

## Next Session

- Install VirtualBox Guest Additions on WinServ (enable shared folders)
- Boot Kali and verify Block firewall rules
- Begin Active Directory setup on WinServ (LAN_Admin)
- Configure DNS and DHCP on DNS Server (LAN_DMZ)
- Test DHCP lease assignment on all VMs
