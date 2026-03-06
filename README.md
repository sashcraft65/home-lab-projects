<<<<<<< HEAD
# 🖥️ Virtual IT Security Home Lab

> A segmented virtual network environment built for hands-on learning in networking, security operations, and ethical hacking. Designed to support progression toward CompTIA Network+, Security+, AWS Cloud Practitioner, and eJPT certifications.

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
| **Architecture** | Segmented dual-subnet with firewall enforcement |
| **Purpose** | Attack/defense simulation, network administration, cloud integration |
| **Status** | 🔧 In Progress — Phase 1 |

---

## 🌐 Network Topology

```
                        INTERNET
                            │
                     ┌──────────────┐
                     │   OPNsense   │  WAN: DHCP (NAT)
                     │   Firewall   │
                     └──────┬───────┘
                            │
              ┌─────────────┴─────────────┐
              │                           │
     ┌────────────────┐         ┌─────────────────┐
     │  LAN_Admin     │         │   LAN_Attack     │
     │ 192.168.10.0/28│         │192.168.10.16/28  │
     │                │         │                  │
     │ Windows Server │         │  Kali Linux      │
     │ Ubuntu Monitor │         │  Metasploitable 2│
     └────────────────┘         └──────────────────┘

     Admin → Attack : ALLOW
     Attack → Admin : BLOCK
     Admin → WAN    : ALLOW
     Attack → WAN   : BLOCK
```

---

## 📐 Subnet Design

**Base Block:** `192.168.10.0/27` (30 usable IPs)
Split using VLSM into two `/28` subnets:

| Segment | Subnet | Network Addr | Usable Range | Broadcast | Purpose |
|---|---|---|---|---|---|
| LAN_Admin | `192.168.10.0/28` | `.0` | `.1 – .14` | `.15` | Admin & Monitoring |
| LAN_Attack | `192.168.10.16/28` | `.16` | `.17 – .30` | `.31` | Attack Lab |

> **Design rationale:** Segmenting attack machines from admin machines prevents lateral movement. OPNsense enforces all inter-segment traffic via stateful firewall rules.

---

## 🖧 IP Assignment

### LAN_Admin — `192.168.10.0/28`

| Device | IP Address | Role |
|---|---|---|
| OPNsense LAN | `192.168.10.1` | Gateway / Firewall |
| Windows Server 2022 | `192.168.10.2` | Active Directory, DNS, DHCP |
| Ubuntu Desktop | `192.168.10.3` | Monitoring / SOC Workstation |
| *(Reserved)* | `.4 – .14` | Future expansion |

### LAN_Attack — `192.168.10.16/28`

| Device | IP Address | Role |
|---|---|---|
| OPNsense OPT1 | `192.168.10.17` | Gateway / Firewall |
| Kali Linux | `192.168.10.18` | Attack Machine |
| Metasploitable 2 | `192.168.10.19` | Vulnerable Target |
| *(Reserved)* | `.20 – .30` | Future expansion |

---

## 🔥 Firewall Policy

Rules applied at OPNsense. Read as: `SOURCE → DESTINATION : ACTION`

| # | Source | Destination | Action | Reason |
|---|---|---|---|---|
| 1 | `192.168.10.0/28` | `192.168.10.16/28` | ✅ Allow | Admin can reach attack lab |
| 2 | `192.168.10.16/28` | `192.168.10.0/28` | ❌ Block | Prevent lateral movement to admin |
| 3 | `192.168.10.0/28` | `0.0.0.0/0` | ✅ Allow | Admin internet access |
| 4 | `192.168.10.16/28` | `0.0.0.0/0` | ❌ Block | Isolate attack lab from internet |
| 5 | `ANY` | `ANY` | ❌ Block | Implicit deny — all unmatched traffic |

> **Note:** OPNsense is a **stateful firewall**. Return traffic for established connections is automatically permitted — no explicit inbound rules required.

---

## 🖥️ VM Inventory

