# Wazuh Agent Installation - Windows Server 2025

## Prerequisites

- Windows Server 2025 or Windows 10/11
- RDP access to the instance
- Administrator privileges
- PowerShell 5.0 or higher
- Internet access

---

## Step 1: Connect via RDP

### From Windows Machine

1. **Open Remote Desktop Connection**

   - Press `Win + R` → Type `mstsc`

2. **Connect to Windows Instance**

   - Computer: `<WINDOWS_CLIENT_PUBLIC_IP>`
   - Username: `Administrator`
   - Password: (Get from AWS EC2 Connect)

3. **Accept Certificate Warning**

---

## Step 2: Access Wazuh Dashboard

### From Windows Browser

1. **Open Internet Explorer or Edge**

   ```
   https://<WAZUH_SERVER_PUBLIC_IP>
   ```

2. **Login**

   ```
   Username: admin
   Password: [from wazuh-passwords.txt]
   ```

3. **Navigate to Agent Deployment**
   - Menu → **Agents Management** → **Summary**
   - Click **"Deploy new agent"**

---

## Step 3: Get Agent Installation Command

### In Wazuh Dashboard

1. **Select Windows**

   - Choose Windows as the Operating System

2. **Agent Configuration**

   ```
   Manager IP: 10.0.32.10
   Agent Name: WINDOWS-CLIENT
   Agent Group: default
   ```

3. **Copy the PowerShell Command**
   (The dashboard will generate a one-liner)

---

## Step 4: Install Wazuh Agent via PowerShell

### Open PowerShell as Administrator

1. **Press `Win + X`** → Select **"Windows PowerShell (Admin)"**

2. **Paste the Wazuh Installation Command**

   ```powershell
   # Example (your command will be different):
   Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.5-1.msi -OutFile wazuh-agent-4.7.5-1.msi;
   msiexec.exe /i wazuh-agent-4.7.5-1.msi /q WAZUH_MANAGER='10.0.32.10' WAZUH_AGENT_NAME='WINDOWS-CLIENT'
   ```

3. **Press Enter**
   - Wait for installation to complete
   - Should see exit code 0 (success)

---

## Step 5: Verify Agent Installation

### Check Service Status

1. **Open Services Manager**

   - Press `Win + R` → Type `services.msc`

2. **Look for "Wazuh Agent"**

   - Status should be: **Running**
   - Startup Type: **Automatic**

3. **If Not Running**
   - Right-click "Wazuh Agent" → **Start**

### Check Installation Directory

```powershell
# Open PowerShell and check
dir "C:\Program Files (x86)\ossec-agent\"

# Should show:
# - bin/
# - etc/
# - logs/
# - queue/
```

---

## Step 6: Configure Agent (Optional)

### Edit Agent Configuration

```powershell
# Open notepad as Admin
notepad "C:\Program Files (x86)\ossec-agent\ossec.conf"
```

**Key sections** (usually pre-configured):

```xml
<!-- Manager configuration -->
<client>
    <server>
        <address>10.0.32.10</address>
        <port>1514</port>
        <protocol>tcp</protocol>
    </server>
    <config-profile>default</config-profile>
    <notify_time>10</notify_time>
    <time-reconnect>40</time-reconnect>
    <auto_restart>yes</auto_restart>
</client>

<!-- Log collection (Windows Event Log) -->
<localfile>
    <location>Security</location>
    <log_format>eventchannel</log_format>
</localfile>
```

---

## Step 7: Install & Configure Sysmon (EDR Enhancement)

### Download Sysmon

```powershell
# Download Sysmon
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "C:\Sysmon.zip"

# Extract
Expand-Archive -Path "C:\Sysmon.zip" -DestinationPath "C:\Sysmon" -Force
```

### Download Sysmon Configuration

```powershell
# Use a community config (example: SwiftOnSecurity config)
cd C:\Sysmon

Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" `
    -OutFile "sysmonconfig.xml"
```

### Install Sysmon with Configuration

```powershell
# Install Sysmon with config
C:\Sysmon\sysmon.exe -accepteula -i C:\Sysmon\sysmonconfig.xml

