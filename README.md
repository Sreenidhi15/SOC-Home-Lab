# SOC Home Lab — Active Directory Attack Detection & SIEM Monitoring

A fully operational Security Operations Center (SOC) home lab built with VMware, Active Directory, Splunk Enterprise, and simulated brute-force attacks. This project demonstrates end-to-end security monitoring: from log collection and SIEM configuration to attack simulation, detection engineering, and alerting.

**Status:** ✅ Fully operational and tested

---

## Table of Contents
- [Overview](#overview)
- [Architecture](#architecture)
- [Lab Setup](#lab-setup)
- [Active Directory Configuration](#active-directory-configuration)
- [Splunk SIEM Configuration](#splunk-siem-configuration)
- [Attack Simulation](#attack-simulation)
- [Detection & Alerting](#detection--alerting)
- [Dashboard](#dashboard)
- [Skills Demonstrated](#skills-demonstrated)
- [Repository Structure](#repository-structure)

---

## Overview

This project simulates a real-world enterprise environment to practice threat detection, log analysis, and incident response. The lab includes:

- A Windows Active Directory domain with multiple user accounts
- A Splunk Enterprise SIEM for centralized log collection and analysis
- A Windows 10 endpoint forwarding logs via Splunk Universal Forwarder
- A Kali Linux attacker machine simulating brute-force attacks
- Custom detection rules and alerting for suspicious authentication activity
- A real-time Splunk dashboard monitoring attack indicators

**Key capabilities:**
- ✅ Endpoint monitoring via Windows Security Event Logs
- ✅ Centralized SIEM log analysis with Splunk Enterprise
- ✅ Simulated brute-force attacks using Hydra
- ✅ Custom detection rules with configurable thresholds
- ✅ Real-time dashboard visualization
- ✅ Incident response practice aligned with MITRE ATT&CK

---

## Architecture

```
┌─────────────────────┐          Attack          ┌──────────────────────┐
│   Kali Linux        │ ◄────────────────────►   │  Windows 10 (Target)│
│  192.168.10.250     │   SMB & RDP Brute Force   │  192.168.10.100     │
│  (Attacker)         │                           │  cyberlab.local     │
└─────────────────────┘                           └──────────┬───────────┘
                                                               │
                                                   Log Forwarding via
                                                   Universal Forwarder
                                                               │
        ┌──────────────────────────┐             ┌────────────▼────────────┐
        │  Windows Server 2022     │             │   Ubuntu Server         │
        │  (AD Domain Controller)  │────────────►│   (Splunk SIEM)         │
        │  192.168.10.7            │ Log Forward │   192.168.10.10         │
        │  cyberlab.local          │             │   Splunk Enterprise 10.2.1│
        └───────────────────────────┘             └─────────────────────────┘

NAT Network: 192.168.10.0/24
```

---

## Lab Setup

### Network Configuration

| Machine              | IP Address      | Role                | Operating System      |
|-----------------------|------------------|---------------------|------------------------|
| Kali Linux            | 192.168.10.250   | Attacker            | Kali Linux             |
| Windows Server 2022   | 192.168.10.7     | Active Directory DC | Windows Server 2022    |
| Ubuntu Server         | 192.168.10.10    | Splunk SIEM         | Ubuntu 22.04.5 LTS     |
| Windows 10            | 192.168.10.100   | Target Endpoint     | Windows 10             |

**Network type:** VMware NAT Network
**Network range:** 192.168.10.0/24

![Virtual Network Editor](screenshots/01-network-config.png)

---

## Active Directory Configuration

- **Domain:** `cyberlab.local`
- **Domain Controller:** Windows Server 2022 (`192.168.10.7`)
- **DNS:** Configured and operational

### Domain Users

| Username | Full Name    | Role            |
|----------|--------------|-----------------|
| asmith   | Alice Smith  | SOC Analyst     |
| bjones   | Bob Jones    | SOC Analyst     |
| mbrown   | Macy Brown   | Security Manager|

![Active Directory Users and Computers](screenshots/02-active-directory-users.png)

### Windows 10 — Domain Joined

- **Computer name:** `DESKTOP-2DV84F6.cyberlab.local`
- **Domain:** `cyberlab.local`
- **Status:** Domain membership confirmed

![Windows 10 Domain Properties](screenshots/03-windows-10-domain-joined.png)

---

## Splunk SIEM Configuration

- **Version:** Splunk Enterprise 10.2.1
- **Host:** Ubuntu Server (`192.168.10.10`)
- **Web UI:** `http://192.168.10.10:8000`
- **Receiving port:** 9997
- **Index:** `endpoint`

### Log Forwarding

Splunk Universal Forwarders are installed on:
- Windows 10 (`192.168.10.100`)
- Windows Server 2022 (`192.168.10.7`)

**Data source:** `WinEventLog:Security` (Windows Security Event Logs)

![Ubuntu Splunk Server Booted](screenshots/05-ubuntu-server-booted.png)
![Splunk Web Interface Login](screenshots/06-splunk-home-login.png)

### Monitored Event IDs

| Event ID | Description                           |
|----------|----------------------------------------|
| 4625     | An account failed to log on            |
| 4624     | An account was successfully logged on  |

![Splunk Log Collection — EventCode 4625](screenshots/04-splunk-search-results.png)

---

## Attack Simulation

Brute-force attacks were launched from the Kali Linux machine (`192.168.10.250`) against domain-joined systems using **Hydra**.

### SMB Brute Force
```bash
hydra -l asmith -P /usr/share/wordlists/rockyou.txt smb://192.168.10.100
```

### RDP Brute Force
```bash
hydra -l bjones -P /usr/share/wordlists/rockyou.txt rdp://192.168.10.100
```

These attacks generated a high volume of **Event ID 4625** (Failed Logon) entries in the Windows Security Event Log, which were forwarded to Splunk for analysis.

---

## Detection & Alerting

### Detection Query
```spl
index=endpoint EventCode=4625
| stats count by src_ip, user, ComputerName
| where count > 5
| sort -count
```

### Alert: "Brute Force Detected"

| Setting  | Value                                                                        |
|----------|-------------------------------------------------------------------------------|
| Search   | `index=endpoint EventCode=4625 \| stats count by src_ip \| where count > 10` |
| Schedule | Every hour (`0 * * * *`)                                                      |
| Trigger  | When number of results > 0                                                    |
| Action   | Add to triggered alerts                                                       |
| Severity | High                                                                          |

---

## Dashboard

### SOC Home Lab — Brute Force Monitor

| Panel                     | Visualization | SPL Summary                                              |
|----------------------------|----------------|------------------------------------------------------------|
| Failed Logins Over Time    | Timechart      | `EventCode=4625 \| timechart count`                        |
| Top Attacked Accounts      | Bar Chart      | `EventCode=4625 \| top user`                               |
| Attacker Source IPs        | Pie Chart      | `EventCode=4625 \| top src_ip`                             |
| Login Success vs Failure   | Pie Chart      | `EventCode IN (4624,4625) \| stats count by EventCode`     |
| Targeted Machines          | Bar Chart      | `EventCode=4625 \| top ComputerName`                       |
| Failed Logins by User      | Bar Chart      | `EventCode=4625 \| stats count by user`                    |

![SOC Home Lab Dashboard](screenshots/07-splunk-dashboard-complete.png)

All queries and their full SPL are documented in [`splunk-queries/queries.md`](splunk-queries/queries.md).

---

## Skills Demonstrated

**Security Operations Center (SOC)**
- End-to-end SOC infrastructure design
- Centralized log management and real-time monitoring

**SIEM Administration**
- Splunk Enterprise deployment, index/input configuration
- Universal Forwarder deployment across endpoints

**Active Directory**
- Domain controller configuration and user account management

**Threat Detection**
- Windows Security Event log analysis
- Attack pattern recognition and custom SPL detection rules

**Attack Simulation**
- Brute-force attack execution in a controlled environment (Hydra)

**Incident Response**
- Event investigation, alert triage, and timeline reconstruction

---

## Repository Structure

```
SOC-Home-Lab/
├── README.md                    # Complete documentation
├── screenshots/                 # All proof screenshots
│   ├── 01-network-config.png
│   ├── 02-active-directory-users.png
│   ├── 03-windows-10-domain-joined.png
│   ├── 04-splunk-search-results.png
│   ├── 05-ubuntu-server-booted.png
│   ├── 06-splunk-home-login.png
│   └── 07-splunk-dashboard-complete.png
├── setup-notes/                 # Detailed setup instructions
│   └── lab-config.md
├── splunk-queries/               # SPL queries and alert definitions
│   └── queries.md
└── .gitignore
```

---

**Last updated:** October 2026
**All components verified and operational.**