| VM | OS | vDisk | Purpose | Network |
|---|---|---|---|---|
| OPNsense | FreeBSD-based | 16 GB | Firewall / Router | WAN + LAN_Admin + LAN_Attack |
| Windows Server 2022 | Windows Server | 50 GB | AD, DNS, DHCP, Admin | LAN_Admin |
| Ubuntu Desktop | Ubuntu 22.04 | 25 GB | Monitoring, SIEM | LAN_Admin |
| Kali Linux | Debian-based | 40 GB | Penetration Testing | LAN_Attack |
| Metasploitable 2 | Ubuntu-based | 8 GB | Vulnerable Target | LAN_Attack |

### Planned Additions
| VM | Purpose | Network |
|---|---|---|
| Ubuntu Server | Web / App Server | LAN_Admin |
| AWS EC2 + S3 | Cloud hosting, hybrid extension | Public / VPN |

---

## 🗺️ Build Phases

### ✅ Phase 0 — Planning & Design
- [x] Define lab objectives and certification targets
- [x] Design network topology and segmentation
- [x] Calculate subnets using VLSM
- [x] Define firewall policy
- [x] Provision all VMs in VirtualBox

### 🔧 Phase 1 — Foundation (Current)
- [ ] Configure VirtualBox adapters for all VMs
- [ ] Reset OPNsense and assign WAN / LAN / OPT1 interfaces
- [ ] Set static IPs on OPNsense
- [ ] Configure DHCP on OPNsense for both subnets
- [ ] Verify inter-subnet routing and firewall rules
- [ ] Test connectivity with ping and nmap

### ⏳ Phase 2 — Administration
- [ ] Install Active Directory on Windows Server 2022
- [ ] Configure DNS and DHCP on Windows Server
- [ ] Join Ubuntu Desktop to domain
- [ ] Install Wazuh or Splunk Free on Ubuntu (SIEM)

### ⏳ Phase 3 — Attack & Defense
- [ ] Run nmap scans from Kali → Metasploitable
- [ ] Exploit vsftpd 2.3.4 backdoor via Metasploit
- [ ] Capture and analyze traffic in Wireshark
- [ ] Review alerts generated in SIEM

### ⏳ Phase 4 — Cloud Extension
- [ ] Deploy Ubuntu Server locally
- [ ] Launch AWS EC2 instance
- [ ] Host static website on S3
- [ ] Configure Security Groups (AWS equivalent of firewall rules)
- [ ] Set up IAM roles and least-privilege access

---

## 🎓 Certification Mapping

| Lab Activity | Certification | Domain |
|---|---|---|
| Subnetting, VLSM, routing | CompTIA Network+ | IP Addressing |
| Firewall rules, segmentation | CompTIA Security+ | Network Security |
| OSI model application | CompTIA Network+ | Networking Concepts |
| nmap, Metasploit | eJPT | Penetration Testing |
| EC2, S3, IAM, VPC | AWS Cloud Practitioner / SAA | Cloud Concepts |
| SIEM, log monitoring | CompTIA Security+ | Security Operations |

---

## 📓 Journal

### Entry 001 — Lab Design & Theory
**Date:** 2026-03-06
**Status:** ✅ Complete

**Completed:**
- Designed dual-subnet architecture using VLSM
- Split `192.168.10.0/27` into two `/28` subnets
- Defined 4-rule firewall policy with implicit deny
- Studied stateful vs stateless firewall behavior
- Mapped lab components to OSI model layers
- Understood nmap operation across Layers 2, 3, and 4

**Key Concepts Learned:**
- `/27 → two /28s` using VLSM — network address boundaries
- Firewall rules read top-down, first match wins
- `ANY → ANY : Allow` at top = no firewall at all (misconfiguration risk)
- Stateful firewalls track connection state — return traffic auto-permitted
- OPNsense operates at Layers 3, 4, and 7 (with DPI)
- Network address (`.16`) is reserved — first usable host is `.17`

**Next Session:**
- Configure VirtualBox adapters
- Reset and reconfigure OPNsense from scratch

---

*Built for learning. Documented for growth.*
=======
# Home-Lab-Projects
>>>>>>> 92b8aed28417a25b292420642d9b6ce444f3f79c
