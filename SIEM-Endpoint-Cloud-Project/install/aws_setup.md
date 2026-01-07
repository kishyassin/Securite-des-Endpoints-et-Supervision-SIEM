# AWS Setup Guide - SIEM Lab Infrastructure

## Prerequisite

- AWS Learner Lab Account
- Basic AWS knowledge
- SSH client (PuTTY, OpenSSH)

---

## Step 1: Create VPC

### Via AWS Console

1. **Navigate to VPC**

   - Services → VPC → Your VPCs → Create VPC

2. **VPC Settings**

   ```
   Name: SIEM-Lab-VPC
   IPv4 CIDR block: 10.0.0.0/16
   IPv6 CIDR block: No IPv6 CIDR block
   Tenancy: Default
   ```

3. **Click Create VPC**

---

## Step 2: Create Subnet

1. **Navigate to Subnets**

   - VPC → Subnets → Create Subnet

2. **Subnet Settings**

   ```
   VPC ID: vpc-059722f94a2b45028 (SIEM-Lab-VPC-vpc)
   Subnet name: SIEM-Lab-Subnet
   Availability Zone: No preference
   IPv4 CIDR block: 10.0.32.0/24
   ```

3. **Click Create Subnet**

4. **Enable Auto-assign Public IPv4**
   - Select subnet → Actions → Edit subnet settings
   - ✅ Enable auto-assign public IPv4 address
   - Save

---

## Step 3: Create Internet Gateway

1. **Navigate to Internet Gateways**

   - VPC → Internet Gateways → Create Internet Gateway

2. **Settings**

   ```
   Name: SIEM-Lab-IGW
   ```

3. **Attach to VPC**
   - Select the created IGW → Actions → Attach to VPC
   - Select: SIEM-Lab-VPC
   - Click Attach Internet Gateway

---

## Step 4: Create Route Table

1. **Navigate to Route Tables**

   - VPC → Route Tables → Create Route Table

2. **Settings**

   ```
   Name: SIEM-Lab-RT
   VPC: SIEM-Lab-VPC
   ```

3. **Add Route**

   - Select Route Table → Routes tab → Edit Routes → Add Route

   ```
   Destination: 0.0.0.0/0
   Target: Internet Gateway → SIEM-Lab-IGW
   ```

   - Save Routes

4. **Associate with Subnet**
   - Subnet Associations tab → Edit Subnet Associations
   - Select: SIEM-Lab-Subnet
   - Click Save

---

## Step 5: Create Security Groups

### Wazuh-Server SG

1. **Create Security Group**

   ```
   Name: SIEM-Lab-Wazuh-SG
   Description: Security Group for Wazuh Server
   VPC: SIEM-Lab-VPC
   ```

2. **Inbound Rules**
   | Protocol | Port | Source | Purpose |
   |----------|------|--------|---------|
   | TCP | 22 | YOUR_IP/32 | SSH Management |
   | TCP | 443 | YOUR_IP/32 | HTTPS Dashboard |
   | TCP | 1514 | sg-clients | Agent Communication |
   | TCP | 1515 | sg-clients | Enrollment |

3. **Outbound Rules** (Default Allow All)

### Clients SG (Linux + Windows)

1. **Create Security Group**

   ```
   Name: SIEM-Lab-Clients-SG
   Description: Security Group for Client Endpoints
   VPC: SIEM-Lab-VPC
   ```

2. **Inbound Rules**
   | Protocol | Port | Source | Purpose |
   |----------|------|--------|---------|
   | TCP | 22 | YOUR_IP/32 | SSH (Linux) |
   | TCP | 3389 | YOUR_IP/32 | RDP (Windows) |
   | TCP | 1514 | SIEM-Lab-Wazuh-SG | Return traffic |

3. **Outbound Rules**
   ```
   Allow all to Wazuh-Server SG on 1514, 1515
   Allow all HTTPS outbound (443)
   ```

---

## Step 6: Create EC2 Instances

### Instance 1: Wazuh Server

1. **Navigate to EC2 → Instances → Launch Instance**

2. **Basic Configuration**

   ```
   Name: Wazuh-Server
   OS: Ubuntu Server 22.04 LTS (x86)
   Architecture: 64-bit (x86)
   Instance Type: t3.large
   Key Pair: Select or create (save .pem file)
   ```

3. **Network Settings**

   ```
   VPC: SIEM-Lab-VPC
   Subnet: SIEM-Lab-Subnet
   Auto-assign Public IP: Enable
   Security Group: SIEM-Lab-Wazuh-SG
   ```

4. **Storage**

   ```
   Size: 30 GB (gp3)
   ```

5. **Launch Instance**
   - Wait for "Running" state
   - Note Public IP address

---

### Instance 2: Linux Client

1. **Launch Instance**

   ```
   Name: Linux-Client
   OS: Ubuntu Server 22.04 LTS (x86)
   Instance Type: t2.micro or t3.micro
   Key Pair: Same as Wazuh-Server
   ```

2. **Network Settings**

   ```
   VPC: SIEM-Lab-VPC
   Subnet: SIEM-Lab-Subnet
   Auto-assign Public IP: Enable
   Security Group: SIEM-Lab-Clients-SG
   ```

3. **Storage**

   ```
   Size: 8 GB
   ```

4. **Note Public IP** (e.g., 3.230.21.137)

---

### Instance 3: Windows Client

1. **Launch Instance**

   ```
   Name: Windows-Client
   OS: Windows Server 2025 Base
   Instance Type: t2.medium (minimum)
   Key Pair: Same as others
   ```

2. **Network Settings**

   ```
   VPC: SIEM-Lab-VPC
   Subnet: SIEM-Lab-Subnet
   Auto-assign Public IP: Enable
   Security Group: SIEM-Lab-Clients-SG
   ```

3. **Storage**

   ```
   Size: 30 GB
   ```

4. **Launch**
   - Wait for "Running" state
   - Get RDP credentials via "Connect" button

---

## Step 7: Verify Connectivity

### Test SSH to Wazuh Server

```bash
ssh -i your-key.pem ubuntu@<WAZUH_PUBLIC_IP>
```

### Test SSH to Linux Client

```bash
ssh -i your-key.pem ubuntu@<LINUX_PUBLIC_IP>
```

### Test RDP to Windows Client

```
RDP Client → <WINDOWS_PUBLIC_IP>
Username: Administrator
Password: (from EC2 Connect)
```

---

## ✅ Summary

| Resource    | Name                | Details                              |
| ----------- | ------------------- | ------------------------------------ |
| VPC         | SIEM-Lab-VPC        | 10.0.0.0/16                          |
| Subnet      | SIEM-Lab-Subnet     | 10.0.32.0/24                         |
| IGW         | SIEM-Lab-IGW        | Attached                             |
| Route Table | SIEM-Lab-RT         | 0.0.0.0/0 → IGW                      |
| SG Wazuh    | SIEM-Lab-Wazuh-SG   | Ports: 22, 443, 1514, 1515           |
| SG Clients  | SIEM-Lab-Clients-SG | Ports: 22, 3389                      |
| EC2-1       | Wazuh-Server        | t3.large, Ubuntu 22.04, 30GB         |
| EC2-2       | Linux-Client        | t2.micro, Ubuntu 22.04, 8GB          |
| EC2-3       | Windows-Client      | t2.medium, Windows Server 2025, 30GB |

---

**Infrastructure is ready for Wazuh deployment!**
