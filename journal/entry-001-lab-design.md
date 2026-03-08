# Entry 001 — Lab Design & Theory

**Date:** 2026-03-06
**Status:** ✅ Complete
**Phase:** Phase 0 — Planning & Design

---

## What Was Accomplished

- Defined lab objectives and mapped to certification targets
- Designed four-segment network architecture from scratch
- Split `192.168.10.0/27` into four `/29` subnets using VLSM
- Separated Windows Server roles — AD in LAN_Admin, DNS/DHCP in LAN_DMZ
- Added DMZ segment for internet-facing services
- Defined 8-rule firewall policy with implicit deny
- Assigned all host IPs across all four subnets
- Provisioned all VMs in VirtualBox

---

## Final Network Design

| Segment | Subnet | Gateway | Purpose |
|---|---|---|---|
| LAN_Admin | 192.168.10.0/29 | 192.168.10.1 | AD, Monitoring |
| LAN_DMZ | 192.168.10.8/29 | 192.168.10.9 | DNS, Web Services |
| LAN_Attack | 192.168.10.16/29 | 192.168.10.17 | Penetration Testing |
| Reserved | 192.168.10.24/29 | — | Future Expansion |

## Host IP Assignments

| Device | IP | Segment |
|---|---|---|
| OPNsense LAN | 192.168.10.1 | LAN_Admin |
| Windows Server (AD) | 192.168.10.2 | LAN_Admin |
| Ubuntu Desktop | 192.168.10.3 | LAN_Admin |
| OPNsense OPT1 | 192.168.10.9 | LAN_DMZ |
| Windows Server (DNS) | 192.168.10.10 | LAN_DMZ |
| Ubuntu Server | 192.168.10.11 | LAN_DMZ |
| OPNsense OPT2 | 192.168.10.17 | LAN_Attack |
| Kali Linux | 192.168.10.18 | LAN_Attack |
| Metasploitable 2 | 192.168.10.19 | LAN_Attack |

---

## Key Concepts Learned

- Each borrowed subnet bit halves available host addresses — `/27 → /28 → /29`
- VLSM right-sizes each subnet to actual host requirements
- Firewall rules read top-down — first match wins — rule order is critical
- `ANY → ANY : Allow` at the top of a ruleset nullifies all rules below it
- Stateful firewalls auto-permit return traffic via connection tracking
- Network address is never assignable — first usable IP is always the gateway by convention
- DMZ isolates internet-facing services from internal AD/admin infrastructure
- OPNsense operates at OSI Layers 3, 4, and 7 (with Deep Packet Inspection)
- nmap operates across OSI Layers 2, 3, 4 — and Layer 7 with `-sV` flag
- IDS (detect only) vs IPS (inline block) — IDS chosen for lab visibility

---

## Subnetting Work

Starting from `192.168.10.0/27`:

```
/27 = 32 addresses → split into four /29s

LAN_Admin  /29 → Network: .0   Usable: .1–.6    Broadcast: .7
LAN_DMZ    /29 → Network: .8   Usable: .9–.14   Broadcast: .15
LAN_Attack /29 → Network: .16  Usable: .17–.22  Broadcast: .23
Reserved   /29 → Network: .24  Usable: .25–.30  Broadcast: .31
```

---

## Evidence
- See `diagrams/topology.html` for visual network diagram
- See `docs/opnsense-setup.md` for full configuration checklist

---

## Next Session
- Configure VirtualBox adapters (4 adapters on OPNsense)
- Boot OPNsense and assign WAN / LAN / OPT1 / OPT2 interfaces
- Set static gateway IPs on all interfaces
