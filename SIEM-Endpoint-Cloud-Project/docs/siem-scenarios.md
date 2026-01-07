# SIEM/EDR Scenarios - Demonstration Guide

## Overview

This document outlines all the security scenarios tested during the SIEM/EDR lab exercise. Each scenario generates specific events that are captured and analyzed by Wazuh.

---

## Scenario 1: SSH Brute Force Attack (Linux)

### Objective

Simulate multiple failed SSH login attempts to trigger authentication failure alerts.

### Environment

- **Target**: Linux-Client (Ubuntu 22.04)
- **Source**: Local machine or another client
- **Public IP**: 3.230.21.137
- **Port**: 22/TCP

### Execution Steps

#### Step 1: Attempt Multiple Failed Logins

```bash
# From your local machine
# Attempt SSH with invalid credentials (10 times)

for i in {1..10}; do
  ssh fakeuser@3.230.21.137
  # When prompted: enter any incorrect password
  # System will return "Permission denied (publickey)"
done
```

#### Step 2: Expected Behavior

```
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
...
fakeuser@3.230.21.137: Permission denied (publickey).
```

**Each failed attempt is logged in**: `/var/log/auth.log`

### Events Generated

| Field                 | Value                                                                                        |
| --------------------- | -------------------------------------------------------------------------------------------- |
| **Log Source**        | `/var/log/auth.log`                                                                          |
| **Service**           | sshd (SSH Daemon)                                                                            |
| **Event Type**        | Authentication Failed                                                                        |
| **Pattern**           | "Invalid user", "Connection closed by authenticating user"                                   |
| **Log Entry Example** | `Jan 7 16:45:32 ip-10-0-9-39 sshd[1234]: Invalid user fakeuser from 203.0.113.45 port 54321` |

### Wazuh Detection

**In Wazuh Dashboard**:

1. Navigate to: **Threat hunting** or **Security events**
2. Filter by:
   - **Agent**: LINUX-CLIENT
   - **Rule**: sshd or authentication_failed
3. Look for Alert Rule IDs:
   - **5710**: SSH authentication failures
   - **5702**: Multiple SSH failed attempts

**Alert Details**:

```json
{
  "timestamp": "2026-01-07T16:45:32.000Z",
  "agent": "LINUX-CLIENT",
  "rule.id": 5710,
  "rule.description": "sshd: authentication failed",
  "source.ip": "203.0.113.45",
  "source.user": "fakeuser",
  "action": "authentication_failed",
  "event.module": "sshd"
}
```

### Forensic Analysis

```bash
# On Linux-Client, analyze the auth log
grep "fakeuser" /var/log/auth.log

# Output:
# Jan  7 16:45:32 ip-10-0-9-39 sshd[1234]: Invalid user fakeuser from 203.0.113.45 port 54321
# Jan  7 16:45:33 ip-10-0-9-39 sshd[1235]: Invalid user fakeuser from 203.0.113.45 port 54322
# ... (repeated 10 times)

# Count failed attempts
grep "fakeuser" /var/log/auth.log | wc -l
# Output: 10
```

### MITRE ATT&CK Mapping

| Category          | Technique         | ID        |
| ----------------- | ----------------- | --------- |
| **Tactic**        | Credential Access |
| **Technique**     | Brute Force       | T1110     |
| **Sub-technique** | Password Guessing | T1110.001 |

---

## Scenario 2: Privilege Escalation (Linux)

### Objective

Demonstrate privilege escalation detection via sudo command.

### Execution Steps

```bash
# On Linux-Client SSH session
sudo su -

# Or simulate multiple sudo attempts
sudo whoami
sudo id
sudo cat /etc/shadow
```

### Events Generated

| Field          | Value                     |
| -------------- | ------------------------- |
| **Log Source** | `/var/log/auth.log`       |
| **Event Type** | Sudo command execution    |
| **Pattern**    | "sudo:", "sudo execution" |

### Wazuh Detection

**Alert Rule**:

- **ID**: 5402 (sudo execution)
- **Description**: User executed sudo command

**Dashboard Filter**:

- Agent: LINUX-CLIENT
- Rule: sudo or privilege_escalation

### Example Alert

