# Wazuh Server Installation Guide - CentOS/Ubuntu

## Prerequisites

- EC2 Instance: t3.large (minimum)
- OS: Ubuntu 22.04 LTS or CentOS 8+
- Storage: 30GB minimum
- Network: Security Group allows ports 22, 443, 1514, 1515
- Internet access for package downloads

---

## Step 1: Connect to Wazuh Server

```bash
# SSH into the instance
ssh -i your-key.pem ubuntu@<WAZUH_SERVER_PUBLIC_IP>

# Update system
sudo apt update && sudo apt -y upgrade
```

---

## Step 2: Download Wazuh Installation Script

```bash
# Download the official Wazuh installation script
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh

# Make it executable
chmod +x wazuh-install.sh
```

---

## Step 3: Install Wazuh All-in-One

```bash
# Run the installation script with all-in-one mode
sudo bash wazuh-install.sh -a

# Expected output after completion:
# INFO: Wazuh installation completed successfully
# Wazuh Dashboard URL: https://<SERVER_IP>
# Username: admin
# Password: [RANDOM_GENERATED_PASSWORD]
```

---

## Step 4: Verify Installation

### Check Services Status

```bash
# Wazuh Manager
sudo systemctl status wazuh-manager

# Wazuh Indexer (OpenSearch)
sudo systemctl status wazuh-indexer

# Wazuh Dashboard
sudo systemctl status wazuh-dashboard
```

All three services should show **active (running)**.

---

## Step 5: Retrieve Admin Credentials

```bash
# Extract the installation files
sudo tar -xf wazuh-install-files.tar

# Display the passwords
cat wazuh-install-files/wazuh-passwords.txt

# Output example:
# admin: ChBidg89A.Qwr8LhZw+2t?vJyhoJPHG
# wazuh-wui: anotherPassword123!
# indexer_admin: indexerPassword456!
```

**Save these credentials securely!**

---

## Step 6: Access Wazuh Dashboard

### Via Web Browser

1. **Open URL**

   ```
   https://<WAZUH_SERVER_PUBLIC_IP>
   ```

   (Note: HTTPS may show certificate warning - accept it)

2. **Login**

   ```
   Username: admin
   Password: [FROM wazuh-passwords.txt]
   ```

3. **First Login**
   - You may be prompted to change password (optional)
   - Explore the dashboard interface

---

## Step 7: Configure Network Access

### Security Group Verification

Ensure your AWS Security Group has these **Inbound Rules**:

| Type       | Protocol | Port | Source     |
| ---------- | -------- | ---- | ---------- |
| SSH        | TCP      | 22   | Your IP/32 |
| HTTPS      | TCP      | 443  | Your IP/32 |
| Custom TCP | TCP      | 1514 | sg-clients |
| Custom TCP | TCP      | 1515 | sg-clients |

### Test Connectivity

```bash
# From your local machine, test dashboard access
curl -k https://<WAZUH_SERVER_PUBLIC_IP>
# Should return HTML response (not connection refused)

# Test agent port from client
nc -zv <WAZUH_SERVER_PRIVATE_IP> 1514
nc -zv <WAZUH_SERVER_PRIVATE_IP> 1515
```

---

## Step 8: Configure Wazuh for Agents

### Enable Auto-enrollment (Recommended)

1. **SSH into Wazuh Server**

   ```bash
   sudo nano /var/ossec/etc/ossec.conf
   ```

2. **Find the `<auto_enrollment>` section** and ensure:

   ```xml
   <auto_enrollment>
       <enabled>yes</enabled>
       <manager_address>10.0.32.10</manager_address>
       <port>1515</port>
       <groups>default</groups>
   </auto_enrollment>
   ```

3. **Restart Wazuh Manager**
   ```bash
   sudo systemctl restart wazuh-manager
   ```

---

## Step 9: Agent Deployment via Dashboard

### Prepare Agent Enrollment

1. **Login to Wazuh Dashboard**

   - Navigate to: **Agents Management** → **Summary**

2. **Click "Deploy new agent"**

3. **Select Operating System**

   - For Linux: Select "Linux"
   - For Windows: Select "Windows"

4. **Agent Configuration**

   - Manager IP: 10.0.32.10 (Private IP of Wazuh Server)
   - Agent Group: default
   - Agent Name: LINUX-CLIENT or WINDOWS-CLIENT

5. **Copy the provided installation commands**
   (Use these on the respective client)

---

## Step 10: Firewall Rules (UFW on Ubuntu)

If UFW is enabled, add rules:

```bash
# Allow SSH
sudo ufw allow 22/tcp

# Allow HTTPS
sudo ufw allow 443/tcp

# Allow agent communication
sudo ufw allow 1514/tcp
sudo ufw allow 1515/tcp

# Enable firewall
sudo ufw enable
```

---

## Troubleshooting

### Service Not Starting

```bash
# Check logs
sudo tail -f /var/ossec/logs/ossec.log

# Restart services
sudo systemctl restart wazuh-manager
sudo systemctl restart wazuh-indexer
sudo systemctl restart wazuh-dashboard
```

### Port Already in Use

```bash
# Check what's using port 1514
sudo lsof -i :1514

# If needed, kill the process and restart
sudo systemctl restart wazuh-manager
```

### High Disk Usage

```bash
# Check disk space
df -h

# Clean old logs if necessary
sudo find /var/ossec/logs -name "*.log.*" -delete
```

### Dashboard Not Accessible

```bash
# Check if port 443 is open
sudo netstat -tlnp | grep 443

# Verify Security Group allows your IP on port 443
# (Check AWS Console → EC2 → Security Groups)
```

---

## Next Steps

✅ **Wazuh Server is installed and running**

Proceed to:

1. [Deploy Linux Agent](wazuh_agent_linux.md)
2. [Deploy Windows Agent](wazuh_agent_windows.md)

---

## Reference

- **Wazuh Official Docs**: https://documentation.wazuh.com/
- **OpenSearch Documentation**: https://opensearch.org/docs/
- **Wazuh API**: https://documentation.wazuh.com/current/user-manual/api/

---

**Last Updated**: 7 January 2026
