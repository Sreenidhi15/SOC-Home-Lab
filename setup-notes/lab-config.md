# Lab Configuration Notes

Setup steps and configuration details for the SOC Home Lab.

---

## VMware Network Setup

1. Open VMware Workstation → Edit → Virtual Network Editor
2. Add a new NAT network: `VMnet8` (or custom)
3. Set subnet to `192.168.10.0` / `255.255.255.0`
4. Assign static IPs to each VM via OS network settings (not DHCP)

---

## Windows Server 2022 — Active Directory Setup

### Install AD DS Role

```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
```

### Promote to Domain Controller

```powershell
Install-ADDSForest `
  -DomainName "cyberlab.local" `
  -DomainNetbiosName "CYBERLAB" `
  -InstallDns:$true `
  -Force:$true
```

### Create Domain Users

```powershell
$password = ConvertTo-SecureString "Password123!" -AsPlainText -Force

New-ADUser -Name "Alice Smith"  -SamAccountName "asmith" -UserPrincipalName "asmith@cyberlab.local"  -AccountPassword $password -Enabled $true
New-ADUser -Name "Bob Jones"    -SamAccountName "bjones" -UserPrincipalName "bjones@cyberlab.local"  -AccountPassword $password -Enabled $true
New-ADUser -Name "Mary Brown"   -SamAccountName "mbrown" -UserPrincipalName "mbrown@cyberlab.local"  -AccountPassword $password -Enabled $true
```

---

## Windows 10 — Join Domain

1. Set DNS to `192.168.10.7` (AD DC IP)
2. System Properties → Change → Domain: `cyberlab.local`
3. Enter domain admin credentials when prompted
4. Reboot

---

## Ubuntu Server — Splunk Installation

### Download and Install Splunk Enterprise

```bash
wget -O splunk.deb 'https://download.splunk.com/products/splunk/releases/10.2.1/linux/splunk-10.2.1-<build>-linux-2.6-amd64.deb'
sudo dpkg -i splunk.deb
sudo /opt/splunk/bin/splunk start --accept-license
sudo /opt/splunk/bin/splunk enable boot-start
```

### Enable Receiving on Port 9997

```bash
sudo /opt/splunk/bin/splunk enable listen 9997 -auth admin:changeme
```

### Access Splunk Web UI

```
http://192.168.10.10:8000
```

---

## Windows Endpoints — Splunk Universal Forwarder Setup

### Install on Windows 10 and Windows Server 2022

1. Download Splunk Universal Forwarder from splunk.com
2. Run installer, set:
   - Receiving indexer: `192.168.10.10`
   - Receiving port: `9997`

### Configure inputs.conf (add Windows Security logs)

Path: `C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf`

```ini
[WinEventLog://Security]
index = endpoint
disabled = false
```

### Restart the forwarder

```powershell
Restart-Service SplunkForwarder
```

---

## Kali Linux — Hydra Attack Commands

### SMB Brute Force

```bash
hydra -l asmith -P /usr/share/wordlists/rockyou.txt smb://192.168.10.100
```

### RDP Brute Force

```bash
hydra -l bjones -P /usr/share/wordlists/rockyou.txt rdp://192.168.10.100
```

> Note: RDP brute force may require the `freerdp` or `xfreerdp` package installed on Kali.

---

## Splunk — Create Alert

1. Run the detection search in Splunk Web
2. Click **Save As → Alert**
3. Configure:
   - Title: `Brute Force Detected`
   - Schedule: `Run on Cron Schedule` → `0 * * * *` (hourly)
   - Trigger: `Number of Results > 0`
   - Actions: `Add to Triggered Alerts`

---

## Splunk — Create Dashboard

1. Go to **Dashboards → Create New Dashboard**
2. Name: `SOC Home Lab - Brute Force Monitor`
3. Add panels using the SPL queries in [splunk-queries/queries.md](../splunk-queries/queries.md)
4. Set panel visualizations (timechart, bar chart, pie chart) as appropriate
