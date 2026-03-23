# SOC Home Lab

A fully functional Security Operations Center (SOC) home lab built using VMware with Active Directory, Splunk SIEM, and attack simulation using Kali Linux.

---

## Overview

This project simulates a real-world enterprise environment to practice threat detection, log analysis, and SIEM operations. The lab includes a Windows Active Directory domain, a Splunk SIEM instance for centralized log collection, a Windows 10 target machine, and a Kali Linux attacker machine. Brute force attacks are simulated against SMB and RDP services, and the resulting telemetry is detected and visualized in Splunk.

---

## Architecture

```
+-------------------+          +-------------------------+
|   Kali Linux      |  Attack  |   Windows 10 (Target)  |
|  192.168.10.250   +--------->+   192.168.10.100        |
|  (Attacker)       |          |   Domain: cyberlab.local|
+-------------------+          +-------------------------+
                                          |
                                          | Logs via
                                          | Universal Forwarder
                                          v
+-------------------+          +-------------------------+
| Windows Server    |          |   Ubuntu Server         |
|  2022 (AD DC)     |          |   (Splunk SIEM)         |
|  192.168.10.7     +--------->+   192.168.10.10         |
|  cyberlab.local   | Logs via |   Splunk Enterprise     |
+-------------------+ Forwarder|   10.2.1                |
                               +-------------------------+

NAT Network: 192.168.10.0/24
```

---

## Lab Setup

### Network Configuration

| Machine              | IP Address       | Role              | OS                    |
|----------------------|------------------|-------------------|-----------------------|
| Kali Linux           | 192.168.10.250   | Attacker          | Kali Linux            |
| Windows Server 2022  | 192.168.10.7     | Active Directory  | Windows Server 2022   |
| Ubuntu Server        | 192.168.10.10    | Splunk SIEM       | Ubuntu Server         |
| Windows 10           | 192.168.10.100   | Target Endpoint   | Windows 10            |

All machines are connected via a VMware NAT network on the `192.168.10.0/24` subnet.

### Active Directory Configuration

- **Domain:** `cyberlab.local`
- **Domain Controller:** Windows Server 2022 (`192.168.10.7`)
- **Domain Users Created:**
  - `asmith`
  - `bjones`
  - `mbrown`

### Splunk SIEM Configuration

- **Version:** Splunk Enterprise 10.2.1
- **Host:** Ubuntu Server (`192.168.10.10`)
- **Web UI:** `http://192.168.10.10:8000`
- **Splunk Universal Forwarder** installed on:
  - Windows 10 (`192.168.10.100`)
  - Windows Server 2022 (`192.168.10.7`)
- **Data Input:** `WinEventLog:Security` (Windows Security Event Logs)
- **Receiving Port:** `9997`

---

## Attack Simulation

### Tool: Hydra (from Kali Linux)

Brute force attacks were launched from `192.168.10.250` against the domain-joined machines using a wordlist to test SMB and RDP services.

#### SMB Brute Force

```bash
hydra -l asmith -P /usr/share/wordlists/rockyou.txt smb://192.168.10.100
```

#### RDP Brute Force

```bash
hydra -l bjones -P /usr/share/wordlists/rockyou.txt rdp://192.168.10.100
```

These attacks generated a high volume of **Event ID 4625** (Failed Logon) entries in the Windows Security Event Log, which were forwarded to Splunk for analysis.

---

## Detection & Alerting

### Key Event ID

| Event ID | Description              |
|----------|--------------------------|
| 4625     | An account failed to log on |
| 4624     | An account was successfully logged on |

### Detection SPL Query

```spl
index=endpoint EventCode=4625
| stats count by src_ip, user, ComputerName
| where count > 5
| sort -count
```

### Splunk Alert: "Brute Force Detected"

| Setting        | Value                                 |
|----------------|---------------------------------------|
| Search         | `index=endpoint EventCode=4625 \| stats count by src_ip \| where count > 10` |
| Schedule       | Every hour (Cron: `0 * * * *`)        |
| Trigger        | When number of results > 0            |
| Action         | Add to triggered alerts               |

---

## Dashboard

### SOC Home Lab - Brute Force Monitor

A custom Splunk dashboard was built with the following panels:

| Panel                          | Visualization | SPL Summary                                  |
|-------------------------------|---------------|----------------------------------------------|
| Failed Logins Over Time        | Timechart     | `EventCode=4625 \| timechart count`           |
| Top Attacked Accounts          | Bar Chart     | `EventCode=4625 \| top user`                  |
| Attacker Source IPs            | Pie Chart     | `EventCode=4625 \| top src_ip`                |
| Login Success vs Failure       | Pie Chart     | `EventCode IN (4624,4625) \| stats count by EventCode` |
| Targeted Machines              | Bar Chart     | `EventCode=4625 \| top ComputerName`          |
| Failed Logins by User          | Bar Chart     | `EventCode=4625 \| stats count by user`       |

All queries and their full SPL are documented in [splunk-queries/queries.md](splunk-queries/queries.md).

---

## Tools Used

| Tool                        | Purpose                                      |
|-----------------------------|----------------------------------------------|
| VMware Workstation          | Hypervisor for hosting all VMs               |
| Windows Server 2022         | Active Directory Domain Controller           |
| Windows 10                  | Target endpoint joined to domain             |
| Ubuntu Server               | Host for Splunk SIEM                         |
| Splunk Enterprise 10.2.1    | SIEM — log ingestion, search, alerting       |
| Splunk Universal Forwarder  | Log shipping from Windows endpoints          |
| Kali Linux                  | Attacker machine                             |
| Hydra                       | Brute force attack simulation tool           |

---

## Key Learnings

- **SIEM Deployment:** Gained hands-on experience deploying and configuring Splunk Enterprise, including setting up data inputs, index management, and the Universal Forwarder.
- **Active Directory:** Configured a fully functional AD domain with multiple user accounts, simulating a real enterprise environment.
- **Log Forwarding:** Understood how Windows Security Event Logs are collected and forwarded to a centralized SIEM using the Splunk Universal Forwarder.
- **Threat Detection:** Built SPL queries to identify brute force patterns using Event ID 4625, with statistical thresholds to reduce noise.
- **Alerting:** Created time-based scheduled alerts in Splunk to automate detection and notification of suspicious activity.
- **Dashboard Building:** Developed a multi-panel SOC dashboard providing real-time visibility into authentication events, attack sources, and targeted accounts.
- **Attack Simulation:** Executed realistic brute force attacks using Hydra against SMB and RDP protocols and observed the resulting telemetry end-to-end.

---

## Screenshots

Screenshots of the lab setup, Splunk dashboard, and attack results are located in the [/screenshots](screenshots/) folder.

---

## Repository Structure

```
SOC-Home-Lab/
├── README.md
├── .gitignore
├── screenshots/          # Lab and dashboard screenshots
├── splunk-queries/
│   └── queries.md        # All SPL queries used in the lab
└── setup-notes/
    └── lab-config.md     # Configuration notes and setup steps
```
