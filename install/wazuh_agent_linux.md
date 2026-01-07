# Wazuh Agent Installation - Ubuntu/Linux

## Prerequisites

- Ubuntu 22.04 LTS EC2 instance
- SSH access
- Sudo privileges
- Internet access for package downloads
- Private IP of Wazuh Server: 10.0.32.10

---

## Step 1: Connect via SSH

```bash
# SSH into Linux Client
ssh -i your-key.pem ubuntu@<LINUX_CLIENT_PUBLIC_IP>

# Example:
ssh -i siem-lab.pem ubuntu@3.230.21.137

# Once connected, verify:
uname -a
# Should show: Ubuntu 22.04 LTS or similar
```

---

## Step 2: Access Wazuh Dashboard

### From Your Local Machine

1. **Open Web Browser**

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

## Step 3: Get Linux Agent Installation Command

### In Wazuh Dashboard

1. **Select Operating System**

   - Choose **Linux**

2. **Agent Configuration**

   ```
   Manager IP: 10.0.32.10
   Agent Name: LINUX-CLIENT
   Agent Group: default
   ```

3. **Copy the Installation Commands**
   (Dashboard will show bash commands)

---

## Step 4: Install Wazuh Agent on Linux

### SSH into Linux Client

```bash
# The dashboard provides a command like this:
# (Exact command will be shown in your dashboard)

# Typically:
curl -sO https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent-4.7.5-1_amd64.deb
sudo dpkg -i wazuh-agent-4.7.5-1_amd64.deb

# Configure the manager
sudo systemctl edit wazuh-agent --full

# Or manually edit:
sudo nano /var/ossec/etc/ossec.conf
```

---

## Step 5: Configure Agent Manager Address

### Edit ossec.conf

```bash
sudo nano /var/ossec/etc/ossec.conf
```

**Find and modify the manager section**:

```xml
<!-- Manager section (usually near line 50-60) -->
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
    <crypto_method>aes</crypto_method>
</client>
```

**Important**: Use **10.0.32.10** (Private IP), not the public IP!

---

## Step 6: Configure Log Collection

### Add Log Paths in ossec.conf

```bash
sudo nano /var/ossec/etc/ossec.conf
```

**Add these sections** (find `<localfile>` section):

```xml
<!-- SSH log monitoring -->
<localfile>
    <log_format>syslog</log_format>
    <location>/var/log/auth.log</location>
</localfile>

<!-- System logs -->
<localfile>
    <log_format>syslog</log_format>
    <location>/var/log/syslog</location>
</localfile>

<!-- Sudo command monitoring -->
<localfile>
    <log_format>syslog</log_format>
    <location>/var/log/sudo.log</location>
</localfile>

<!-- Auditd logs (if available) -->
<localfile>
    <log_format>syslog</log_format>
    <location>/var/log/audit/audit.log</location>
</localfile>
```

---

## Step 7: Enable Auditd (Optional but Recommended)

### Install Auditd

```bash
# Install auditd for system call auditing
sudo apt install auditd audispd-plugins -y

# Start auditd
sudo systemctl start auditd
sudo systemctl enable auditd

# Verify
sudo systemctl status auditd
```

### Configure Auditd Rules

```bash
# Edit auditd rules
sudo nano /etc/audit/rules.d/audit.rules
```

**Add audit rules** (at the end):

```bash
# Monitor /etc/passwd
-w /etc/passwd -p wa -k passwd_changes

# Monitor /etc/sudoers
-w /etc/sudoers -p wa -k sudoers_changes

# Monitor /etc/shadow
-w /etc/shadow -p wa -k shadow_changes

# Monitor file execution
-a always,exit -F arch=b64 -S execve -k exec

# Restart auditd to apply rules
sudo systemctl restart auditd
```

---

## Step 8: Start Wazuh Agent

### Start the Agent Service

```bash
# Start Wazuh agent
sudo systemctl start wazuh-agent

# Enable on boot
sudo systemctl enable wazuh-agent

# Verify status
sudo systemctl status wazuh-agent

# Expected output:
# ● wazuh-agent.service - Wazuh Agent
#    Loaded: loaded (/etc/systemd/system/wazuh-agent.service; enabled)
#    Active: active (running) since ...
```

---

## Step 9: Verify Agent Connection

### Check Agent Logs

```bash
# View agent logs
sudo tail -f /var/ossec/logs/ossec.log

# Look for:
# 2026/01/07 16:45:32 ossec-agent: INFO: Connected to the manager at 10.0.32.10:1514
# 2026/01/07 16:45:32 ossec-agent: INFO: Valid key received from agent
```