```json
{
  "timestamp": "2026-01-07T16:50:15.000Z",
  "agent": "LINUX-CLIENT",
  "rule.id": 5402,
  "rule.description": "sudo: executed",
  "source.user": "ubuntu",
  "action": "privilege_escalation",
  "command": "sudo su -"
}
```

### MITRE ATT&CK Mapping

| Category          | Technique                         | ID        |
| ----------------- | --------------------------------- | --------- |
| **Tactic**        | Privilege Escalation              |
| **Technique**     | Abuse Elevation Control Mechanism | T1548     |
| **Sub-technique** | Sudo/Sudo Caching                 | T1548.003 |

---

## Scenario 3: File Integrity Monitoring (Linux)

### Objective

Monitor and detect changes to critical system files.

### Execution Steps

```bash
# On Linux-Client, modify a critical file
echo "test" | sudo tee -a /etc/passwd

# Or create a suspicious file
sudo touch /etc/suspicious-file.txt
sudo echo "malware" > /etc/malware.sh

# Or modify sudoers
sudo echo "ubuntu ALL=(ALL) NOPASSWD: ALL" | sudo tee -a /etc/sudoers.d/ubuntu
```

### Events Generated

| Field               | Value                              |
| ------------------- | ---------------------------------- |
| **Event Type**      | File Integrity Monitoring (FIM)    |
| **Files Monitored** | /etc, /home, /root, /bin, /usr/bin |
| **Detection**       | File added, modified, or deleted   |

### Wazuh Detection

**Alert Rule**:

- **ID**: 550 (File modification detected)
- **Description**: File integrity monitoring alert

**Dashboard**:

- Agent: LINUX-CLIENT
- Rule: file_integrity or FIM
- File: /etc/passwd or /etc/sudoers

### MITRE ATT&CK Mapping

| Category      | Technique                     | ID    |
| ------------- | ----------------------------- | ----- |
| **Tactic**    | Defense Evasion / Persistence |
| **Technique** | Modify System Files           | T1601 |

---

## Scenario 4: Windows Failed Logon (Windows)

### Objective

Detect multiple failed RDP/login attempts on Windows Server.

### Environment

- **Target**: Windows-Client (Windows Server 2025)
- **Public IP**: 54.ABC.XYZ.DEF (example)
- **Port**: 3389/TCP (RDP)

### Execution Steps

#### Step 1: Attempt Failed RDP Logins

```powershell
# Method 1: Via RDP Client (from local machine)
# Remote Desktop Connection
# Computer: <WINDOWS_CLIENT_PUBLIC_IP>
# Username: baduser
# Password: wrongpassword123
# Click "Connect"
# Wait for "The user name or password is incorrect" error
# Repeat 3-5 times
```

#### Step 2: Expected Event

**Windows Event Viewer**:

- Event ID: **4625** (An account failed to log on)

### Events Generated

| Field            | Value                                  |
| ---------------- | -------------------------------------- |
| **Event Source** | Windows Security Log                   |
| **Event ID**     | 4625                                   |
| **Event Type**   | Authentication Failure                 |
| **Log Path**     | Event Viewer → Windows Logs → Security |

### Wazuh Detection

**Alert Rule**:

- **ID**: 60100 (Windows failed logon)
- **Description**: Failed logon attempt detected

**Dashboard Filter**:

- Agent: WINDOWS-CLIENT
- Event ID: 4625
- Rule: Failed authentication

### Example Alert in Wazuh

```json
{
  "timestamp": "2026-01-07T17:00:45.000Z",
  "agent": "WINDOWS-CLIENT",
  "rule.id": 60100,
  "rule.description": "Windows: Failed authentication attempt",
  "source.user": "baduser",
  "event.code": 4625,
  "winlog.event_data.SubStatus": "0xC000006E",
  "source.ip": "203.0.113.100",
  "action": "authentication_failed"
}
```

### Forensic Analysis (Windows)

```powershell
# View failed logon events
Get-WinEvent -LogName Security -FilterXPath "*[System[EventID=4625]]" -MaxEvents 10 |
  Format-List TimeCreated, Message

# Output:
# TimeCreated : 1/7/2026 5:00:45 PM
# Message    : An account failed to log on.
#              Subject: Security ID S-1-0-0, Account Name -, Domain -, ...
#              Failure Information: Failure Reason: Unknown user or bad password
#              Account For Which Logon Failed: Account Name: baduser
```

