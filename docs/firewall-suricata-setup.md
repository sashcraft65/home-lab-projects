# Firewall & Suricata Configuration

> Reference document for OPNsense firewall rules and Suricata IDS setup.
> Read as: `SOURCE → DESTINATION : ACTION`
> Rules are processed **top-down — first match wins.**

---

## Table of Contents
- [Firewall Rules](#firewall-rules)
- [Interface Naming](#interface-naming)
- [Suricata IDS — OPNsense](#suricata-ids--opnsense)
- [Suricata Agent — Metasploitable](#suricata-agent--metasploitable-2)
- [Wazuh SIEM Integration](#wazuh-siem-integration)
- [Verification Tests](#verification-tests)
- [Status Checklist](#status-checklist)

---

## Interface Naming

Before applying rules, rename interfaces in the OPNsense web GUI for clarity:

```
Interfaces → [OPT1] → Description: LAN_DMZ
Interfaces → [OPT2] → Description: LAN_Attack
```

| OPNsense Name | Our Name | Subnet |
|---|---|---|
| LAN | LAN_Admin | 192.168.10.0/29 |
| OPT1 | LAN_DMZ | 192.168.10.8/29 |
| OPT2 | LAN_Attack | 192.168.10.16/29 |

---

## Firewall Rules

Navigate to: `Firewall → Rules`

> ⚠️ **Rule order is critical.** Rules are read top-down — first match wins and stops processing. A permissive rule above a block rule will silently nullify the block.

### LAN_Admin Rules

| Priority | Source | Destination | Protocol | Action | Description |
|---|---|---|---|---|---|
| 1 | LAN_Admin net | LAN_DMZ net | any | ✅ Allow | Admin manages DMZ servers |
| 2 | LAN_Admin net | LAN_Attack net | any | ✅ Allow | Admin can reach attack lab |
| 3 | LAN_Admin net | any | any | ✅ Allow | Admin internet access |

### LAN_DMZ Rules

| Priority | Source | Destination | Protocol | Action | Description |
|---|---|---|---|---|---|
| 1 | LAN_DMZ net | LAN_Admin net | any | ❌ Block | DMZ cannot reach AD/admin |
| 2 | LAN_DMZ net | any | any | ✅ Allow | DMZ services need internet |

### LAN_Attack Rules

| Priority | Source | Destination | Protocol | Action | Description |
|---|---|---|---|---|---|
| 1 | LAN_Attack net | LAN_Admin net | any | ❌ Block | Prevent lateral movement |
| 2 | LAN_Attack net | LAN_DMZ net | any | ✅ Allow | Attack lab can target DMZ |
| 3 | LAN_Attack net | any | any | ❌ Block | Attack lab internet isolated |

### Implicit Deny
```
ANY → ANY : BLOCK  ← always last, always invisible, always enforced
```

> 🎓 **Stateful firewall note:** OPNsense automatically permits return traffic
> for established connections. No explicit inbound rules are needed.
> States: NEW → ESTABLISHED → RELATED

---

## How to Add a Rule in OPNsense Web GUI

```
Firewall → Rules → Select interface tab
Click: + Add (arrow up = insert at top, arrow down = append at bottom)

Fields to set:
  Action:            Pass / Block
  Interface:         select correct interface
  Direction:         in
  TCP/IP Version:    IPv4
  Protocol:          any
  Source:            select network alias or type subnet
  Destination:       select network alias or type subnet
  Description:       always fill this in — explain the rule purpose

Click: Save → Apply Changes
```

> ⚠️ Always click **Apply Changes** after saving — rules are staged but not
> active until applied.

---

## Suricata IDS — OPNsense

> **IDS mode only — not IPS.**
> IDS = detect and alert. IPS = detect and block inline.
> IDS is chosen deliberately so attacks complete and generate full observable
> alert chains for learning. Blocking would hide attack behavior.

### Install Plugin
```
System → Firmware → Plugins
Search: os-suricata
Click:  Install
Reboot OPNsense after install completes
```

### Configure Suricata
```
Services → Intrusion Detection → Administration → Settings tab

  ☑  Enabled
  ☑  IDS Mode          ← NOT IPS
  Interface:  LAN_Attack (vtnet3)
  Pattern matcher: Hyperscan (preferred) or AC (fallback)
  
Click: Apply
```

### Enable Emerging Threats Ruleset
```
Services → Intrusion Detection → Administration → Download tab

  ☑  ET open/emerging-all.rules
     Emerging Threats Community — free, actively maintained
     Industry standard ruleset used in real SOC environments

Click: Download & Update Rules
Wait for completion
```

### Verify Alerts
```
Services → Intrusion Detection → Alerts

Run nmap from Kali: nmap 192.168.10.19
Alerts should appear within seconds showing:
  - Timestamp
  - Source IP (192.168.10.18 — Kali)
  - Destination IP (192.168.10.19 — Metasploitable)
  - Signature name (e.g. "ET SCAN Nmap Scripting Engine")
  - Category
  - Severity
```

### Alert Log Location
```
/var/log/suricata/eve.json    ← JSON format, ingested by Wazuh
```

---

## Suricata Agent — Metasploitable 2

Host-level IDS on the vulnerable target provides a second visibility layer —
shows exactly which attacks reached the host vs what OPNsense caught at the
perimeter.

### Install Suricata
```bash
# SSH into Metasploitable from Kali or Admin machine
ssh msfadmin@192.168.10.19
# default password: msfadmin

sudo apt-get update
sudo apt-get install suricata -y
```

### Configure Interface Monitoring
```bash
sudo nano /etc/suricata/suricata.yaml

# Set interface (eth0 is default on Metasploitable)
af-packet:
  - interface: eth0

# Verify eve-log output is enabled
outputs:
  - eve-log:
      enabled: yes
      filename: /var/log/suricata/eve.json
```

### Download Rules and Start
```bash
sudo suricata-update
sudo systemctl enable suricata
sudo systemctl start suricata

# Verify running
sudo systemctl status suricata
sudo tail -f /var/log/suricata/eve.json
```

---

## Wazuh SIEM Integration

Wazuh on Ubuntu Desktop (192.168.10.3) ingests Suricata logs from both
OPNsense and Metasploitable for centralized correlation.

### OPNsense Log Forwarding to Wazuh
```
OPNsense: Services → Intrusion Detection → Administration
  ☑  Enable syslog forwarding
  Syslog destination: 192.168.10.3
  Port: 514 UDP
```

### Wazuh Agent on Metasploitable
```bash
# Install Wazuh agent — follow Wazuh docs for Ubuntu/Debian agent install
# Point agent manager IP to: 192.168.10.3
```

### Add Suricata Log Source to Wazuh Manager
```xml
<!-- Add to /var/ossec/etc/ossec.conf on Wazuh manager (Ubuntu Desktop) -->
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

### Restart Wazuh Manager
```bash
sudo systemctl restart wazuh-manager
```

---

## Visibility Architecture

```
Kali Linux  192.168.10.18
      │
      │  attack traffic
      ▼
┌─────────────────────────────┐
│  OPNsense — Suricata IDS    │  ← perimeter detection
│  LAN_Attack interface       │    sees ALL traffic leaving subnet
│  /var/log/suricata/eve.json │
└─────────────┬───────────────┘
              │  IDS mode — traffic passes through
              ▼
┌─────────────────────────────┐
│  Metasploitable 2           │  ← host-level detection
│  Suricata Agent             │    sees attacks that reach the target
│  /var/log/suricata/eve.json │
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│  Ubuntu Desktop             │  ← centralized correlation
│  Wazuh SIEM                 │    alerts from both sources
│  192.168.10.3               │    compare perimeter vs host detections
└─────────────────────────────┘
```

---

## Verification Tests

| # | Test | Command | From | Expected Result |
|---|---|---|---|---|
| 1 | Admin → DMZ | `ping 192.168.10.9` | WinServ AD | ✅ Success |
| 2 | Admin → Attack | `ping 192.168.10.17` | WinServ AD | ✅ Success |
| 3 | Admin → Internet | `ping 8.8.8.8` | WinServ AD | ✅ Success |
| 4 | DMZ → Admin blocked | `ping 192.168.10.1` | Ubuntu Server | ❌ Timeout |
| 5 | Attack → Admin blocked | `ping 192.168.10.1` | Kali | ❌ Timeout |
| 6 | Attack → DMZ | `ping 192.168.10.9` | Kali | ✅ Success |
| 7 | Attack → Internet blocked | `ping 8.8.8.8` | Kali | ❌ Timeout |
| 8 | nmap scan | `nmap 192.168.10.19` | Kali | ✅ Open ports listed |
| 9 | Suricata alert | check OPNsense UI after test 8 | OPNsense | ✅ Alerts generated |

---

## Status Checklist

### Firewall
- [ ] OPT1 renamed to LAN_DMZ in web GUI
- [ ] OPT2 renamed to LAN_Attack in web GUI
- [ ] LAN_Admin rules applied in correct order
- [ ] LAN_DMZ rules applied in correct order
- [ ] LAN_Attack rules applied in correct order
- [ ] All verification ping tests passed

### Suricata
- [ ] os-suricata plugin installed on OPNsense
- [ ] IDS mode enabled on LAN_Attack interface
- [ ] Emerging Threats ruleset downloaded and active
- [ ] Alerts visible after nmap test from Kali
- [ ] Suricata installed on Metasploitable 2
- [ ] eve.json logging confirmed on Metasploitable
- [ ] Wazuh ingesting logs from both sources
- [ ] Alerts correlating in Wazuh dashboard
