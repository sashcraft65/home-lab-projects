# 🖥️ Virtual IT Security Home Lab

> A segmented virtual network environment built for hands-on learning in networking, security operations, and ethical hacking. Designed to support progression toward CompTIA Security+, CCNA, and AWS SysOps / CloudOps Engineer certifications.

---

## 📋 Table of Contents
- [Lab Overview](#lab-overview)
- [Network Topology](#network-topology)
- [Subnet Design](#subnet-design)
- [IP Assignment](#ip-assignment)
- [Firewall Policy](#firewall-policy)
- [VM Inventory](#vm-inventory)
- [Build Phases](#build-phases)
- [Certification Mapping](#certification-mapping)
- [Journal](#journal)

---

## 🧭 Lab Overview

| Property | Detail |
|---|---|
| **Hypervisor** | Oracle VirtualBox |
| **Firewall / Router** | OPNsense |
| **Architecture** | Four-segment VLSM network with stateful firewall enforcement |
| **Base Block** | `192.168.10.0/27` |
| **Subnetting** | Four `/29` subnets via VLSM |
| **Purpose** | Attack/defense simulation, network administration, DMZ services, cloud integration |
| **Status** | 🔧 In Progress — Phase 1 |

---

## 🌐 Network Topology

```
                            INTERNET
                                │
                         ┌──────────────┐
                         │   OPNsense   │  WAN: DHCP (NAT)
                         │   Firewall   │  vtnet0/vtnet1/vtnet2/vtnet3
                         └──────┬───────┘
                                │
          ┌─────────────┬───────┴────────┬─────────────┐
          │             │                │             │
  ┌────────────┐ ┌────────────┐ ┌─────────────┐ ┌──────────────┐
  │ LAN_Admin  │ │  LAN_DMZ   │ │ LAN_Attack  │ │   Reserved   │
  │ .0/29      │ │ .8/29      │ │ .16/29      │ │ .24/29       │
  │            │ │            │ │             │ │              │
  │ WinServ AD │ │ WinServ DNS│ │ Kali Linux  │ │ Future use   │
  │ Ubuntu Mon │ │ Ubuntu Svr │ │ Metasploit  │ │              │
  └────────────┘ └────────────┘ └─────────────┘ └──────────────┘

  Admin  → DMZ    : ALLOW       DMZ    → Admin  : BLOCK
  Admin  → Attack : ALLOW       Attack → Admin  : BLOCK
  Admin  → WAN    : ALLOW       Attack → WAN    : BLOCK
  DMZ    → WAN    : ALLOW       Attack → DMZ    : ALLOW
```

---

## 📐 Subnet Design

**Base Block:** `192.168.10.0/27` — 32 addresses total
Split using VLSM into four equal `/29` subnets (6 usable hosts each):

| Segment | Subnet | Network Addr | Usable Range | Broadcast | Purpose |
|---|---|---|---|---|---|
| LAN_Admin | `192.168.10.0/29` | `.0` | `.1 – .6` | `.7` | Admin, AD, Monitoring |
| LAN_DMZ | `192.168.10.8/29` | `.8` | `.9 – .14` | `.15` | DNS, DHCP, Web Services |
| LAN_Attack | `192.168.10.16/29` | `.16` | `.17 – .22` | `.23` | Penetration Testing |
| Reserved | `192.168.10.24/29` | `.24` | `.25 – .30` | `.31` | Future Expansion |

> **Design rationale:** Four-segment architecture isolates administrative, DMZ, and attack traffic. OPNsense enforces all inter-segment policy. The DMZ allows internet-facing services without exposing the admin domain. The reserved /29 supports future additions without requiring a full redesign.

---

## 🖧 IP Assignment

### LAN_Admin — `192.168.10.0/29`

| Device | IP Address | Role |
|---|---|---|
| OPNsense LAN | `192.168.10.1` | Gateway / Firewall (vtnet1) |
| Windows Server 2022 | `192.168.10.2` | Active Directory / Domain Controller |
| Ubuntu Desktop | `192.168.10.3` | Monitoring / SOC Workstation / SIEM |
| *(Spare)* | `.4 – .6` | Future expansion |

### LAN_DMZ — `192.168.10.8/29`

| Device | IP Address | Role |
|---|---|---|
| OPNsense OPT1 | `192.168.10.9` | Gateway / Firewall (vtnet2) |
| Windows Server 2022 | `192.168.10.10` | DNS / DHCP Server |
| Ubuntu Server | `192.168.10.11` | Web / App Server |
| *(Spare)* | `.12 – .14` | Future: AWS VPN endpoint, additional services |

### LAN_Attack — `192.168.10.16/29`

| Device | IP Address | Role |
|---|---|---|
| OPNsense OPT2 | `192.168.10.17` | Gateway / Firewall (vtnet3) |
| Kali Linux | `192.168.10.18` | Penetration Testing / Attack Machine |
| Metasploitable 2 | `192.168.10.19` | Vulnerable Target |
| *(Spare)* | `.20 – .22` | Future: additional targets |

### Reserved — `192.168.10.24/29`

| Device | IP Address | Role |
|---|---|---|
| *(Unassigned)* | `.25 – .30` | Future expansion — no gateway configured until needed |

---

## 🔥 Firewall Policy

Rules applied at OPNsense. Read as: `SOURCE → DESTINATION : ACTION`
Rules are processed **top-down — first match wins.**

| # | Source | Destination | Action | Reason |
|---|---|---|---|---|
| 1 | `192.168.10.0/29` | `192.168.10.8/29` | ✅ Allow | Admin manages DMZ servers |
| 2 | `192.168.10.0/29` | `192.168.10.16/29` | ✅ Allow | Admin can reach attack lab |
| 3 | `192.168.10.0/29` | `0.0.0.0/0` | ✅ Allow | Admin internet access |
| 4 | `192.168.10.8/29` | `192.168.10.0/29` | ❌ Block | DMZ cannot reach admin / AD |
| 5 | `192.168.10.8/29` | `0.0.0.0/0` | ✅ Allow | DMZ services need internet |
| 6 | `192.168.10.16/29` | `192.168.10.0/29` | ❌ Block | Prevent lateral movement to admin |
| 7 | `192.168.10.16/29` | `192.168.10.8/29` | ✅ Allow | Attack lab can target DMZ |
| 8 | `192.168.10.16/29` | `0.0.0.0/0` | ❌ Block | Attack lab internet isolated |
| 9 | `ANY` | `ANY` | ❌ Block | Implicit deny — all unmatched traffic |

> **Note:** OPNsense is a **stateful firewall**. Return traffic for established connections is automatically permitted via connection tracking — no explicit inbound rules required.

---

## 🖥️ VM Inventory

| VM | OS | vDisk | Purpose | Network |
|---|---|---|---|---|
| OPNsense | FreeBSD-based | 16 GB | Firewall / Router | WAN + LAN_Admin + LAN_DMZ + LAN_Attack |
| Windows Server 2022 (AD) | Windows Server | 50 GB | Active Directory, Domain Controller | LAN_Admin |
| Ubuntu Desktop | Ubuntu 22.04 | 25 GB | Monitoring, SIEM | LAN_Admin |
| Windows Server 2022 (DNS) | Windows Server | 50 GB | DNS, DHCP | LAN_DMZ |
| Ubuntu Server | Ubuntu Server LTS | 25 GB | Web / App Server | LAN_DMZ |
| Kali Linux | Debian-based | 40 GB | Penetration Testing | LAN_Attack |
| Metasploitable 2 | Ubuntu-based | 8 GB | Vulnerable Target | LAN_Attack |

### Planned Additions
| VM / Service | Purpose | Network |
|---|---|---|
| AWS EC2 + S3 | Cloud hosting, hybrid cloud extension | Public / VPN into LAN_DMZ |

---

## 🔍 Intrusion Detection — Suricata

Suricata is deployed in two places for layered visibility:

| Component | Location | Role |
|---|---|---|
| **Suricata IDS — OPNsense** | OPNsense (choke point) | Monitors all inter-segment traffic at the firewall level |
| **Suricata Agent — Metasploitable 2** | `192.168.10.19` | Host-level visibility of attacks landing on the vulnerable target |

> **IDS vs IPS — Design Decision:** Suricata is configured in **IDS mode (detect only)** — not IPS (inline blocking). This is intentional. In a learning lab, blocking attacks at the firewall prevents you from observing the full attack chain. IDS mode lets all traffic through while generating alerts for analysis in the SIEM.

### Suricata on OPNsense
- Installed via: `System → Firmware → Plugins → os-suricata`
- Monitors: LAN_Attack interface — primary choke point for all outbound attack traffic
- Ruleset: **Emerging Threats Community** (free, industry-standard, actively maintained)
- Alert log: `/var/log/suricata/eve.json`
- Mode: **IDS — Alert only, no blocking**

### Suricata Agent on Metasploitable 2
- Installed directly on the vulnerable VM for host-level detection
- Captures attack payloads that reach the target after passing through OPNsense
- Provides a second alert perspective: firewall-level vs host-level
- Useful for understanding which attacks OPNsense detects vs what actually reaches the host

### Visibility Architecture
```
Kali (attacker)  192.168.10.18
        │
        ▼
[OPNsense — Suricata IDS]       ← sees ALL traffic leaving LAN_Attack
        │
        │  IDS mode — traffic passes through, alerts generated
        ▼
[Metasploitable — Suricata Agent]  ← sees attacks that reach the host
        │
        ▼
   Alert logs (eve.json)
        │
        ▼
[Ubuntu Desktop — Wazuh SIEM]   ← correlates alerts from both sources
```

---

## 🗺️ Build Phases

### ✅ Phase 0 — Planning & Design
- [x] Define lab objectives and certification targets
- [x] Design four-segment network topology
- [x] Calculate subnets using VLSM (`/27` → four `/29`s)
- [x] Define firewall policy for all segments
- [x] Assign host IPs across all subnets
- [x] Provision all VMs in VirtualBox

### 🔧 Phase 1 — Foundation (Current)
- [ ] Configure VirtualBox adapters for all VMs
- [ ] Reset OPNsense to factory defaults
- [ ] Assign WAN / LAN / OPT1 / OPT2 interfaces in OPNsense
- [ ] Set static gateway IPs on all OPNsense interfaces
- [ ] Configure DHCP pools for all three active subnets
- [ ] Apply firewall rules per policy table
- [ ] Verify routing and firewall with ping and nmap

### ⏳ Phase 2 — Administration & Monitoring
- [ ] Install Active Directory on Windows Server (LAN_Admin)
- [ ] Configure DNS and DHCP on Windows Server (LAN_DMZ)
- [ ] Join Ubuntu Desktop to domain
- [ ] Install Wazuh on Ubuntu Desktop (SIEM)
- [ ] Install Suricata plugin on OPNsense (`os-suricata`)
- [ ] Enable Emerging Threats Community ruleset on Suricata
- [ ] Set Suricata to monitor LAN_Attack interface in IDS mode
- [ ] Install Suricata agent on Metasploitable 2
- [ ] Configure Wazuh to ingest Suricata `eve.json` logs from both sources

### ⏳ Phase 3 — Attack & Defense
- [ ] Run nmap scans from Kali → Metasploitable
- [ ] Exploit vsftpd 2.3.4 backdoor via Metasploit Framework
- [ ] Capture and analyze traffic in Wireshark
- [ ] Review Suricata alerts on OPNsense triggered by nmap and exploits
- [ ] Review Suricata agent alerts on Metasploitable for host-level visibility
- [ ] Correlate OPNsense vs host-level alerts in Wazuh SIEM
- [ ] Compare what OPNsense detects vs what reaches the target

### ⏳ Phase 4 — Cloud Extension
- [ ] Configure Ubuntu Server as web host (LAN_DMZ)
- [ ] Launch AWS EC2 instance
- [ ] Host static website on S3
- [ ] Configure Security Groups (AWS firewall equivalent)
- [ ] Set up IAM roles with least-privilege access
- [ ] Establish VPN tunnel between lab DMZ and AWS VPC

---

## 🎓 Certification Mapping

| Lab Activity | Certification | Domain |
|---|---|---|
| Subnetting, VLSM, routing | CCNA | IP Connectivity |
| OSI model, switching concepts | CCNA | Network Fundamentals |
| Inter-VLAN routing, OPNsense | CCNA | IP Connectivity & Services |
| Firewall rules, segmentation | CompTIA Security+ | Network Security |
| Stateful inspection, threat detection | CompTIA Security+ | Security Architecture |
| DMZ design, defense-in-depth | CompTIA Security+ | Security Architecture |
| nmap, Metasploit, vulnerability scanning | CompTIA Security+ | Security Operations |
| Suricata IDS, alert tuning, ruleset management | CompTIA Security+ | Security Operations |
| Network traffic analysis, threat signatures | CompTIA Security+ | Threat Detection |
| Log correlation, dual-source SIEM ingestion | CompTIA Security+ | Incident Response |
| SIEM, log monitoring, incident response | CompTIA Security+ | Security Operations |
| EC2, S3, IAM, VPC, Security Groups | AWS SysOps / CloudOps Engineer | Cloud Infrastructure |
| CloudWatch monitoring, automation | AWS SysOps / CloudOps Engineer | Monitoring & Reporting |
| Least-privilege IAM, compliance controls | AWS SysOps / CloudOps Engineer | Security & Compliance |

---

## 📓 Journal

### Entry 001 — Lab Design & Theory
**Date:** 2026-03-06
**Status:** ✅ Complete

**Completed:**
- Designed four-segment architecture: LAN_Admin, LAN_DMZ, LAN_Attack, Reserved
- Split `192.168.10.0/27` into four `/29` subnets using VLSM
- Separated Windows Server roles — AD in LAN_Admin, DNS/DHCP in LAN_DMZ
- Defined 8-rule firewall policy with implicit deny
- Studied stateful vs stateless firewall behavior
- Mapped lab components to OSI model layers 1–7
- Understood nmap operation across Layers 2, 3, and 4

**Key Concepts Learned:**
- Each borrowed subnet bit halves available host addresses
- VLSM right-sizes subnets to actual host requirements
- Firewall rules read top-down — first match wins — rule order is critical
- `ANY → ANY : Allow` at top nullifies all rules below it
- Stateful firewalls auto-permit return traffic via connection tracking
- Network address is never assignable — first usable IP goes to the gateway
- DMZ isolates internet-facing services from internal AD/admin infrastructure

**Next Session:**
- Configure VirtualBox adapters (4 adapters on OPNsense)
- Reset and reconfigure OPNsense from scratch
- Assign all four interfaces and set gateway IPs

---

*Built for learning. Documented for growth.*
