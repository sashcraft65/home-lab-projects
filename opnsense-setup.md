# OPNsense Configuration Checklist

> Reference document for Phase 1 OPNsense setup. Follow steps in order.

---

## Pre-Flight — VirtualBox Adapter Wiring

Configure these adapters in VirtualBox **before** booting OPNsense:

| Adapter | Type | Internal Network Name | OPNsense Interface |
|---|---|---|---|
| Adapter 1 | NAT | *(internet facing)* | WAN — vtnet0 |
| Adapter 2 | Internal Network | `LAN_Admin` | LAN — vtnet1 |
| Adapter 3 | Internal Network | `LAN_DMZ` | OPT1 — vtnet2 |
| Adapter 4 | Internal Network | `LAN_Attack` | OPT2 — vtnet3 |

**All other VMs — single adapter each:**

| VM | Adapter 1 |
|---|---|
| Windows Server 2022 (AD) | Internal Network: `LAN_Admin` |
| Ubuntu Desktop | Internal Network: `LAN_Admin` |
| Windows Server 2022 (DNS) | Internal Network: `LAN_DMZ` |
| Ubuntu Server | Internal Network: `LAN_DMZ` |
| Kali Linux | Internal Network: `LAN_Attack` |
| Metasploitable 2 | Internal Network: `LAN_Attack` |

> ⚠️ Internal Network names are **case-sensitive** in VirtualBox. `LAN_Admin` ≠ `lan_admin`

---

## Step 1 — Factory Reset OPNsense

At the OPNsense console menu:
```
Option 4 → Reset to factory defaults
Confirm  → YES
Wait for reboot
```

---

## Step 2 — Assign Interfaces

At console after reboot:
```
WAN  → vtnet0   (Adapter 1 — NAT)
LAN  → vtnet1   (Adapter 2 — LAN_Admin)
OPT1 → vtnet2   (Adapter 3 — LAN_DMZ)
OPT2 → vtnet3   (Adapter 4 — LAN_Attack)
```

---

## Step 3 — Set Interface IPs

**LAN (LAN_Admin):**
```
IP Address:  192.168.10.1
Subnet Mask: /29  (255.255.255.248)
```

**OPT1 (LAN_DMZ):**
```
IP Address:  192.168.10.9
Subnet Mask: /29  (255.255.255.248)
```

**OPT2 (LAN_Attack):**
```
IP Address:  192.168.10.17
Subnet Mask: /29  (255.255.255.248)
```

**WAN:**
```
Type: DHCP — assigned automatically by VirtualBox NAT
```

---

## Step 4 — Configure DHCP Pools

Enable DHCP on LAN, OPT1, and OPT2:

| Interface | DHCP Range |
|---|---|
| LAN_Admin | `192.168.10.2 – 192.168.10.6` |
| LAN_DMZ | `192.168.10.10 – 192.168.10.14` |
| LAN_Attack | `192.168.10.18 – 192.168.10.22` |

> Set static DHCP leases for servers so their IPs never change after reboot.

---

## Step 5 — Apply Firewall Rules

Navigate to: `Firewall → Rules`

### LAN_Admin Rules (in order):
| # | Source | Destination | Action |
|---|---|---|---|
| 1 | LAN_Admin net | LAN_DMZ net | Allow |
| 2 | LAN_Admin net | LAN_Attack net | Allow |
| 3 | LAN_Admin net | any | Allow |

### LAN_DMZ Rules (in order):
| # | Source | Destination | Action |
|---|---|---|---|
| 1 | LAN_DMZ net | LAN_Admin net | Block |
| 2 | LAN_DMZ net | any | Allow |

### LAN_Attack Rules (in order):
| # | Source | Destination | Action |
|---|---|---|---|
| 1 | LAN_Attack net | LAN_Admin net | Block |
| 2 | LAN_Attack net | LAN_DMZ net | Allow |
| 3 | LAN_Attack net | any | Block |

> Implicit deny covers all unmatched traffic automatically.

---

## Step 6 — Verification Tests

| Test | Command | From | Expected |
|---|---|---|---|
| Admin → DMZ | `ping 192.168.10.9` | Windows Server (AD) | Success |
| Admin → Attack | `ping 192.168.10.17` | Windows Server (AD) | Success |
| Admin → Internet | `ping 8.8.8.8` | Windows Server (AD) | Success |
| DMZ → Admin blocked | `ping 192.168.10.1` | Ubuntu Server | Timeout |
| Attack → Admin blocked | `ping 192.168.10.1` | Kali | Timeout |
| Attack → DMZ | `ping 192.168.10.9` | Kali | Success |
| Attack → Internet blocked | `ping 8.8.8.8` | Kali | Timeout |
| nmap Attack → DMZ | `nmap 192.168.10.10` | Kali | Open ports listed |
| nmap Attack → Metasploit | `nmap 192.168.10.19` | Kali | Open ports listed |

---

## Status Checklist

- [ ] VirtualBox adapters configured on all VMs
- [ ] OPNsense reset to factory defaults
- [ ] All four interfaces assigned (WAN/LAN/OPT1/OPT2)
- [ ] Static gateway IPs set on LAN, OPT1, OPT2
- [ ] DHCP pools configured for all three active subnets
- [ ] Firewall rules applied in correct order
- [ ] All verification tests passed