### Check Agent Status

```bash
# Alternative: check agent status directly
sudo /var/ossec/bin/wazuh-control status

# Output should show:
# wazuh-agent is running
```

---

## Step 10: Verify in Wazuh Dashboard

### Check Dashboard

1. **Login to Wazuh Dashboard**
2. **Navigate to**: Agents Management → Summary
3. **Look for "LINUX-CLIENT"**
   - Status should be: **Active**
   - Last Seen: Recent timestamp
   - OS: Ubuntu 22.04 LTS
   - IP: 10.0.32.X

---

## Step 11: Enable FIM (File Integrity Monitoring) - Optional

### Configure FIM for Important Files

```bash
sudo nano /var/ossec/etc/ossec.conf
```

**Add FIM configuration**:

```xml
<!-- File Integrity Monitoring -->
<syscheck>
    <frequency>43200</frequency>
    <directories realtime="yes">/etc</directories>
    <directories realtime="yes">/home</directories>
    <directories realtime="yes">/root</directories>
    <directories realtime="yes">/bin</directories>
    <directories realtime="yes">/usr/bin</directories>
    <directories realtime="yes">/usr/local/bin</directories>
    <directories realtime="yes">/boot</directories>

    <!-- Exclude some paths to reduce overhead -->
    <ignore>/proc</ignore>
    <ignore>/sys</ignore>
    <ignore>/dev</ignore>
    <ignore>/tmp</ignore>
</syscheck>
```

**Restart agent**:

```bash
sudo systemctl restart wazuh-agent
```

---

## Step 12: Test Agent Communication

### Generate a Test Log Event

```bash
# Attempt failed SSH login (generates auth.log event)
ssh fakeuser@localhost

# Should fail with: Permission denied (publickey)

# Or simulate multiple attempts:
for i in {1..5}; do
  ssh fakeuser@localhost 2>&1 | grep -i permission
done
```

### Check Dashboard

After a few moments:

1. **Wazuh Dashboard** → **Threat hunting** or **Security events**
2. **Filter by Agent**: LINUX-CLIENT
3. **Look for**:
   - SSH authentication failures
   - sshd events
   - auth.log entries

---

## Troubleshooting

### Agent Not Connecting

```bash
# Check logs for errors
sudo tail -100 /var/ossec/logs/ossec.log | grep -i error

# Common issues:
# - Wrong manager IP
# - Firewall blocking port 1514
# - Agent key not registered
```

### Verify Network Connectivity

```bash
# Test connection to Wazuh manager
nc -zv 10.0.32.10 1514

# Should output: success or Connection to 10.0.32.10 port 1514 [tcp/*] succeeded!
```

### Restart Agent

```bash
# Restart the agent
sudo systemctl restart wazuh-agent

# Wait 30 seconds and check status
sleep 30
sudo systemctl status wazuh-agent
```

### Check Configuration Syntax

```bash
# Validate ossec.conf syntax
sudo /var/ossec/bin/wazuh-control verify-configuration

# Output: ok (if valid)
```

### Agent Disk/Memory Usage

```bash
# Check resource usage
ps aux | grep ossec-agent

# Should use minimal resources (< 5% CPU, < 50MB RAM)

# If high usage, check logs:
sudo du -sh /var/ossec/logs/
sudo du -sh /var/ossec/queue/
```

---

## Performance Optimization

### Adjust Collection Frequency

If system is slow, increase the notify_time:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Change:

```xml
<!-- From: -->
<notify_time>10</notify_time>

<!-- To: -->
<notify_time>30</notify_time>

<!-- Restart: -->
sudo systemctl restart wazuh-agent
```

### Disable Expensive Features

```bash
sudo nano /var/ossec/etc/ossec.conf

# Comment out if not needed:
<!-- <realtime_checks>yes</realtime_checks> -->
```

---

## Next Steps

✅ **Linux Agent is installed and connected**

Proceed to:

1. [Windows Agent Installation](wazuh_agent_windows.md)
2. [Run SIEM Scenarios](../docs/siem-scenarios.md)

---

## Reference

- **Wazuh Linux Agent**: https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-linux.html
- **Auditd Documentation**: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/security_guide/chap-system_auditing
- **FIM Configuration**: https://documentation.wazuh.com/current/user-manual/capabilities/file-integrity/index.html

---

**Last Updated**: 7 January 2026
