# AWS NAT Instance Setup 
**Note:** This setup is in Amazon Linux 2023, but you can use any other 'os' but the networking forward commands might be differnet.


This document explains how to configure an **EC2 NAT Instance** using **Amazon Linux 2023** so that instances in a **private subnet** can access the internet.

---

## 1. Architecture Overview

- **VPC CIDR**: 10.0.0.0/16
- **Public Subnet**: 10.0.1.0/24
  - NAT EC2 Instance (10.0.1.72)
  - Internet Gateway attached to VPC
- **Private Subnet**: 10.0.2.0/24
  - Private EC2 Instance (10.0.2.5)

Traffic Flow:

Private EC2 → VPC Router → NAT EC2 → Internet Gateway → Internet

---

## 2. NAT Instance Requirements

### EC2 Configuration
- AMI: Amazon Linux 2023
- Subnet: Public Subnet
- Public IP / Elastic IP: Required

### EC2 Settings
- **Source/Destination Check**: DISABLED

---

## 3. Route Table Configuration

### Public Subnet Route Table

| Destination | Target |
|------------|--------|
| 10.0.0.0/16 | local |
| 0.0.0.0/0 | Internet Gateway |

### Private Subnet Route Table

| Destination | Target |
|------------|--------|
| 10.0.0.0/16 | local |
| 0.0.0.0/0 | NAT Instance ID |

---

## 4. Security Group Configuration

### NAT Instance Security Group

**Inbound Rules**
- SSH (22) → Your IP
- All Traffic → Private Subnet CIDR (10.0.2.0/24)

**Outbound Rules**
- All traffic → 0.0.0.0/0

### Private Instance Security Group

**Outbound Rules**
- All traffic → 0.0.0.0/0

---

## 5. NAT Instance OS Configuration

### 5.1 Enable IP Forwarding

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Persist setting:
```bash
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

---

### 5.2 Install iptables

```bash
sudo dnf install iptables iptables-services -y
```

---

### 5.3 Configure NAT (MASQUERADE)

Identify network interface (example: `enX0`):
```bash
ip a
```

Add NAT rule:
```bash
sudo iptables -t nat -A POSTROUTING -o enX0 -j MASQUERADE
```

---

### 5.4 Fix FORWARD Chain (CRITICAL FOR AL2023)

Amazon Linux 2023 includes a default **REJECT** rule in the FORWARD chain.

#### Remove REJECT rule:
```bash
sudo iptables -D FORWARD -j REJECT --reject-with icmp-host-prohibited
```

(If needed, delete by rule number using `iptables -L FORWARD --line-numbers`.)

#### Allow forwarding:
```bash
sudo iptables -P FORWARD ACCEPT
sudo iptables -A FORWARD -i enX0 -o enX0 -j ACCEPT
sudo iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
```

---

### 5.5 Save Rules

```bash
sudo systemctl enable iptables
sudo service iptables save
```

---

## 6. Verification

### On NAT Instance
```bash
ping google.com
```

### On Private Instance
```bash
ping 8.8.8.8
curl google.com
```

Expected result: Successful replies and HTML output.

---

## 7. Common Errors & Fixes

| Error | Cause | Fix |
|------|------|-----|
| Destination Host Prohibited | FORWARD REJECT rule | Remove REJECT + allow FORWARD |
| No internet from private | Wrong route table | Point 0.0.0.0/0 to NAT instance |
| NAT has no internet | Missing IGW route | Add IGW to public subnet |
| Traffic dropped | Source/Dest check enabled | Disable it on NAT EC2 |

---

## 8. Best Practices

- NAT Instance is suitable for:
  - Labs
  - Dev/Test
  - Cost-sensitive environments

- For production:
  - Use NAT Gateway
  - High availability
  - No OS firewall maintenance

---

## 9. Summary

You successfully configured a working NAT Instance on Amazon Linux 2023 by:
- Correct routing
- Proper security groups
- Enabling IP forwarding
- Configuring MASQUERADE
- Removing default FORWARD REJECT rule

This setup enables secure outbound internet access for private subnet instances.

---

**End of Document**

