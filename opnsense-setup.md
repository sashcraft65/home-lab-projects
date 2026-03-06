# OPNsense Configuration Checklist

> Reference document for Phase 1 OPNsense setup. Follow in order.

---

## Pre-Flight — VirtualBox Adapter Wiring

Before booting OPNsense, verify adapter assignments in VirtualBox:

| Adapter | Type | Internal Network Name |
|---|---|---|
| Adapter 1 | NAT | *(none — internet facing)* |
| Adapter 2 | Internal Network | `LAN_Admin` |
| Adapter 3 | Internal Network | `LAN_Attack` |

**All other VMs:**

| VM | Adapter 1 |
|---|---|
| Windows Server 2022 | Internal Network: `LAN_Admin` |
| Ubuntu Desktop | Internal Network: `LAN_Admin` |
| Kali Linux | Internal Network: `LAN_Attack` |
| Metasploitable 2 | Internal Network: `LAN_Attack` |

> ⚠️ Internal Network names are case-sensitive in VirtualBox. `LAN_Admin` ≠ `lan_admin`

---

## Step 1 — Factory Reset OPNsense

At the OPNsense console:
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
OPT1 → vtnet2   (Adapter 3 — LAN_Attack)
```

---

## Step 3 — Set Interface IPs

**LAN (Admin):**
```
IP Address : 192.168.10.1
Subnet Mask: /28
```

**OPT1 (Attack Lab):**
```
IP Address : 192.168.10.17
Subnet Mask: /28
```

**WAN:**
```
Type: DHCP (automatic from VirtualBox NAT)
```

---

## Step 4 — Configure DHCP

Enable DHCP on both interfaces so VMs get IPs automatically:

**LAN_Admin DHCP Pool:**
```
Range: 192.168.10.2 – 192.168.10.14
```

**LAN_Attack DHCP Pool:**
```
Range: 192.168.10.18 – 192.168.10.30
```

> Set static DHCP leases for servers so their IPs never change.

---

## Step 5 — Apply Firewall Rules

Navigate to: `Firewall → Rules`

### LAN_Admin Rules (in order):
| # | Source | Destination | Action |
|---|---|---|---|
| 1 | LAN_Admin net | LAN_Attack net | Allow |
| 2 | LAN_Admin net | any | Allow |

### LAN_Attack Rules (in order):
| # | Source | Destination | Action |
|---|---|---|---|
| 1 | LAN_Attack net | LAN_Admin net | Block |
| 2 | LAN_Attack net | any | Block |

> Implicit deny covers everything not listed.

---

## Step 6 — Verification Tests

| Test | Command | Expected Result |
|---|---|---|
| Admin → Attack reachable | `ping 192.168.10.19` from Windows | Success |
| Attack → Admin blocked | `ping 192.168.10.2` from Kali | Timeout |
| Admin → Internet | `ping 8.8.8.8` from Windows | Success |
| Attack → Internet blocked | `ping 8.8.8.8` from Kali | Timeout |
| nmap from Kali → Metasploitable | `nmap 192.168.10.19` | Open ports listed |

---

## Status

- [ ] VirtualBox adapters configured
- [ ] OPNsense reset to defaults
- [ ] Interfaces assigned
- [ ] Static IPs set
- [ ] DHCP configured
- [ ] Firewall rules applied
- [ ] Verification tests passed
