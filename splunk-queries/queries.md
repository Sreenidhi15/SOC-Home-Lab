# Splunk SPL Queries — SOC Home Lab

All SPL (Search Processing Language) queries used in the SOC Home Lab for detection, alerting, and dashboard panels.

---

## Detection Queries

### 1. Failed Logins (Event ID 4625)

```spl
index=endpoint EventCode=4625
| stats count by src_ip, user, ComputerName
| sort -count
```

### 2. Brute Force Detection (threshold: >10 failures)

```spl
index=endpoint EventCode=4625
| stats count by src_ip
| where count > 10
| sort -count
```

### 3. Brute Force per User (threshold: >5 failures)

```spl
index=endpoint EventCode=4625
| stats count by src_ip, user, ComputerName
| where count > 5
| sort -count
```

---

## Alert Query

### Brute Force Detected Alert (runs hourly)

```spl
index=endpoint EventCode=4625
| stats count by src_ip
| where count > 10
```

**Trigger condition:** Number of results > 0
**Schedule:** Every hour (`0 * * * *`)

---

## Dashboard Panel Queries

### Panel 1: Failed Logins Over Time

```spl
index=endpoint EventCode=4625
| timechart span=5m count as "Failed Logins"
```

### Panel 2: Top Attacked Accounts

```spl
index=endpoint EventCode=4625
| top limit=10 user
| rename user as "Account", count as "Failed Attempts", percent as "Percent"
```

### Panel 3: Attacker Source IPs

```spl
index=endpoint EventCode=4625
| top limit=10 src_ip
| rename src_ip as "Source IP", count as "Attempts", percent as "Percent"
```

### Panel 4: Login Success vs Failure

```spl
index=endpoint EventCode IN (4624, 4625)
| eval Status=if(EventCode=4624, "Success", "Failure")
| stats count by Status
```

### Panel 5: Targeted Machines

```spl
index=endpoint EventCode=4625
| top limit=10 ComputerName
| rename ComputerName as "Target Machine", count as "Failed Logins", percent as "Percent"
```

### Panel 6: Failed Logins by User

```spl
index=endpoint EventCode=4625
| stats count by user
| sort -count
| rename user as "Username", count as "Failed Login Count"
```

---

## General Investigation Queries

### All Security Events in Last 24 Hours

```spl
index=endpoint sourcetype="WinEventLog:Security"
| head 100
```

### Search All Events from Attacker IP

```spl
index=endpoint src_ip=192.168.10.250
| table _time, EventCode, user, ComputerName, src_ip
| sort -_time
```

### Successful Logins Following Failed Attempts (Potential Compromise)

```spl
index=endpoint EventCode IN (4624, 4625)
| transaction user maxspan=10m
| where eventcount > 5 AND EventCode=4624
| table _time, user, src_ip, ComputerName
```

### Unique Accounts Targeted per Source IP

```spl
index=endpoint EventCode=4625
| stats dc(user) as unique_accounts, count as total_attempts by src_ip
| where unique_accounts > 2
| sort -total_attempts
```
