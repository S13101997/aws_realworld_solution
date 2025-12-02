# Security Groups & Network ACLs — Firewall Configurations and Network Access Controls

**Complete Guide with Detailed Examples and Top 5 Interview Questions**

---

## Table of Contents

1. [Overview](#overview)
2. [Security Groups (SG)](#security-groups)
3. [Network ACLs (NACLs)](#network-acls)
4. [Comparison & Differences](#comparison--differences)
5. [Real-World Examples](#real-world-examples)
6. [Best Practices](#best-practices)
7. [Troubleshooting](#troubleshooting)
8. [Top 5 Interview Questions](#top-5-interview-questions)

---

## Overview

AWS provides two layers of virtual firewalls to control network traffic within a VPC:

**Security Groups (SG)** - Instance/ENI-level stateful firewall
**Network ACLs (NACLs)** - Subnet-level stateless firewall

Together, they form a defense-in-depth approach to network security.

### Network Traffic Flow (Inbound)

```
Internet Traffic
    ↓
Internet Gateway (IGW)
    ↓
VPC Route Table
    ↓
Subnet (Network ACL - Layer 1)
    ↓
ENI (Elastic Network Interface)
    ↓
EC2 Instance Security Group (Layer 2)
    ↓
Application
```

### Key Principle: Layered Security

Both layers must allow traffic for communication to succeed:
- **NACL blocks** → Traffic blocked (doesn't reach SG)
- **NACL allows** → Traffic passes to SG
- **SG blocks** → Traffic blocked (even if NACL allows)
- **Both allow** → Traffic reaches application ✓

---

## Security Groups

### Definition & Characteristics

A **Security Group** is a stateful virtual firewall that controls inbound and outbound traffic at the EC2 instance/ENI (Elastic Network Interface) level.

**Key Characteristics:**
- **Scope:** Instance/ENI level (Elastic Network Interface)
- **Statefulness:** Stateful (connection-aware)
- **Rule Type:** Allow rules only (implicit deny)
- **Rule Order:** All rules evaluated simultaneously (no priority)
- **Default Behavior:** Deny all inbound, Allow all outbound
- **Can Associate:** Multiple SGs per instance
- **Can Reference:** Other security groups and CIDR blocks

### How Security Groups Work

**Stateful Behavior Example:**

```
Client → Server (Port 80)

Step 1: Outbound from client
- Client SG outbound rule: Allow 0.0.0.0/0 on port 80
- Traffic allowed to server

Step 2: Server receives inbound
- Server SG inbound rule: Allow 0.0.0.0/0 on port 80
- Traffic allowed

Step 3: Server → Client (response)
- No explicit outbound rule needed on server
- Stateful firewall automatically allows response
- Response sent back to client

Step 4: Client receives response
- No explicit inbound rule needed on client
- Stateful firewall automatically allows response
- Connection completes
```

### Security Group Rules

#### Rule Components

```
Inbound Rule Structure:
┌─────────────────────────────────────────────┐
│ Type | Protocol | Port Range | Source      │
├─────────────────────────────────────────────┤
│ HTTP | TCP      | 80         | 0.0.0.0/0   │
│ SSH  | TCP      | 22         | 10.0.0.0/8  │
│ RDP  | TCP      | 3389       | 203.0.113.0 │
└─────────────────────────────────────────────┘

Outbound Rule Structure:
┌──────────────────────────────────────────────┐
│ Type | Protocol | Port Range | Destination │
├──────────────────────────────────────────────┤
│ HTTPS| TCP      | 443        | 0.0.0.0/0    │
│ DNS  | UDP      | 53         | 0.0.0.0/0    │
│ NTP  | UDP      | 123        | 0.0.0.0/0    │
└──────────────────────────────────────────────┘
```

#### Rule Options

**Source/Destination Options:**

```
1. CIDR Block
   - 0.0.0.0/0 (anywhere IPv4)
   - 10.0.0.0/8 (private network)
   - 203.0.113.42/32 (single IP)

2. Security Group
   - sg-12345678 (another SG)
   - Enables dynamic instance-to-instance communication

3. Prefix List
   - Amazon S3 prefix list (pl-12345678)
   - Service-specific traffic control

4. IPv6 CIDR
   - ::/0 (anywhere IPv6)
   - 2001:db8::/32 (IPv6 CIDR)
```

### Creating Security Groups

#### Using AWS CLI

```bash
# Create security group for web servers
aws ec2 create-security-group \
  --group-name "web-servers" \
  --description "Security group for web tier" \
  --vpc-id vpc-12345678

# Response:
# {
#   "GroupId": "sg-0a1b2c3d4e5f6g7h8",
#   "GroupName": "web-servers"
# }

WEB_SG="sg-0a1b2c3d4e5f6g7h8"

# Add inbound rule: Allow HTTP from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id $WEB_SG \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# Add inbound rule: Allow HTTPS from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id $WEB_SG \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# Add inbound rule: Allow SSH from admin network
aws ec2 authorize-security-group-ingress \
  --group-id $WEB_SG \
  --protocol tcp \
  --port 22 \
  --cidr 10.0.1.0/24

# Add outbound rule: Allow outbound HTTPS
aws ec2 authorize-security-group-egress \
  --group-id $WEB_SG \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# List rules
aws ec2 describe-security-groups \
  --group-ids $WEB_SG \
  --query 'SecurityGroups[0].IpPermissions' \
  --output table
```

### Referencing Other Security Groups

**Dynamic Instance-to-Instance Communication:**

```bash
# Create app server security group
APP_SG=$(aws ec2 create-security-group \
  --group-name "app-servers" \
  --description "App tier security group" \
  --vpc-id vpc-12345678 \
  --query 'GroupId' \
  --output text)

# Create database security group
DB_SG=$(aws ec2 create-security-group \
  --group-name "databases" \
  --description "Database tier security group" \
  --vpc-id vpc-12345678 \
  --query 'GroupId' \
  --output text)

# Allow app servers to communicate with database
# (All instances in APP_SG can reach DB_SG on port 3306)
aws ec2 authorize-security-group-ingress \
  --group-id $DB_SG \
  --protocol tcp \
  --port 3306 \
  --source-group $APP_SG

# Benefit: Any new instance added to APP_SG automatically gains access
# No need to add new IP addresses to DB_SG rules
```

---

## Network ACLs

### Definition & Characteristics

A **Network ACL** is a stateless, optional layer of security at the subnet level that acts as a firewall for controlling traffic to and from subnets.

**Key Characteristics:**
- **Scope:** Subnet level (all instances in subnet)
- **Statefulness:** Stateless (connection-unaware)
- **Rule Type:** Allow and Deny rules
- **Rule Order:** Processed by rule number (lowest first)
- **Default NACL:** Allows all traffic
- **Can Associate:** One NACL per subnet (subnet can have only one)
- **Can Reference:** CIDR blocks only (not security groups)

### How Network ACLs Work

**Stateless Behavior Example:**

```
Client → Server (Port 80) - Request

Step 1: Outbound from client
- Client NACL outbound rule #100: Allow TCP 80 to 0.0.0.0/0
- Traffic allowed

Step 2: Server receives inbound
- Server NACL inbound rule #100: Allow TCP 80 from 0.0.0.0/0
- Traffic allowed

Step 3: Server → Client (response)
- Must have explicit outbound rule
- Server NACL outbound rule #100: Allow TCP 80 to 0.0.0.0/0
- Or: Allow ephemeral ports (1024-65535) to client

Step 4: Client receives response
- Must have explicit inbound rule
- Client NACL inbound rule #100: Allow TCP 80 from 0.0.0.0/0
- Or: Allow ephemeral ports (1024-65535) from server
- Connection completes

⚠️ Common Mistake: Forgetting ephemeral port ranges
For stateless firewalls, responses need explicit inbound/outbound rules
```

### Network ACL Rules

#### Rule Structure

```
NACL Rule Components:
┌────────────────────────────────────────────────┐
│ # | Type | Protocol | Port Range | CIDR | Action │
├────────────────────────────────────────────────┤
│100| HTTP | TCP      | 80         | 0/0  | Allow  │
│110| HTTPS| TCP      | 443        | 0/0  | Allow  │
│120| SSH  | TCP      | 22         | 10/8 | Allow  │
│130| -    | TCP      | 1024-65535 | 0/0  | Allow  │
│140| -    | -        | -          | 0/0  | Deny   │
└────────────────────────────────────────────────┘

Rule #140 is the default deny-all rule (implicit)
```

#### Ephemeral Ports

**What are ephemeral ports?**

When a client initiates a connection, the OS assigns a random port from the ephemeral range (1024-65535) for the return traffic.

**Example HTTP Request:**

```
Client Port 54321 → Server Port 80 (client assigns 54321)
Server Port 80 → Client Port 54321 (response)

Client NACL must allow inbound on port 54321
Server NACL must allow outbound on port 54321

Or use range: 1024-65535 to cover all ephemeral ports
```

**Default Ephemeral Port Ranges by OS:**

```
Windows:      49152-65535
Linux/Unix:   32768-65535
```

### Creating and Modifying Network ACLs

#### Using AWS CLI

```bash
# Create custom NACL
NACL_ID=$(aws ec2 create-network-acl \
  --vpc-id vpc-12345678 \
  --query 'NetworkAcl.NetworkAclId' \
  --output text)

echo "NACL ID: $NACL_ID"

# Add inbound HTTP rule
aws ec2 create-network-acl-entry \
  --network-acl-id $NACL_ID \
  --rule-number 100 \
  --protocol tcp \
  --port-range From=80,To=80 \
  --cidr-block 0.0.0.0/0 \
  --ingress

# Add inbound HTTPS rule
aws ec2 create-network-acl-entry \
  --network-acl-id $NACL_ID \
  --rule-number 110 \
  --protocol tcp \
  --port-range From=443,To=443 \
  --cidr-block 0.0.0.0/0 \
  --ingress

# Add inbound SSH (restricted to admin network)
aws ec2 create-network-acl-entry \
  --network-acl-id $NACL_ID \
  --rule-number 120 \
  --protocol tcp \
  --port-range From=22,To=22 \
  --cidr-block 10.0.1.0/24 \
  --ingress

# Add inbound ephemeral ports (for response traffic)
aws ec2 create-network-acl-entry \
  --network-acl-id $NACL_ID \
  --rule-number 130 \
  --protocol tcp \
  --port-range From=1024,To=65535 \
  --cidr-block 0.0.0.0/0 \
  --ingress

# Add outbound HTTP
aws ec2 create-network-acl-entry \
  --network-acl-id $NACL_ID \
  --rule-number 100 \
  --protocol tcp \
  --port-range From=80,To=80 \
  --cidr-block 0.0.0.0/0 \
  --egress

# Add outbound HTTPS
aws ec2 create-network-acl-entry \
  --network-acl-id $NACL_ID \
  --rule-number 110 \
  --protocol tcp \
  --port-range From=443,To=443 \
  --cidr-block 0.0.0.0/0 \
  --egress

# Add outbound ephemeral ports
aws ec2 create-network-acl-entry \
  --network-acl-id $NACL_ID \
  --rule-number 120 \
  --protocol tcp \
  --port-range From=1024,To=65535 \
  --cidr-block 0.0.0.0/0 \
  --egress

# Associate NACL with subnet
aws ec2 associate-network-acl \
  --network-acl-id $NACL_ID \
  --subnet-id subnet-12345678

# List NACL rules
aws ec2 describe-network-acls \
  --network-acl-ids $NACL_ID \
  --query 'NetworkAcls[0].Entries' \
  --output table
```

#### Denying Traffic

**Blocking a Malicious IP:**

```bash
# Block traffic from specific IP (malicious attacker)
aws ec2 create-network-acl-entry \
  --network-acl-id $NACL_ID \
  --rule-number 50 \
  --protocol -1 \
  --cidr-block 203.0.113.42/32 \
  --ingress \
  --egress-rule false

# ⚠️ Rule 50 is evaluated before rule 100 (lower number = higher priority)
# This blocks all traffic from 203.0.113.42 before allowing rules
```

---

## Comparison & Differences

### Side-by-Side Comparison Table

```
┌──────────────────────────┬──────────────────────┬──────────────────┐
│ Feature                  │ Security Group (SG)  │ Network ACL      │
├──────────────────────────┼──────────────────────┼──────────────────┤
│ Scope Level              │ Instance/ENI         │ Subnet           │
│ Statefulness             │ Stateful             │ Stateless        │
│ Rule Actions             │ Allow only           │ Allow & Deny     │
│ Rule Evaluation          │ All evaluated        │ By rule number   │
│ Default Inbound          │ Deny all             │ Allow all        │
│ Default Outbound         │ Allow all            │ Allow all        │
│ Instances per SG         │ Multiple             │ N/A              │
│ SG per Instance          │ Multiple             │ N/A              │
│ Subnets per NACL         │ N/A                  │ Single           │
│ NACL per Subnet          │ N/A                  │ Single           │
│ Source/Dest Options      │ CIDR, SG, Prefix List│ CIDR only        │
│ Same-Subnet Traffic      │ Filtered             │ Not filtered     │
│ Cross-Subnet Traffic     │ Filtered             │ Filtered         │
│ Return Traffic Required  │ No (stateful)        │ Yes (stateless)  │
│ Performance Impact       │ Minimal              │ Minimal          │
│ Use Case                 │ Fine-grained control │ Broad subnet     │
│                          │ Application-level    │ control          │
└──────────────────────────┴──────────────────────┴──────────────────┘
```

### Stateful vs Stateless Example

**Scenario: Client sends HTTP request to web server**

**Security Group (Stateful):**
```
Inbound Rule:  Allow TCP port 80 from 0.0.0.0/0
Outbound Rule: Allow all traffic to 0.0.0.0/0

Flow:
1. Client sends SYN on port 80 → Inbound rule allows
2. Server responds on ephemeral port → Automatically allowed (stateful)
3. Return data flows without explicit rule → Works! ✓

Why: SG remembers connection state
```

**Network ACL (Stateless):**
```
Inbound Rules:
  #100 Allow TCP port 80 from 0.0.0.0/0
  #110 Allow ephemeral ports 1024-65535 from 0.0.0.0/0

Outbound Rules:
  #100 Allow TCP port 80 to 0.0.0.0/0
  #110 Allow ephemeral ports 1024-65535 to 0.0.0.0/0

Flow:
1. Client sends SYN on port 80 → Inbound rule #100 allows
2. Server responds on ephemeral port → Inbound rule #110 allows (explicit)
3. Return data flows with matching rule → Works! ✓

Why: Each direction needs explicit rules
```

---

## Real-World Examples

### Example 1: Three-Tier Web Application

**Architecture:**

```
Internet
    ↓
ALB (Application Load Balancer)
    ↓
┌─────────────────────────────────────────┐
│ Public Subnet (Bastion/NAT)             │
│ ├─ Bastion Security Group               │
│ └─ NACL: Allow SSH, Ephemeral           │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│ Private Subnet (Web Tier)               │
│ ├─ Web Servers Security Group           │
│ └─ NACL: Allow HTTP/HTTPS, Ephemeral   │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│ Private Subnet (App Tier)               │
│ ├─ App Servers Security Group           │
│ └─ NACL: Allow 8080, Ephemeral         │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│ Private Subnet (Database Tier)          │
│ ├─ Database Security Group              │
│ └─ NACL: Allow 3306, Ephemeral         │
└─────────────────────────────────────────┘
```

**Configuration:**

```bash
#!/bin/bash

# Create Security Groups
BASTION_SG=$(aws ec2 create-security-group \
  --group-name "bastion-sg" \
  --description "Bastion host" \
  --vpc-id vpc-12345678 \
  --query 'GroupId' --output text)

WEB_SG=$(aws ec2 create-security-group \
  --group-name "web-sg" \
  --description "Web servers" \
  --vpc-id vpc-12345678 \
  --query 'GroupId' --output text)

APP_SG=$(aws ec2 create-security-group \
  --group-name "app-sg" \
  --description "App servers" \
  --vpc-id vpc-12345678 \
  --query 'GroupId' --output text)

DB_SG=$(aws ec2 create-security-group \
  --group-name "db-sg" \
  --description "Databases" \
  --vpc-id vpc-12345678 \
  --query 'GroupId' --output text)

# Bastion: Allow SSH from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id $BASTION_SG \
  --protocol tcp --port 22 --cidr 0.0.0.0/0

# Web: Allow HTTP/HTTPS from anywhere, SSH from Bastion
aws ec2 authorize-security-group-ingress \
  --group-id $WEB_SG \
  --protocol tcp --port 80 --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-id $WEB_SG \
  --protocol tcp --port 443 --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-id $WEB_SG \
  --protocol tcp --port 22 \
  --source-group $BASTION_SG

# App: Allow port 8080 from Web tier, SSH from Bastion
aws ec2 authorize-security-group-ingress \
  --group-id $APP_SG \
  --protocol tcp --port 8080 \
  --source-group $WEB_SG

aws ec2 authorize-security-group-ingress \
  --group-id $APP_SG \
  --protocol tcp --port 22 \
  --source-group $BASTION_SG

# Database: Allow port 3306 from App tier, SSH from Bastion
aws ec2 authorize-security-group-ingress \
  --group-id $DB_SG \
  --protocol tcp --port 3306 \
  --source-group $APP_SG

aws ec2 authorize-security-group-ingress \
  --group-id $DB_SG \
  --protocol tcp --port 22 \
  --source-group $BASTION_SG

# Web: Allow outbound to App tier on port 8080
aws ec2 authorize-security-group-egress \
  --group-id $WEB_SG \
  --protocol tcp --port 8080 \
  --destination-group $APP_SG

# App: Allow outbound to DB on port 3306
aws ec2 authorize-security-group-egress \
  --group-id $APP_SG \
  --protocol tcp --port 3306 \
  --destination-group $DB_SG

echo "✓ Security Groups configured"
```

### Example 2: Blocking a Known Attacker with NACL

**Scenario: DDoS attack from 203.0.113.0/24**

```bash
# Create NACL to block attacker
NACL_ID=$(aws ec2 create-network-acl \
  --vpc-id vpc-12345678 \
  --query 'NetworkAcl.NetworkAclId' \
  --output text)

# Deny all from attacker (rule 10 - highest priority)
aws ec2 create-network-acl-entry \
  --network-acl-id $NACL_ID \
  --rule-number 10 \
  --protocol -1 \
  --cidr-block 203.0.113.0/24 \
  --ingress

# Allow normal traffic (rule 100)
aws ec2 create-network-acl-entry \
  --network-acl-id $NACL_ID \
  --rule-number 100 \
  --protocol -1 \
  --cidr-block 0.0.0.0/0 \
  --ingress

# Apply to vulnerable subnet
aws ec2 associate-network-acl \
  --network-acl-id $NACL_ID \
  --subnet-id subnet-12345678

# Verification
aws ec2 describe-network-acls \
  --network-acl-ids $NACL_ID \
  --query 'NetworkAcls[0].Entries' \
  --output table

# Result: Rule 10 evaluated first, blocks attacker before rule 100
```

### Example 3: Same-Subnet Communication (Subnet Isolation)

**Scenario: Two EC2 instances in same subnet need different access levels**

**Instance A (Web Server):** Accept public traffic
**Instance B (Database):** Accept only from Instance A

**Configuration:**

```bash
# Create Security Groups
WEB_SG=$(aws ec2 create-security-group \
  --group-name "web" --vpc-id vpc-12345678 \
  --query 'GroupId' --output text)

DB_SG=$(aws ec2 create-security-group \
  --group-name "db" --vpc-id vpc-12345678 \
  --query 'GroupId' --output text)

# Web: Accept HTTP from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id $WEB_SG \
  --protocol tcp --port 80 --cidr 0.0.0.0/0

# DB: Accept MySQL only from Web SG
aws ec2 authorize-security-group-ingress \
  --group-id $DB_SG \
  --protocol tcp --port 3306 \
  --source-group $WEB_SG

# Key Point: NACLs do NOT filter same-subnet traffic
# Even with NACL deny rules, SG filtering still applies
# SGs provide fine-grained control within subnet
```

---

## Best Practices

### Security Group Best Practices

✓ **DO:**
- Use principle of least privilege (allow minimum required)
- Create security groups with descriptive names
- Document purpose of each rule
- Use security group references for dynamic access
- Regularly audit security group rules
- Remove unused security groups
- Use separate SGs for different tiers
- Use default deny inbound, allow outbound approach
- Monitor security group changes with CloudTrail

✗ **DON'T:**
- Allow 0.0.0.0/0 on SSH (port 22) without justification
- Allow 0.0.0.0/0 on RDP (port 3389) to databases
- Use overly permissive outbound rules
- Mix production and development in same SG
- Leave rules without documented purpose
- Forget about stateless return traffic in NACLs
- Use security group for all access control
- Allow protocols you don't use

### Network ACL Best Practices

✓ **DO:**
- Number rules in increments of 10 or 100 (allow room for future)
- Document each rule's purpose
- Use deny rules sparingly and explicitly
- Explicitly allow ephemeral ports for return traffic
- Use lowest rule numbers for most important rules
- Test NACL rules before deployment
- Monitor NACL performance
- Apply principle of least privilege

✗ **DON'T:**
- Use sequential rule numbers (1, 2, 3...)
- Make ACL changes without testing
- Forget about stateless return traffic
- Use NACL for fine-grained access (use SG instead)
- Apply deny rules globally without reason
- Change default NACL without understanding impact
- Use NACL alone without security groups

### Layered Security Architecture

**Recommended Setup:**

```
┌─────────────────────────────────────────┐
│ Network Layer (NACL)                    │
│ - Broad subnet-level filtering          │
│ - Block known malicious IPs             │
│ - Enforce ephemeral port ranges         │
│ - VPC-level compliance controls         │
└─────────────────────────────────────────┘
            ↓
┌─────────────────────────────────────────┐
│ Instance Layer (Security Group)         │
│ - Fine-grained application access       │
│ - Reference other security groups       │
│ - Port-specific rules                   │
│ - Service-level control                 │
└─────────────────────────────────────────┘
            ↓
┌─────────────────────────────────────────┐
│ Application Layer (WAF/Firewall)        │
│ - Protocol-level filtering              │
│ - DDoS protection                       │
│ - Application-specific rules            │
│ - IP reputation blocking                │
└─────────────────────────────────────────┘
```

---

## Troubleshooting

### Problem 1: Traffic Blocked (Cannot Connect)

**Diagnosis Checklist:**

```
1. Check Security Group Inbound Rules
   aws ec2 describe-security-groups --group-ids sg-12345678

2. Check Security Group Outbound Rules
   aws ec2 describe-security-groups --group-ids sg-12345678 \
     --query 'SecurityGroups[0].IpPermissionsEgress'

3. Check NACL Inbound Rules
   aws ec2 describe-network-acls --subnet-ids subnet-12345678

4. Check NACL Outbound Rules
   aws ec2 describe-network-acls --subnet-ids subnet-12345678 \
     --query 'NetworkAcls[0].Entries[?Egress==true]'

5. Verify Route Table
   aws ec2 describe-route-tables --route-table-ids rtb-12345678

6. Check Security Group References
   Does SG reference another SG? Verify both exist.

7. Test Connectivity
   - From source instance: ping, telnet, or nc to target
   - Check CloudWatch logs
   - Review VPC Flow Logs
```

### Problem 2: Return Traffic Blocked (NACL)

**Scenario: Client sends request, server response blocked**

```
Client: 10.0.1.50 (ephemeral port 54321) → Server: 10.0.2.50 (port 80)

Issue: Server can send response to port 54321, but NACL blocks return

Solution: Add NACL outbound rule allowing ephemeral ports
aws ec2 create-network-acl-entry \
  --network-acl-id acl-12345678 \
  --rule-number 110 \
  --protocol tcp \
  --port-range From=1024,To=65535 \
  --cidr-block 10.0.1.0/24 \
  --egress
```

### Problem 3: Performance Issues

**Possible Causes:**

```
1. NACL Rule Evaluation Overhead
   - Large number of rules can impact performance
   - Minimize NACL rules (use SGs instead)
   - Keep critical rules at lower rule numbers

2. Security Group Evaluation
   - Reference many other SGs = slower evaluation
   - Consolidate SG rules where possible
   - Use prefix lists for AWS services

3. Instance Size
   - Large number of rules on small instance type
   - Consider larger instance
   - Offload to load balancer with WAF

Diagnosis:
aws ec2 describe-instances --instance-ids i-12345678 \
  --query 'Reservations[0].Instances[0].SecurityGroups'
```

---

## Top 5 Interview Questions

### Question 1: Fundamental Difference (Beginner)

**Question:**
"Explain the key difference between Security Groups and Network ACLs. Why do we need both?"

**Model Answer:**

**Security Groups:**
- **Scope:** Instance/ENI level
- **Statefulness:** Stateful (connection-aware)
- **Rules:** Allow only (implicit deny)
- **Default:** Deny inbound, allow outbound
- **Use:** Fine-grained, instance-level access control

**Network ACLs:**
- **Scope:** Subnet level
- **Statefulness:** Stateless (no connection state)
- **Rules:** Allow and Deny
- **Default:** Allow all (unless custom NACL)
- **Use:** Broad subnet-level filtering

**Why Both?**

**Layered Defense:**
```
SG = Instance firewall (application-level control)
NACL = Subnet firewall (network-level control)

Both must allow traffic for connection:
├─ NACL denies → Blocked (doesn't reach SG)
├─ SG denies → Blocked (even if NACL allows)
└─ Both allow → Connection succeeds
```

**Real-World Example:**
```
Scenario: Block a malicious IP (203.0.113.42)

Option 1 - Security Group Only:
- Problem: Must add rule to every SG
- Time-consuming, error-prone
- Doesn't scale

Option 2 - NACL Only:
- Add single deny rule to subnet NACL
- Blocks at subnet level for all instances
- Efficient but lacks fine-grained control

Option 3 - Both (Recommended):
- NACL: Block 203.0.113.42 at subnet level
- SG: Allow specific services to specific IPs
- Layered, scalable, auditable
```

**Why Both Matter:**
1. **Performance:** Each layer filters without over-processing
2. **Compliance:** Different regulatory requirements at different layers
3. **Scalability:** NACL for broad control, SG for individual instances
4. **Flexibility:** Different teams manage different layers
5. **Disaster Recovery:** Multiple layers survive failures

**Key Insight:**
> "Think of Security Groups as doors to each house (fine-grained), and Network ACLs as gates to the neighborhood (broad). You need both for complete protection."

---

### Question 2: Stateful vs Stateless (Intermediate)

**Question:**
"Explain why Network ACLs are stateless while Security Groups are stateful. What impact does this have on firewall rules?"

**Model Answer:**

**Stateful (Security Groups):**

Maintains connection state and remembers bidirectional traffic flow.

```
Connection Flow:
┌──────────┐              ┌──────────┐
│  Client  │              │  Server  │
│ Port 80  │              │ Port 443 │
└──────────┘              └──────────┘
     │                           │
     │─────── Request ─────────→ │  (1) Inbound rule matches
     │                           │
     │ ← ─ ─ Response ← ─ ─ ─ ─ │  (2) Automatically allowed
     │                           │      (remembered connection)

Client SG:
  Inbound: Allow 0.0.0.0/0 on 80 (not needed for response!)
  Outbound: Allow 0.0.0.0/0 (covers request)

Server SG:
  Inbound: Allow 0.0.0.0/0 on 443 (accepts request)
  Outbound: Allow 0.0.0.0/0 (response automatically allowed)

Why Works: SG remembers request-response pair
```

**Stateless (Network ACLs):**

No connection tracking; each direction requires explicit rules.

```
Connection Flow:
┌──────────┐              ┌──────────┐
│  Client  │              │  Server  │
│ Port 80  │              │ Port 443 │
└──────────┘              └──────────┘
     │                           │
     │─────── Request ─────────→ │  (1) Outbound rule matches
     │  (ephemeral 54321)        │
     │                           │
     │ ← ─ ─ Response ← ─ ─ ─ ─ │  (2) Inbound rule must match
     │  (from port 443)          │      ephemeral port 54321

Client NACL:
  Outbound: Allow TCP 443 to 0.0.0.0/0
  Inbound: Allow TCP 1024-65535 from 0.0.0.0/0 ← REQUIRED!

Server NACL:
  Inbound: Allow TCP 443 from 0.0.0.0/0
  Outbound: Allow TCP 1024-65535 to 0.0.0.0/0 ← REQUIRED!

Why Needed: NACL doesn't track connections
```

**Side-by-Side Comparison:**

```
┌──────────────────────────┬──────────────────┬──────────────────┐
│ Scenario                 │ Security Group   │ NACL             │
├──────────────────────────┼──────────────────┼──────────────────┤
│ Allow inbound port 443   │ Rule: Allow 443  │ Rule: Allow 443  │
│ Response traffic allowed?│ YES (automatic)  │ NO (explicit req)│
│                          │                  │ Need ephemeral   │
│                          │                  │ ports rule       │
├──────────────────────────┼──────────────────┼──────────────────┤
│ Connection memory        │ Yes (state table)│ No (stateless)   │
├──────────────────────────┼──────────────────┼──────────────────┤
│ Rules needed for 1 flow  │ ~1-2 per tier    │ ~2-4 per tier    │
│                          │ (bidirectional   │ (each direction  │
│                          │ auto-allowed)    │ explicit)        │
├──────────────────────────┼──────────────────┼──────────────────┤
│ Rule complexity          │ Simple           │ Complex          │
│                          │                  │ (ephemeral)      │
└──────────────────────────┴──────────────────┴──────────────────┘
```

**Impact on Firewall Design:**

1. **Rule Count:** NACLs require ~2x rules for same policy
2. **Configuration:** NACL ephemeral ports (1024-65535) is common default
3. **Troubleshooting:** NACL failures harder to diagnose (no connection state)
4. **Performance:** SG more efficient for stateful filtering
5. **Use Case:** NACL good for blocking entire subnets, SG for instances

**Real Example - Database Access:**

```
Stateful (Security Group):
App SG → DB SG on port 3306
Rule: Allow 3306 from app-sg
Return traffic: Automatic ✓

Stateless (NACL):
App NACL → DB NACL on port 3306
Inbound: Allow 3306 from app subnet
Outbound: Allow 1024-65535 to app subnet (for response)
DB Inbound: Allow 3306 from app subnet
DB Outbound: Allow 1024-65535 to app subnet
Return traffic: Requires all rules ✓
```

---

### Question 3: Real-World Scenario (Intermediate-Advanced)

**Question:**
"A company has a web server in subnet A and database in subnet B. Developers can SSH to both from subnet C (bastion). Design firewall rules. Why would NACLs fail if you only allow port 3306?"

**Model Answer:**

**Architecture:**

```
┌─────────────────────────────────────────────────────┐
│ VPC: 10.0.0.0/16                                   │
├─────────────────────────────────────────────────────┤
│                                                     │
│ Subnet A (Web)          Subnet B (DB)  Subnet C    │
│ 10.0.1.0/24             10.0.2.0/24    (Bastion)   │
│                                                     │
│ ┌────────────────┐  ┌─────────────┐  ┌─────────┐   │
│ │ Web Server     │  │ DB Server   │  │ Bastion │   │
│ │ sg-web         │  │ sg-db       │  │ sg-bas  │   │
│ └────────────────┘  └─────────────┘  └─────────┘   │
│         │                   │              │        │
│      NACL A            NACL B         NACL C        │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Required Connectivity:**

```
1. Bastion → Web: SSH (port 22)
2. Bastion → DB: SSH (port 22)
3. Web → DB: MySQL (port 3306)
4. Web ↔ Internet: HTTP/HTTPS (80, 443)
5. Return traffic for all above
```

**NAIVE NACL Configuration (INCORRECT):**

```
Subnet B (DB) NACL:
┌─────────┬──────────┬─────┬────────────┬────────┐
│ # | Dir │ Protocol │ Port│ CIDR       │ Action │
├─────────┼──────────┼─────┼────────────┼────────┤
│100│In   │ TCP      │3306 │10.0.1.0/24 │ Allow  │
│110│In   │ TCP      │ 22  │10.0.3.0/24 │ Allow  │
│120│In   │ TCP      │ -   │ 0.0.0.0/0  │ Deny   │
│  │      │          │     │            │        │
│100│Out  │ All      │ All │ 0.0.0.0/0  │ Allow  │
└─────────┴──────────┴─────┴────────────┴────────┘

Problem: Port 3306 only! What about return traffic?
```

**CORRECT NACL Configuration:**

```
Subnet B (DB) NACL - INBOUND:
┌─────┬──────────┬──────────────┬────────────┬────────┐
│ #   │ Protocol │ Port Range   │ CIDR       │ Action │
├─────┼──────────┼──────────────┼────────────┼────────┤
│100  │ TCP      │ 3306-3306    │10.0.1.0/24 │ Allow  │
│110  │ TCP      │ 22-22        │10.0.3.0/24 │ Allow  │
│120  │ TCP      │1024-65535    │10.0.0.0/16 │ Allow  │
│130  │ TCP      │1024-65535    │ 0.0.0.0/0  │ Allow  │
│*    │ All      │ All          │ 0.0.0.0/0  │ Deny   │
└─────┴──────────┴──────────────┴────────────┴────────┘

OUTBOUND:
┌─────┬──────────┬──────────────┬────────────┬────────┐
│ #   │ Protocol │ Port Range   │ CIDR       │ Action │
├─────┼──────────┼──────────────┼────────────┼────────┤
│100  │ TCP      │1024-65535    │10.0.0.0/16 │ Allow  │
│110  │ TCP      │1024-65535    │ 0.0.0.0/0  │ Allow  │
│*    │ All      │ All          │ 0.0.0.0/0  │ Deny   │
└─────┴──────────┴──────────────┴────────────┴────────┘
```

**Why Port 3306 Only Fails:**

```
Web → DB Query (Port 3306):
┌─────────┐          ┌──────────┐
│ Web Srv │          │ DB Srv   │
└─────────┘          └──────────┘
     │                    │
     │──(Port 3306)──→    │  ✓ NACL inbound rule 100 allows
     │                    │
     │   ←─(Response)─ ←──│  ✗ Problem: Response from port 3306
     │                    │     back to ephemeral port 54321
     │                    │
     │                    │     NACL outbound needs to allow
     │                    │     ephemeral ports 1024-65535
     │                    │     Rule 100 enables this

Without ephemeral port rule:
- Query goes through (port 3306)
- Response blocked (no rule for ephemeral return)
- Connection hangs
- Application timeout after seconds
```

**Security Group Configuration:**

```bash
# Create SGs
WEB_SG=$(aws ec2 create-security-group \
  --group-name web --vpc-id vpc-12345678 \
  --query 'GroupId' --output text)

DB_SG=$(aws ec2 create-security-group \
  --group-name db --vpc-id vpc-12345678 \
  --query 'GroupId' --output text)

BASTION_SG=$(aws ec2 create-security-group \
  --group-name bastion --vpc-id vpc-12345678 \
  --query 'GroupId' --output text)

# Bastion SSH (public)
aws ec2 authorize-security-group-ingress \
  --group-id $BASTION_SG \
  --protocol tcp --port 22 --cidr 203.0.113.0/24

# Web: SSH from Bastion, HTTP to internet
aws ec2 authorize-security-group-ingress \
  --group-id $WEB_SG \
  --protocol tcp --port 22 --source-group $BASTION_SG

aws ec2 authorize-security-group-ingress \
  --group-id $WEB_SG \
  --protocol tcp --port 80 --cidr 0.0.0.0/0

# DB: MySQL from Web, SSH from Bastion
aws ec2 authorize-security-group-ingress \
  --group-id $DB_SG \
  --protocol tcp --port 3306 --source-group $WEB_SG

aws ec2 authorize-security-group-ingress \
  --group-id $DB_SG \
  --protocol tcp --port 22 --source-group $BASTION_SG

# Web outbound to DB (SG references work!)
aws ec2 authorize-security-group-egress \
  --group-id $WEB_SG \
  --protocol tcp --port 3306 --destination-group $DB_SG

echo "✓ Configured"
```

**Key Learnings:**

1. **Ephemeral Ports:** NACL requires explicit rules for return traffic
2. **SG References:** Security groups can reference other SGs (dynamic)
3. **Stateless Complexity:** NACLs need more rules for same policy
4. **Layered Approach:** Use both for defense-in-depth
5. **Troubleshooting:** If NACL only allows specific port, add ephemeral range

---

### Question 4: Blocking Attack (Advanced)

**Question:**
"Your infrastructure is under DDoS from 203.0.113.0/24. You have 50 instances across 10 subnets. Quickly block this traffic. Which tool (SG or NACL) and why? Show the command."

**Model Answer:**

**Decision Matrix:**

```
Option 1: Add deny rule to 50 Security Groups
├─ Time: 50 × 2 minutes = 100 minutes ✗
├─ Complexity: Manual, error-prone ✗
├─ Maintenance: Update each when instances added ✗
└─ Verdict: Too slow ✗

Option 2: Add deny rule to 10 Subnet NACLs
├─ Time: 10 × 1 minute = 10 minutes ✓
├─ Complexity: Single rule per NACL ✓
├─ Maintenance: Automatic for all instances in subnet ✓
└─ Verdict: Best approach ✓

Option 3: WAF/GuardDuty
├─ Time: ALB WAF rule = 5 minutes ✓
├─ But: Only works if ALB present (not all architectures)
└─ Recommendation: Use with NACL for comprehensive blocking
```

**Recommended Approach: NACL Rule**

**Why NACL?**
1. **Speed:** 10 rules vs 50 rules
2. **Scope:** Blocks at subnet level (before reaching instance)
3. **Efficiency:** No need to update when instances added/removed
4. **Priority:** Deny rule can be rule #1 (evaluated first)

**Implementation:**

```bash
#!/bin/bash

# Get all NACLs in VPC
NACL_IDS=$(aws ec2 describe-network-acls \
  --filters "Name=vpc-id,Values=vpc-12345678" \
  --query 'NetworkAcls[].NetworkAclId' \
  --output text)

ATTACKER_CIDR="203.0.113.0/24"
RULE_NUMBER=10

# Add deny rule to each NACL (highest priority)
for NACL in $NACL_IDS; do
  echo "Adding deny rule to $NACL"
  
  # Inbound deny
  aws ec2 create-network-acl-entry \
    --network-acl-id $NACL \
    --rule-number $RULE_NUMBER \
    --protocol -1 \
    --cidr-block $ATTACKER_CIDR \
    --ingress \
    2>/dev/null || echo "Rule may already exist"
  
  # Outbound deny (block exfiltration)
  aws ec2 create-network-acl-entry \
    --network-acl-id $NACL \
    --rule-number $RULE_NUMBER \
    --protocol -1 \
    --cidr-block $ATTACKER_CIDR \
    --egress \
    2>/dev/null || echo "Rule may already exist"
done

echo "✓ DDoS blocked in 10 NACLs"

# Verification
aws ec2 describe-network-acls \
  --filters "Name=vpc-id,Values=vpc-12345678" \
  --query 'NetworkAcls[0].Entries[?CidrBlock==`'$ATTACKER_CIDR'`]' \
  --output table

# Result: Traffic from 203.0.113.0/24 blocked at all subnets
```

**Why Not Security Groups?**

```
Problem: 50 instances × 2 commands (inbound+outbound) = 100 operations

# Option: Modify each SG (time-consuming)
for SG in sg-001 sg-002 ... sg-050; do
  aws ec2 authorize-security-group-ingress \
    --group-id $SG \
    --protocol -1 \
    --cidr-block 203.0.113.0/24 \
    --action Deny
done

Result: Slow, manual, error-prone, not scalable
```

**Multi-Layer Defense:**

```bash
# Layer 1: NACL (fastest, broadest)
aws ec2 create-network-acl-entry \
  --network-acl-id acl-12345678 \
  --rule-number 10 \
  --protocol -1 \
  --cidr-block 203.0.113.0/24 \
  --ingress

# Layer 2: ALB WAF (pattern-based, L7)
aws wafv2 create-ip-set \
  --name attacker-ips \
  --scope REGIONAL \
  --region us-east-1 \
  --addresses "[\"203.0.113.0/24\"]"

# Layer 3: Security Group (retroactive denial)
aws ec2 authorize-security-group-ingress \
  --group-id sg-12345678 \
  --protocol -1 \
  --cidr-block 203.0.113.0/24 \
  --action Deny

Result: Triple-layered blocking for defense-in-depth
```

**Key Insight:**
> "In DDoS scenarios, use NACLs for immediate, broad protection across entire subnets. Security groups are for fine-grained, per-instance control that's too slow for emergencies."

---

### Question 5: Architecture Design (Advanced)

**Question:**
"Design a secure multi-tier VPC with public, private, and isolated subnets. Include NACLs and Security Groups. What rules do you need for inter-tier communication?"

**Model Answer:**

**Architecture:**

```
┌─────────────────────────────────────────────────────────┐
│ VPC: 10.0.0.0/16                                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────────────────────────────────────┐           │
│  │ Public Subnet (10.0.1.0/24)              │           │
│  │ NACL: PubNACL (allow 80,443,22,eph)     │           │
│  │ Route: 0.0.0.0/0 → IGW                   │           │
│  │                                          │           │
│  │  ┌─────────────────────────────────┐    │           │
│  │  │ ALB/Bastion (sg-pub)            │    │           │
│  │  │ - Inbound: 80,443 (all) + 22 (*) │    │           │
│  │  │ - Outbound: All                 │    │           │
│  │  └─────────────────────────────────┘    │           │
│  └──────────────────────────────────────────┘           │
│              ↓                                          │
│  ┌──────────────────────────────────────────┐           │
│  │ Private Subnet - Web (10.0.2.0/24)      │           │
│  │ NACL: WebNACL (allow 80,443,22,eph)    │           │
│  │ Route: 0.0.0.0/0 → NAT                   │           │
│  │                                          │           │
│  │  ┌─────────────────────────────────┐    │           │
│  │  │ Web Servers (sg-web)            │    │           │
│  │  │ - Inbound: 80,443 (ALB)         │    │           │
│  │  │           22 (Bastion via SG)   │    │           │
│  │  │ - Outbound: 3306 (app sg)       │    │           │
│  │  │            443,80 (internet)    │    │           │
│  │  └─────────────────────────────────┘    │           │
│  └──────────────────────────────────────────┘           │
│              ↓                                          │
│  ┌──────────────────────────────────────────┐           │
│  │ Private Subnet - App (10.0.3.0/24)      │           │
│  │ NACL: AppNACL (allow 8080,22,eph)      │           │
│  │ Route: 0.0.0.0/0 → NAT                   │           │
│  │                                          │           │
│  │  ┌─────────────────────────────────┐    │           │
│  │  │ App Servers (sg-app)            │    │           │
│  │  │ - Inbound: 8080 (web sg)        │    │           │
│  │  │           22 (Bastion via SG)   │    │           │
│  │  │ - Outbound: 3306 (db sg)        │    │           │
│  │  │            443,80 (internet)    │    │           │
│  │  └─────────────────────────────────┘    │           │
│  └──────────────────────────────────────────┘           │
│              ↓                                          │
│  ┌──────────────────────────────────────────┐           │
│  │ Isolated Subnet (10.0.4.0/24)           │           │
│  │ NACL: DBNACLs (allow 3306,22,eph)      │           │
│  │ Route: None (no internet access)         │           │
│  │                                          │           │
│  │  ┌─────────────────────────────────┐    │           │
│  │  │ Databases (sg-db)               │    │           │
│  │  │ - Inbound: 3306 (app sg)        │    │           │
│  │  │           22 (Bastion via SG)   │    │           │
│  │  │ - Outbound: None (isolated)     │    │           │
│  │  └─────────────────────────────────┘    │           │
│  └──────────────────────────────────────────┘           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Security Group Configuration:**

```bash
#!/bin/bash

# Create Security Groups
SG_PUB=$(aws ec2 create-security-group \
  --group-name sg-public --vpc-id vpc-12345678 \
  --description "Public tier (ALB/Bastion)" \
  --query 'GroupId' --output text)

SG_WEB=$(aws ec2 create-security-group \
  --group-name sg-web --vpc-id vpc-12345678 \
  --description "Web tier" \
  --query 'GroupId' --output text)

SG_APP=$(aws ec2 create-security-group \
  --group-name sg-app --vpc-id vpc-12345678 \
  --description "Application tier" \
  --query 'GroupId' --output text)

SG_DB=$(aws ec2 create-security-group \
  --group-name sg-db --vpc-id vpc-12345678 \
  --description "Database tier" \
  --query 'GroupId' --output text)

echo "Security Groups created"

# ─────────────────────────────────────────
# PUBLIC TIER (ALB/Bastion)
# ─────────────────────────────────────────

# Inbound: HTTP from internet
aws ec2 authorize-security-group-ingress \
  --group-id $SG_PUB --protocol tcp --port 80 \
  --cidr 0.0.0.0/0

# Inbound: HTTPS from internet
aws ec2 authorize-security-group-ingress \
  --group-id $SG_PUB --protocol tcp --port 443 \
  --cidr 0.0.0.0/0

# Inbound: SSH from admin network (replace with your IP)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_PUB --protocol tcp --port 22 \
  --cidr 203.0.113.0/24

# ─────────────────────────────────────────
# WEB TIER
# ─────────────────────────────────────────

# Inbound: HTTP from ALB
aws ec2 authorize-security-group-ingress \
  --group-id $SG_WEB --protocol tcp --port 80 \
  --source-group $SG_PUB

# Inbound: HTTPS from ALB
aws ec2 authorize-security-group-ingress \
  --group-id $SG_WEB --protocol tcp --port 443 \
  --source-group $SG_PUB

# Inbound: SSH from Bastion
aws ec2 authorize-security-group-ingress \
  --group-id $SG_WEB --protocol tcp --port 22 \
  --source-group $SG_PUB

# Outbound: HTTP to internet (default allow all)
# Outbound: HTTPS to internet (default allow all)

# Outbound: Port 8080 to App tier
aws ec2 authorize-security-group-egress \
  --group-id $SG_WEB --protocol tcp --port 8080 \
  --destination-group $SG_APP

# ─────────────────────────────────────────
# APP TIER
# ─────────────────────────────────────────

# Inbound: Port 8080 from Web tier
aws ec2 authorize-security-group-ingress \
  --group-id $SG_APP --protocol tcp --port 8080 \
  --source-group $SG_WEB

# Inbound: SSH from Bastion
aws ec2 authorize-security-group-ingress \
  --group-id $SG_APP --protocol tcp --port 22 \
  --source-group $SG_PUB

# Outbound: MySQL to DB tier
aws ec2 authorize-security-group-egress \
  --group-id $SG_APP --protocol tcp --port 3306 \
  --destination-group $SG_DB

# ─────────────────────────────────────────
# DB TIER
# ─────────────────────────────────────────

# Inbound: MySQL from App tier
aws ec2 authorize-security-group-ingress \
  --group-id $SG_DB --protocol tcp --port 3306 \
  --source-group $SG_APP

# Inbound: SSH from Bastion
aws ec2 authorize-security-group-ingress \
  --group-id $SG_DB --protocol tcp --port 22 \
  --source-group $SG_PUB

# Outbound: Remove default allow-all (locked down)
aws ec2 revoke-security-group-egress \
  --group-id $SG_DB --protocol -1 \
  --cidr 0.0.0.0/0

echo "✓ Security Groups configured"
```

**Network ACL Configuration:**

```bash
#!/bin/bash

# ─────────────────────────────────────────
# PUBLIC SUBNET NACL
# ─────────────────────────────────────────

PUB_NACL=$(aws ec2 create-network-acl \
  --vpc-id vpc-12345678 \
  --tag-specifications 'ResourceType=network-acl,Tags=[{Key=Name,Value=pub-nacl}]' \
  --query 'NetworkAcl.NetworkAclId' --output text)

# Inbound rules
aws ec2 create-network-acl-entry \
  --network-acl-id $PUB_NACL --rule-number 100 \
  --protocol tcp --port-range From=80,To=80 \
  --cidr-block 0.0.0.0/0 --ingress

aws ec2 create-network-acl-entry \
  --network-acl-id $PUB_NACL --rule-number 110 \
  --protocol tcp --port-range From=443,To=443 \
  --cidr-block 0.0.0.0/0 --ingress

aws ec2 create-network-acl-entry \
  --network-acl-id $PUB_NACL --rule-number 120 \
  --protocol tcp --port-range From=22,To=22 \
  --cidr-block 203.0.113.0/24 --ingress

aws ec2 create-network-acl-entry \
  --network-acl-id $PUB_NACL --rule-number 130 \
  --protocol tcp --port-range From=1024,To=65535 \
  --cidr-block 0.0.0.0/0 --ingress

# Outbound rules
aws ec2 create-network-acl-entry \
  --network-acl-id $PUB_NACL --rule-number 100 \
  --protocol -1 --cidr-block 0.0.0.0/0 --egress

aws ec2 associate-network-acl \
  --network-acl-id $PUB_NACL \
  --subnet-id subnet-pub

# ─────────────────────────────────────────
# WEB SUBNET NACL
# ─────────────────────────────────────────

WEB_NACL=$(aws ec2 create-network-acl \
  --vpc-id vpc-12345678 \
  --tag-specifications 'ResourceType=network-acl,Tags=[{Key=Name,Value=web-nacl}]' \
  --query 'NetworkAcl.NetworkAclId' --output text)

# Inbound: HTTP/HTTPS from public (ALB/bastion)
aws ec2 create-network-acl-entry \
  --network-acl-id $WEB_NACL --rule-number 100 \
  --protocol tcp --port-range From=80,To=80 \
  --cidr-block 10.0.1.0/24 --ingress

aws ec2 create-network-acl-entry \
  --network-acl-id $WEB_NACL --rule-number 110 \
  --protocol tcp --port-range From=443,To=443 \
  --cidr-block 10.0.1.0/24 --ingress

# Inbound: SSH from bastion
aws ec2 create-network-acl-entry \
  --network-acl-id $WEB_NACL --rule-number 120 \
  --protocol tcp --port-range From=22,To=22 \
  --cidr-block 10.0.1.0/24 --ingress

# Inbound: Ephemeral ports for responses
aws ec2 create-network-acl-entry \
  --network-acl-id $WEB_NACL --rule-number 130 \
  --protocol tcp --port-range From=1024,To=65535 \
  --cidr-block 10.0.0.0/16 --ingress

aws ec2 create-network-acl-entry \
  --network-acl-id $WEB_NACL --rule-number 140 \
  --protocol tcp --port-range From=1024,To=65535 \
  --cidr-block 0.0.0.0/0 --ingress

# Outbound: All (to internet + internal)
aws ec2 create-network-acl-entry \
  --network-acl-id $WEB_NACL --rule-number 100 \
  --protocol -1 --cidr-block 0.0.0.0/0 --egress

aws ec2 associate-network-acl \
  --network-acl-id $WEB_NACL \
  --subnet-id subnet-web

# Similar for APP and DB subnets...

echo "✓ NACLs configured"
```

**Testing Connectivity:**

```bash
# From Bastion to Web server
ssh ec2-user@web-server-ip
ping web-server-ip

# From Web to App (internal)
curl http://app-server:8080

# From App to DB (internal)
mysql -h db-server -u user -p password

# Verify DB cannot reach internet
ssh ec2-user@db-server
curl http://example.com
# Should timeout (no internet route + no egress rule)
```

**Key Design Principles:**

1. **Layered Security:** Each tier has both SG and NACL rules
2. **Least Privilege:** Only necessary ports/sources allowed
3. **SG for Dynamic:** Reference other SGs for instance-to-instance
4. **NACL for Broad:** Block at subnet level for efficiency
5. **Ephemeral Ports:** Explicitly allow return traffic in NACL
6. **Isolation:** DB tier has no internet access
7. **Bastion Pattern:** Single entry point for SSH
8. **No Implicit Trust:** Even internal traffic must be allowed

---

**End of Document**

---

## Summary

**Security Groups & Network ACLs are complementary firewall layers:**

**Security Groups (Instance-Level):**
- Stateful filtering
- Fine-grained application control
- Reference other security groups
- Best for precise access control

**Network ACLs (Subnet-Level):**
- Stateless filtering
- Broad subnet-level control
- Explicit deny capabilities
- Best for compliance and DDoS blocking

**Use both for defense-in-depth security architecture.**