### MITRE ATT&CK Mapping

| Category          | Technique         | ID        |
| ----------------- | ----------------- | --------- |
| **Tactic**        | Credential Access |
| **Technique**     | Brute Force       | T1110     |
| **Sub-technique** | Password Guessing | T1110.001 |

---

## Scenario 5: User Creation (Windows)

### Objective

Detect creation of new local user accounts.

### Execution Steps

```powershell
# On Windows-Client (run PowerShell as Administrator)

# Create a new user
net user labuser P@ssw0rd! /add

# Add to administrators group
net localgroup administrators labuser /add

# Verify
net user labuser
net localgroup administrators
```

### Events Generated

| Field        | Value                                             |
| ------------ | ------------------------------------------------- |
| **Event ID** | 4720 (User account created)                       |
| **Event ID** | 4732 (User added to security-enabled local group) |
| **Source**   | Windows Security Log                              |

### Wazuh Detection

**Alert Rules**:

- **ID**: 60113 (User created)
- **ID**: 60114 (Group membership changed)

**Dashboard Filter**:

- Agent: WINDOWS-CLIENT
- Event ID: 4720 or 4732
- Rule: user_account_created or group_modified

### Example Wazuh Alerts

```json
{
  "timestamp": "2026-01-07T17:05:20.000Z",
  "agent": "WINDOWS-CLIENT",
  "rule.id": 60113,
  "rule.description": "Windows: User account created",
  "event.code": 4720,
  "winlog.event_data.TargetUserName": "labuser",
  "winlog.event_data.NewUsbDeviceCount": "0"
}
```

### MITRE ATT&CK Mapping

| Category          | Technique      | ID        |
| ----------------- | -------------- | --------- |
| **Tactic**        | Persistence    |
| **Technique**     | Create Account | T1136     |
| **Sub-technique** | Local Account  | T1136.001 |

---

## Scenario 6: Sysmon Process Execution (Windows with Sysmon)

### Objective

Track and monitor process creation events via Sysmon EDR.

### Prerequisites

- Sysmon installed on Windows-Client
- SwiftOnSecurity configuration or similar

### Execution Steps

```powershell
# Run various commands (each triggers Sysmon Event ID 1)

whoami
ipconfig /all
Get-Process
powershell.exe -Command "Get-ChildItem C:\"
tasklist /v
netstat -ano
systeminfo
```

### Events Generated

| Field          | Value                                              |
| -------------- | -------------------------------------------------- |
| **Event ID**   | 1 (Process creation)                               |
| **Log Source** | Microsoft-Windows-Sysmon/Operational               |
| **Tracked**    | Process name, parent process, command line, hashes |

### Wazuh Detection

**Alert Rule**:

- **ID**: 61101 (Sysmon process execution)
- **Description**: Sysmon: Process created

**Dashboard Filter**:

- Agent: WINDOWS-CLIENT
- Log: Sysmon
- Event ID: 1

### Example Sysmon Alert

```json
{
  "timestamp": "2026-01-07T17:10:30.000Z",
  "agent": "WINDOWS-CLIENT",
  "rule.id": 61101,
  "rule.description": "Sysmon: Process creation",
  "source.process.name": "powershell.exe",
  "source.process.parent_name": "explorer.exe",
  "source.process.command_line": "powershell.exe -Command \"Get-ChildItem C:\\\"",
  "source.process.executable": "C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\powershell.exe"
}
```

### MITRE ATT&CK Mapping

| Category          | Technique                         | ID        |
| ----------------- | --------------------------------- | --------- |
| **Tactic**        | Execution                         |
| **Technique**     | Command and Scripting Interpreter | T1059     |
| **Sub-technique** | PowerShell                        | T1059.001 |

---

## Scenario 7: Sysmon Network Connection (Windows with Sysmon)

### Objective

Monitor outbound network connections.

### Execution Steps

```powershell
# Trigger network connection events (Sysmon Event ID 3)

# Ping external server
ping google.com

# DNS lookup
nslookup github.com

# HTTP connection
Invoke-WebRequest -Uri "https://www.google.com" -UseBasicParsing

# SSH/RDP connection attempt
Test-NetConnection -ComputerName 8.8.8.8 -Port 443
```

### Events Generated

