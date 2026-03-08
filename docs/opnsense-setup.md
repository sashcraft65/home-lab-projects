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

---

## Suricata IDS — OPNsense Configuration

### Install Suricata Plugin
```
System → Firmware → Plugins
Search: os-suricata
Click:  Install
Reboot OPNsense after install
```

### Enable and Configure Suricata
```
Services → Intrusion Detection → Administration

Settings tab:
  ☑ Enabled
  ☑ IDS Mode  (NOT IPS — keeps traffic flowing for learning visibility)
  Interface:  LAN_Attack (vtnet3) — monitor all outbound attack traffic
  Pattern matcher: Hyperscan (best performance) or AC (fallback)
```

> ⚠️ **IDS not IPS:** IPS mode would block attacks inline — this prevents you from observing the full attack chain during lab exercises. Always use IDS mode in a learning environment.

### Enable Emerging Threats Ruleset
```
Services → Intrusion Detection → Administration → Download tab

Enable:  ☑ ET open/emerging-all.rules
         (Emerging Threats Community — free, actively maintained)

Click:   Download & Update Rules
```

### Verify Alerts Are Generating
```
Services → Intrusion Detection → Alerts

Run nmap from Kali against any target — alerts should appear within seconds.
Fields to review:
  - Timestamp
  - Source / Destination IP
  - Alert signature name
  - Category (e.g. "Attempted Information Leak", "Exploit")
```

### Alert Log Location
```
/var/log/suricata/eve.json   ← JSON format, ingested by Wazuh SIEM
```

---

## Suricata Agent — Metasploitable 2 Configuration

### Install Suricata on Metasploitable
```bash
# SSH into Metasploitable from Kali or Admin machine
ssh msfadmin@192.168.10.19

# Update package list and install Suricata
sudo apt-get update
sudo apt-get install suricata -y
```

### Configure Suricata to Monitor Local Interface
```bash
# Edit Suricata config
sudo nano /etc/suricata/suricata.yaml

# Set the interface to monitor (eth0 is default on Metasploitable)
af-packet:
  - interface: eth0

# Set eve.json log output (should be enabled by default)
outputs:
  - eve-log:
      enabled: yes
      filename: /var/log/suricata/eve.json
```

### Download Emerging Threats Rules
```bash
sudo suricata-update
sudo systemctl restart suricata
sudo systemctl enable suricata
```

### Verify Suricata is Running
```bash
sudo systemctl status suricata
sudo tail -f /var/log/suricata/eve.json
```

---

## Wazuh SIEM — Ingest Suricata Logs

Configure Wazuh on Ubuntu Desktop to collect `eve.json` from both sources:

### OPNsense Log Forwarding
```
OPNsense: Services → Intrusion Detection → Administration
  Enable syslog forwarding → Ubuntu Desktop IP (192.168.10.3)
  Port: 514 UDP
```

### Metasploitable Agent
```bash
# Install Wazuh agent on Metasploitable
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo apt-key add -
# Follow Wazuh agent install docs for Ubuntu/Debian
# Point agent to Wazuh manager at 192.168.10.3
```

### Wazuh Manager — Add Suricata Decoder
```xml
<!-- Add to /var/ossec/etc/ossec.conf on Wazuh manager -->
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

---

## Suricata Status Checklist

- [ ] Suricata plugin installed on OPNsense
- [ ] Suricata set to IDS mode on LAN_Attack interface
- [ ] Emerging Threats ruleset downloaded and active
- [ ] Alerts visible in OPNsense UI after nmap test
- [ ] Suricata installed on Metasploitable 2
- [ ] eve.json logging confirmed on Metasploitable
- [ ] Wazuh ingesting logs from both sources
- [ ] Alerts correlating in Wazuh dashboard