# Output: Configuration installed
```

### Verify Sysmon

```powershell
# Check service
Get-Service | Where-Object {$_.Name -like "*Sysmon*"}

# Should show Sysmon64 running

# Check Sysmon logs (PowerShell)
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 10
```

---

## Step 8: Configure Wazuh to Collect Sysmon Logs

### SSH into Wazuh Server

```bash
sudo nano /var/ossec/etc/ossec.conf
```

### Add Sysmon Monitoring

Find the agent configuration for `WINDOWS-CLIENT` and add:

```xml
<!-- Sysmon monitoring -->
<localfile>
    <location>Microsoft-Windows-Sysmon/Operational</location>
    <log_format>eventchannel</log_format>
    <only-future-events>no</only-future-events>
</localfile>

<!-- Windows Security monitoring -->
<localfile>
    <location>Security</location>
    <log_format>eventchannel</log_format>
    <only-future-events>no</only-future-events>
</localfile>
```

### Restart Wazuh Manager

```bash
sudo systemctl restart wazuh-manager
```

---

## Step 9: Verify Agent Connection in Wazuh Dashboard

### Check Active Agents

1. **Login to Wazuh Dashboard**
2. **Navigate to**: Agents Management → Summary
3. **Look for "WINDOWS-CLIENT"**
   - Status: **Active**
   - Last Seen: Recent timestamp
   - OS: Windows Server 2025 [version]

---

## Step 10: Test Event Generation (Optional)

### Trigger a Windows Security Event

```powershell
# Attempt failed RDP login (2-5 times)
# This will create Event ID 4625 in Security log
```

### Trigger a Process Creation (Sysmon)

```powershell
# Run a command - Sysmon will log it
whoami
ipconfig /all
Get-Process
```

### Check Dashboard for Events

1. **Wazuh Dashboard** → **Threat hunting** or **Security events**
2. **Filter by Agent**: WINDOWS-CLIENT
3. **You should see**:
   - Windows Security events
   - Sysmon process creation events
   - Network connections (if any)

---

## Troubleshooting

### Agent Not Connecting

```powershell
# Check Wazuh Agent logs
Get-Content "C:\Program Files (x86)\ossec-agent\logs\ossec.log" -Tail 20

# Look for: "Connected to manager"
# Or error: "Failed to connect"
```

### Restart Agent Service

```powershell
# Restart Wazuh Agent
Restart-Service -Name "Wazuh" -Force

# Verify restart
Get-Service -Name "Wazuh" | Select Status
```

### Check Network Connectivity

```powershell
# Test connection to manager
Test-NetConnection -ComputerName 10.0.32.10 -Port 1514 -InformationLevel Detailed

# Should show: TcpTestSucceeded : True
```

### Remove Agent (If Needed)

```powershell
# Uninstall
msiexec.exe /x "C:\Program Files (x86)\ossec-agent\wazuh-agent-4.7.5-1.msi" /quiet

# Or via Control Panel
# Settings → Apps → Apps & features → Search "Wazuh" → Uninstall
```

---

## Performance Considerations

### Monitor System Impact

```powershell
# Check CPU/Memory usage
Get-Process | Where-Object {$_.Name -like "*ossec*"}

# Should be minimal (< 5% CPU, < 50MB RAM)
```

### Adjust Configuration for Performance

If high resource usage:

```powershell
# Reduce log collection frequency
notepad "C:\Program Files (x86)\ossec-agent\ossec.conf"

# Find: <notify_time>10</notify_time>
# Change to: <notify_time>30</notify_time>
```

---

## Next Steps

✅ **Windows Agent is installed and connected**

Proceed to:

1. [Deploy Linux Agent](wazuh_agent_linux.md)
2. [Run SIEM Scenarios](../docs/siem-scenarios.md)

---

## Reference

- **Wazuh Windows Agent**: https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-windows.html
- **Sysmon Guide**: https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon
- **Windows Event Log IDs**: https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/

---

**Last Updated**: 7 January 2026