| Field          | Value                                          |
| -------------- | ---------------------------------------------- |
| **Event ID**   | 3 (Network connection)                         |
| **Log Source** | Microsoft-Windows-Sysmon/Operational           |
| **Tracked**    | Source/Destination IP, Port, Protocol, Process |

### Wazuh Detection

**Alert Rule**:

- **ID**: 61102 (Sysmon network connection)
- **Description**: Sysmon: Network connection detected

### Example Network Alert

```json
{
  "timestamp": "2026-01-07T17:15:45.000Z",
  "agent": "WINDOWS-CLIENT",
  "rule.id": 61102,
  "rule.description": "Sysmon: Network connection",
  "source.process.name": "powershell.exe",
  "source.ip": "10.0.32.Y",
  "destination.ip": "142.251.33.14",
  "destination.port": 443,
  "protocol": "tcp"
}
```

### MITRE ATT&CK Mapping

| Category      | Technique                  | ID    |
| ------------- | -------------------------- | ----- |
| **Tactic**    | Command and Control        |
| **Technique** | Application Layer Protocol | T1071 |

---

## Summary of Events

### Linux Events Captured

| Scenario             | Alert ID | Event Type  | Source          |
| -------------------- | -------- | ----------- | --------------- |
| SSH Brute Force      | 5710     | Auth Failed | sshd            |
| Privilege Escalation | 5402     | Sudo Exec   | auth.log        |
| File Modification    | 550      | FIM         | ossec-syscheckd |

### Windows Events Captured

| Scenario      | Alert ID | Event ID | Source       |
| ------------- | -------- | -------- | ------------ |
| Failed Logon  | 60100    | 4625     | Security Log |
| User Created  | 60113    | 4720     | Security Log |
| Group Changed | 60114    | 4732     | Security Log |

### Sysmon Events Captured (Windows)

| Scenario           | Alert ID | Event ID | Details          |
| ------------------ | -------- | -------- | ---------------- |
| Process Created    | 61101    | 1        | Process creation |
| Network Connection | 61102    | 3        | Network activity |
| Registry Modified  | 61103    | 13       | Registry changes |

---

## Expected Timeline

| Time | Event              | Agent          | Status                 |
| ---- | ------------------ | -------------- | ---------------------- |
| T+0  | SSH attempts begin | LINUX-CLIENT   | ✅ Detected            |
| T+5  | Sudo escalation    | LINUX-CLIENT   | ✅ Detected            |
| T+10 | File modifications | LINUX-CLIENT   | ✅ Detected            |
| T+15 | RDP failed login   | WINDOWS-CLIENT | ✅ Detected            |
| T+20 | User creation      | WINDOWS-CLIENT | ✅ Detected            |
| T+25 | Process execution  | WINDOWS-CLIENT | ✅ Detected via Sysmon |
| T+30 | Network connection | WINDOWS-CLIENT | ✅ Detected via Sysmon |

---

## Analysis in Wazuh Dashboard

### How to View Scenarios

1. **Login to Wazuh Dashboard**

   - URL: https://<WAZUH_SERVER_IP>

2. **Navigate to Threat Hunting**

   - Menu → Threat hunting

3. **Filter by Event Type**

   - Agent: LINUX-CLIENT or WINDOWS-CLIENT
   - Rule ID: 5710, 60100, 61101, etc.
   - Timestamp: During scenario execution

4. **View Alert Details**
   - Click on any alert to see full details
   - Includes source IP, user, process, and more

### Dashboard Queries

**Linux SSH Brute Force**:

```
agent.name: "LINUX-CLIENT" AND rule.id: 5710
```

**Windows Failed Logon**:

```
agent.name: "WINDOWS-CLIENT" AND event.code: 4625
```

**Sysmon Process**:

```
agent.name: "WINDOWS-CLIENT" AND winlog.source: "Sysmon" AND event.code: 1
```

---

## Lessons Learned

1. ✅ **SIEM centralizes logs** - All events from multiple systems visible in one place
2. ✅ **EDR provides process-level insight** - Sysmon reveals behavioral details
3. ✅ **Correlation matters** - Multiple failed logins detected automatically
4. ✅ **Timeliness is critical** - Real-time alerts enable rapid response
5. ✅ **Multi-OS monitoring** - Windows and Linux events collected uniformly

---

**Document Version**: 1.0  
**Last Updated**: 7 January 2026
