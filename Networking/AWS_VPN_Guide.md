# VPN (Site-to-Site and Client VPN) — Secure Connectivity Options

## Complete Guide with Detailed Examples and Top 5 Interview Questions

---

## Table of Contents

1. [Overview](#overview)
2. [VPN Fundamentals](#vpn-fundamentals)
3. [Site-to-Site VPN](#site-to-site-vpn)
4. [Client VPN](#client-vpn)
5. [Architecture Comparison](#architecture-comparison)
6. [Step-by-Step Implementation](#step-by-step-implementation)
7. [Advanced Features](#advanced-features)
8. [Security Best Practices](#security-best-practices)
9. [Troubleshooting & Monitoring](#troubleshooting--monitoring)
10. [Top 5 Interview Questions](#top-5-interview-questions)

---

## Overview

AWS VPN provides secure, encrypted connections to your AWS infrastructure from remote locations and on-premises networks. There are two primary VPN solutions:

**Site-to-Site VPN:**
- Connects entire on-premises network to AWS VPC
- Suitable for branch offices, data centers, hybrid cloud
- Supports static and dynamic routing
- High availability with dual tunnels

**Client VPN:**
- Enables individual users to connect remotely
- Suitable for remote workforce, temporary access
- Per-user authentication and authorization
- Full or split tunnel modes

### Key Benefits

```
✓ Encryption - All traffic encrypted end-to-end (IPsec/TLS)
✓ Security - Protect data in transit
✓ Compliance - Meet regulatory requirements (HIPAA, PCI-DSS)
✓ Cost-effective - Uses existing internet connection
✓ Quick setup - Minutes instead of weeks
✓ Scalable - Supports unlimited users/connections
✓ Flexibility - Works with on-premises, branch offices, remote users
```

---

## VPN Fundamentals

### How VPN Works

**Encryption Process:**

```
Plaintext Traffic
    ↓
Encrypt with symmetric key (AES-128, AES-256)
    ↓
Add authentication (integrity check)
    ↓
Encapsulate in IPsec tunnel
    ↓
Send over public internet
    ↓
(At destination)
    ↓
Decapsulate tunnel
    ↓
Verify authentication
    ↓
Decrypt with same symmetric key
    ↓
Plaintext Traffic
```

### IPsec vs TLS

| Aspect | IPsec (Site-to-Site) | TLS (Client VPN) |
|--------|-------------------|-----------------|
| Protocol Layer | Network Layer (Layer 3) | Transport Layer (Layer 4) |
| Encryption | Built into protocols | Application-based (OpenVPN) |
| Setup | Complex (requires hardware) | Simple (software client) |
| Performance | High throughput | Slightly lower overhead |
| Use Case | Network-to-network | Remote individuals |
| Latency | Low (direct tunnel) | Slightly higher (internet path) |

### Key VPN Concepts

**Tunnel:**
- Encrypted communication channel between two endpoints
- Site-to-Site: Creates 2 redundant tunnels automatically
- Client VPN: Each client creates own tunnel

**Endpoint:**
- VPN gateway where tunnel terminates
- Virtual Private Gateway (Site-to-Site)
- Client VPN Endpoint (Client VPN)

**Customer Gateway:**
- On-premises VPN device or software
- Site-to-Site only
- Connects to Virtual Private Gateway

**Authentication:**
- Pre-shared key (PSK) - Site-to-Site
- Certificates - Client VPN
- Directory Service (Active Directory) - Client VPN

---

## Site-to-Site VPN

### Architecture Overview

```
┌──────────────────────────────────────┐
│ On-Premises Network                  │
│ (10.0.0.0/8)                        │
│                                      │
│ ┌──────────────────────────────┐   │
│ │ Customer Gateway             │   │
│ │ (VPN Device)                 │   │
│ │ IP: 203.0.113.12             │   │
│ └──────────────────────────────┘   │
│                ↓                     │
│      IPsec Tunnel (Encrypted)       │
│                ↓                     │
└──────────────────────────────────────┘
          Public Internet
┌──────────────────────────────────────┐
│ AWS Cloud                            │
│ (172.31.0.0/16)                     │
│                                      │
│ ┌──────────────────────────────┐   │
│ │ Virtual Private Gateway      │   │
│ │ (VPN Endpoint)               │   │
│ │ AWS managed                  │   │
│ └──────────────────────────────┘   │
│                ↓                     │
│      ┌─────────────────────┐       │
│      │ VPC                 │       │
│      │ EC2, RDS, S3 Access │       │
│      └─────────────────────┘       │
└──────────────────────────────────────┘
```

### Components

**1. Customer Gateway (CGW)**
- Represents on-premises VPN device
- Static IP address required
- Can be physical router (Cisco, Juniper) or software (OpenSwan, StrongSwan)
- Initiates connections to Virtual Private Gateway

**2. Virtual Private Gateway (VGW)**
- AWS-managed VPN endpoint
- Attached to VPC
- No configuration needed (AWS manages)
- Receives connections from Customer Gateways

**3. VPN Connection**
- Link between CGW and VGW
- Two tunnels created automatically (for redundancy)
- Each tunnel uses different IP
- Transparent failover if one tunnel fails

**4. Routing**
- Static routing: Manually specify routes
- Dynamic routing: BGP automatic route propagation
- Route tables updated to send traffic through VPN

### Creating Site-to-Site VPN

#### Step 1: Create Customer Gateway

```bash
# Create CGW
aws ec2 create-customer-gateway \
  --type ipsec.1 \
  --bgp-asn 65000 \
  --public-ip 203.0.113.12 \
  --region us-east-1

# Response:
# {
#   "CustomerGateway": {
#     "CustomerGatewayId": "cgw-0a1b2c3d4e5f6g7h8",
#     "Type": "ipsec.1",
#     "State": "available",
#     "BgpAsn": "65000",
#     "PublicIp": "203.0.113.12"
#   }
# }

CGW_ID="cgw-0a1b2c3d4e5f6g7h8"
```

#### Step 2: Create Virtual Private Gateway

```bash
# Create VGW
aws ec2 create-vpn-gateway \
  --type ipsec.1 \
  --bgp-asn 64512 \
  --amazon-side-asn 64512 \
  --region us-east-1

# Response:
# {
#   "VpnGateway": {
#     "VpnGatewayId": "vgw-0x1y2z3a4b5c6d7e8",
#     "Type": "ipsec.1",
#     "State": "available",
#     "AmazonSideAsn": 64512
#   }
# }

VGW_ID="vgw-0x1y2z3a4b5c6d7e8"

# Attach VGW to VPC
aws ec2 attach-vpn-gateway \
  --vpn-gateway-id $VGW_ID \
  --vpc-id vpc-12345678 \
  --region us-east-1

# Enable route propagation
aws ec2 enable-vgw-route-propagation \
  --route-table-id rtb-12345678 \
  --gateway-id $VGW_ID \
  --region us-east-1
```

#### Step 3: Create VPN Connection

```bash
# Create VPN Connection
aws ec2 create-vpn-connection \
  --type ipsec.1 \
  --customer-gateway-id $CGW_ID \
  --vpn-gateway-id $VGW_ID \
  --options StaticRoutesOnly=false \
  --region us-east-1

# Response:
# {
#   "VpnConnection": {
#     "VpnConnectionId": "vpn-0abc1def2ghi3jkl4",
#     "State": "pending",
#     "CustomerGatewayId": "cgw-0a1b2c3d4e5f6g7h8",
#     "VpnGatewayId": "vgw-0x1y2z3a4b5c6d7e8",
#     "Type": "ipsec.1",
#     "VpnConnectionOptions": {
#       "StaticRoutesOnly": false
#     },
#     "VgwTelemetry": [
#       {
#         "AcceptedRouteCount": 0,
#         "LastStatusChange": "2025-12-02T10:00:00Z",
#         "Status": "DOWN",
#         "TunnelAddress": "75.101.0.1"
#       }
#     ]
#   }
# }

VPN_ID="vpn-0abc1def2ghi3jkl4"
```

#### Step 4: Download VPN Configuration

```bash
# Get configuration file for on-premises device
aws ec2 describe-vpn-connections \
  --vpn-connection-ids $VPN_ID \
  --region us-east-1 \
  --query 'VpnConnections[0].CustomerGatewayConfiguration' \
  --output text > vpn-config.xml

# Contains:
# - Tunnel details (IP addresses, encryption parameters)
# - Pre-shared keys (PSK) for authentication
# - IPsec configuration (IKE version, algorithms)
# - Dead peer detection settings
```

#### Step 5: Configure On-Premises VPN Device

**Example: StrongSwan Configuration**

```bash
# /etc/ipsec.d/aws-vpn.conf
conn aws-tunnel1
    left=%defaultroute
    leftsubnet=10.0.0.0/8
    right=75.101.0.1          # From AWS config
    rightsubnet=172.31.0.0/16
    authby=secret
    auto=start
    ikev2=yes
    ike=aes128-sha1-modp1024
    esp=aes128-sha1
    aggressive=no
    type=tunnel
    dpdaction=restart
    dpddelay=10
    dpdtimeout=30

conn aws-tunnel2
    left=%defaultroute
    leftsubnet=10.0.0.0/8
    right=75.101.0.2          # Backup tunnel
    rightsubnet=172.31.0.0/16
    authby=secret
    auto=start
    ikev2=yes
    ike=aes128-sha1-modp1024
    esp=aes128-sha1
    aggressive=no
    type=tunnel
    dpdaction=restart
    dpddelay=10
    dpdtimeout=30

# /etc/ipsec.d/aws-vpn.secrets
# PSK from AWS configuration
75.101.0.1 10.0.0.1 : PSK "abc123xyz456..."
75.101.0.2 10.0.0.1 : PSK "def456uvw789..."
```

#### Step 6: Verify Connection

```bash
# Check VPN connection status
aws ec2 describe-vpn-connections \
  --vpn-connection-ids $VPN_ID \
  --query 'VpnConnections[0].{
    State: State,
    Tunnel1Status: VgwTelemetry[0].Status,
    Tunnel2Status: VgwTelemetry[1].Status,
    Routes: Routes
  }' \
  --output table

# Expected output:
# State: available
# Tunnel1Status: UP
# Tunnel2Status: UP

# Ping test from on-premises
ping 172.31.1.100  # EC2 in AWS VPC
# Should receive response (traffic goes through VPN)

# View routes propagated via BGP
aws ec2 describe-route-tables \
  --route-table-ids rtb-12345678 \
  --query 'RouteTables[0].Routes' \
  --output table
```

### Site-to-Site VPN Routing

**Static Routing:**

```bash
# Manually add route for on-premises network
aws ec2 create-vpn-connection-route \
  --vpn-connection-id $VPN_ID \
  --destination-cidr-block 10.0.0.0/8

# Route table automatically updated when connection is up
aws ec2 describe-route-tables \
  --route-table-ids rtb-12345678 \
  --query 'RouteTables[0].Routes' \
  --output table

# Output:
# RouteTableId: rtb-12345678
# DestinationCidrBlock | TargetId    | State   | Origin
# 172.31.0.0/16        | local       | active  | CreateRouteTable
# 10.0.0.0/8           | vpn-connid  | active  | EnableVgwRouteProp
```

**Dynamic Routing (BGP):**

```bash
# VPN connection automatically advertises routes via BGP
# On-premises BGP router learns AWS routes
# AWS learns on-premises routes

# Create CGW with BGP enabled
aws ec2 create-customer-gateway \
  --type ipsec.1 \
  --bgp-asn 65000 \
  --public-ip 203.0.113.12

# AWS responds with BGP ASN 64512
# BGP peers exchange routes automatically
# No manual route creation needed

# Verify BGP routes
aws ec2 describe-route-tables \
  --route-table-ids rtb-12345678 \
  --query 'RouteTables[0].Routes[*].{
    Destination: DestinationCidrBlock,
    Target: GatewayId,
    Origin: Origin
  }' \
  --output table

# Origin: EnableVgwRoutePropagation indicates BGP-learned route
```

---

## Client VPN

### Architecture Overview

```
┌─────────────────────────────────────────────────┐
│ Remote User (Home/Coffee Shop)                  │
│ ┌──────────────────────────────────────────┐   │
│ │ OpenVPN Client Application               │   │
│ │ - Client certificate                     │   │
│ │ - Client configuration file              │   │
│ │ - IP: 172.27.0.x (allocated from CIDR)  │   │
│ └──────────────────────────────────────────┘   │
│              ↓                                   │
│      TLS Tunnel (Encrypted)                     │
│              ↓                                   │
└─────────────────────────────────────────────────┘
          Public Internet
┌─────────────────────────────────────────────────┐
│ AWS Client VPN Endpoint                         │
│ (172.27.0.0/22 - Client CIDR)                  │
│                                                 │
│ ┌──────────────────────────────────────────┐   │
│ │ AWS Client VPN Service                   │   │
│ │ - Multi-AZ deployment                    │   │
│ │ - Auto-scaling                           │   │
│ │ - Connection logging                     │   │
│ └──────────────────────────────────────────┘   │
│              ↓                                   │
│      Associated Subnets                         │
│              ↓                                   │
│ ┌──────────────────────────────────────────┐   │
│ │ VPC                                      │   │
│ │ EC2, RDS, S3 (via gateway endpoint)      │   │
│ │ On-premises (via Site-to-Site VPN)       │   │
│ └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

### Key Components

**1. Client VPN Endpoint**
- Managed service (serverless)
- Multi-AZ automatically
- Scales to handle thousands of concurrent connections
- Accepts connections on port 443 (UDP/TCP)

**2. Target Network Association**
- Links endpoint to subnets
- Clients route traffic through associated subnets
- Multiple subnet associations for redundancy

**3. Authorization Rules**
- Control which clients access which networks
- Support AD groups, specific users, or all users
- Can restrict to specific CIDR blocks

**4. Client Certificates**
- Mutual TLS authentication
- Client certificate + key for authentication
- Generated using ACM (AWS Certificate Manager)

**5. Network Interface**
- ENI created in target subnet
- Clients communicate through this interface
- Security groups control traffic

### Creating Client VPN

#### Step 1: Generate Certificates

```bash
# Create Certificate Authority (CA) certificate
# Using easy-rsa tool

# Navigate to easy-rsa directory
cd /opt/easy-rsa

# Initialize PKI
./easyrsa init-pki

# Build CA
./easyrsa build-ca nopass

# Generate server certificate
./easyrsa build-server-full server nopass

# Generate client certificate
./easyrsa build-client-full client1.domain.tld nopass

# Generate client certificate for second user
./easyrsa build-client-full client2.domain.tld nopass

# Certificates location
# - CA cert: pki/ca.crt
# - Server cert: pki/issued/server.crt
# - Server key: pki/private/server.key
# - Client cert: pki/issued/client1.domain.tld.crt
# - Client key: pki/private/client1.domain.tld.key
```

#### Step 2: Import Certificates to ACM

```bash
# Import Server Certificate
aws acm import-certificate \
  --certificate fileb://pki/issued/server.crt \
  --certificate-chain fileb://pki/ca.crt \
  --private-key fileb://pki/private/server.key \
  --region us-east-1

# Response:
# {
#   "CertificateArn": "arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012"
# }

SERVER_CERT_ARN="arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012"

# Import Client Certificate
aws acm import-certificate \
  --certificate fileb://pki/issued/client1.domain.tld.crt \
  --certificate-chain fileb://pki/ca.crt \
  --private-key fileb://pki/private/client1.domain.tld.key \
  --region us-east-1

CLIENT_CERT_ARN="arn:aws:acm:us-east-1:123456789012:certificate/87654321-4321-4321-4321-210987654321"
```

#### Step 3: Create Client VPN Endpoint

```bash
# Create endpoint
aws ec2 create-client-vpn-endpoint \
  --client-cidr-block 172.27.0.0/22 \
  --server-certificate-arn $SERVER_CERT_ARN \
  --authentication-options Type=certificate-authentication,MutualAuthentication.ClientRootCertificateChainArn=$CLIENT_CERT_ARN \
  --connection-log-options CloudwatchLogGroup=/aws/clientvpn,Enabled=true \
  --description "Client VPN for remote access" \
  --split-tunnel-options SplitTunnelEnabled=true \
  --vpc-id vpc-12345678 \
  --security-group-ids sg-0a1b2c3d4e5f6g7h8 \
  --tag-specifications ResourceType=client-vpn-endpoint,Tags=[{Key=Name,Value=remote-access-vpn}] \
  --region us-east-1

# Response:
# {
#   "ClientVpnEndpointId": "cvpn-endpoint-0a1b2c3d4e5f6g7h8"
# }

CVPN_ID="cvpn-endpoint-0a1b2c3d4e5f6g7h8"

# Wait for endpoint to be available
aws ec2 describe-client-vpn-endpoints \
  --client-vpn-endpoint-ids $CVPN_ID \
  --query 'ClientVpnEndpoints[0].Status' \
  --output text
# Output: available (wait until this appears)
```

#### Step 4: Associate Target Network

```bash
# Associate subnets (target networks)
aws ec2 associate-client-vpn-target-network \
  --client-vpn-endpoint-id $CVPN_ID \
  --subnet-id subnet-12345678 \
  --region us-east-1

# Response:
# {
#   "AssociationId": "cvpn-assoc-0a1b2c3d4e5f6g7h8"
# }

ASSOC_ID="cvpn-assoc-0a1b2c3d4e5f6g7h8"

# Wait for association to complete
aws ec2 describe-client-vpn-target-networks \
  --client-vpn-endpoint-id $CVPN_ID \
  --query 'ClientVpnTargetNetworks[0].Status' \
  --output text
# Output: available (when ready)

# Associate multiple subnets for redundancy
aws ec2 associate-client-vpn-target-network \
  --client-vpn-endpoint-id $CVPN_ID \
  --subnet-id subnet-87654321 \
  --region us-east-1
```

#### Step 5: Add Authorization Rules

```bash
# Allow all authenticated users to access VPC CIDR
aws ec2 authorize-client-vpn-ingress \
  --client-vpn-endpoint-id $CVPN_ID \
  --target-network-cidr 172.31.0.0/16 \
  --authorize-all-groups \
  --description "Allow access to VPC" \
  --region us-east-1

# Allow specific group to access on-premises network
aws ec2 authorize-client-vpn-ingress \
  --client-vpn-endpoint-id $CVPN_ID \
  --target-network-cidr 10.0.0.0/8 \
  --access-group-id "admin-group" \
  --description "Allow access to on-premises (admin only)" \
  --region us-east-1

# Revoke authorization if needed
aws ec2 revoke-client-vpn-ingress \
  --client-vpn-endpoint-id $CVPN_ID \
  --target-network-cidr 10.0.0.0/8 \
  --revoke-all-groups \
  --region us-east-1

# List all authorization rules
aws ec2 describe-client-vpn-authorization-rules \
  --client-vpn-endpoint-id $CVPN_ID \
  --query 'AuthorizationRules[*].{
    CIDR: DestinationCidr,
    AccessGroups: AccessGroupId,
    Description: Description
  }' \
  --output table
```

#### Step 6: Download Client Configuration

```bash
# Export client configuration
aws ec2 export-client-vpn-client-certificate \
  --client-vpn-endpoint-id $CVPN_ID \
  --region us-east-1

# This generates configuration file

# Get endpoint address
aws ec2 describe-client-vpn-endpoints \
  --client-vpn-endpoint-ids $CVPN_ID \
  --query 'ClientVpnEndpoints[0].ClientVpnEndpointDnsName' \
  --output text
# Output: cvpn-endpoint-0a1b2c3d.prod.clientvpn.us-east-1.amazonaws.com
```

#### Step 7: Client Connection Configuration

**Client Configuration File (client.ovpn):**

```
client
proto udp
remote cvpn-endpoint-0a1b2c3d.prod.clientvpn.us-east-1.amazonaws.com 443
remote-random
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
cipher AES-128-GCM
verb 3
key-direction 1

# Inline certificates
<ca>
-----BEGIN CERTIFICATE-----
MIIDHTCCAgWgAwIBAgI...
-----END CERTIFICATE-----
</ca>

<cert>
-----BEGIN CERTIFICATE-----
MIIDWjCCAkICAQAwDQY...
-----END CERTIFICATE-----
</cert>

<key>
-----BEGIN PRIVATE KEY-----
MIIEvgIBADANBgkqhkiG...
-----END PRIVATE KEY-----
</key>

<tls-crypt>
#
# 2048 bit OpenVPN static key
#
-----BEGIN OpenVPN Static key V1-----
...
-----END OpenVPN Static key V1-----
</tls-crypt>
```

**Steps to Connect:**

1. Download OpenVPN client: `https://openvpn.net/community-downloads/`
2. Import client.ovpn configuration file
3. Enter authentication (if using AD integration)
4. Click "Connect"
5. VPN tunnel established
6. Access VPC resources as if on-premises

```bash
# After connecting, verify connection
ifconfig  # Linux/Mac: See tun interface (e.g., tun0)
ipconfig  # Windows: See new TAP adapter

# You should see:
# Client IP: 172.27.0.x (from Client CIDR block)
# Default route through VPN tunnel

# Test connectivity
ping 172.31.1.100  # EC2 instance in VPC
# Should respond

# Check routing table
route -n  # Linux/Mac
route print  # Windows
```

---

## Architecture Comparison

### Site-to-Site VPN vs Client VPN

| Feature | Site-to-Site VPN | Client VPN |
|---------|------------------|-----------|
| **Purpose** | Network-to-network connectivity | Individual user remote access |
| **Use Case** | Branch offices, data centers, hybrid cloud | Remote workers, temporary access |
| **Scale** | Few connections (typically 1-5) | Thousands of concurrent users |
| **Setup Time** | Hours/Days (complex config) | Minutes (fully managed) |
| **Required Device** | VPN appliance/router | Software client (OpenVPN) |
| **Cost Model** | Per connection + data transfer | Per connection-minute |
| **Latency** | Low (dedicated tunnel) | Slightly higher (internet path) |
| **Failover** | Dual tunnels automatic | Multi-AZ automatic |
| **Throughput** | High (limited by device) | Per-client limited |
| **Authentication** | Pre-shared key (PSK) | Certificates + optional AD |
| **Routing** | Static or BGP dynamic | Client-specific authorization |
| **Scalability** | Limited (VPN device bottleneck) | Unlimited (serverless) |
| **Compliance** | HIPAA, PCI-DSS | HIPAA, PCI-DSS, SOC2 |

### Combined Hybrid Connectivity

```
┌─────────────────────────────────────┐
│ Headquarters                        │
│ (Office Network)                    │
└─────────────────────────────────────┘
            ↓
    Site-to-Site VPN #1
            ↓
┌─────────────────────────────────────┐
│ AWS VPC                             │
├─────────────────────────────────────┤
│ ┌─────────────────────────────────┐ │
│ │ Virtual Private Gateway         │ │
│ │ ┌───────────────────────────┐   │ │
│ │ │ S2S VPN from HQ (always)  │   │ │
│ │ │ S2S VPN from Branch 1     │   │ │
│ │ │ S2S VPN from Branch 2     │   │ │
│ │ └───────────────────────────┘   │ │
│ └─────────────────────────────────┘ │
│                                     │
│ ┌─────────────────────────────────┐ │
│ │ Client VPN Endpoint             │ │
│ │ ┌───────────────────────────┐   │ │
│ │ │ Remote worker 1 (Seattle) │   │ │
│ │ │ Remote worker 2 (Toronto) │   │ │
│ │ │ Contractor (temporary)    │   │ │
│ │ └───────────────────────────┘   │ │
│ └─────────────────────────────────┘ │
│                                     │
│ ┌─────────────────────────────────┐ │
│ │ VPC Resources                   │ │
│ │ - EC2 instances                 │ │
│ │ - RDS databases                 │ │
│ │ - S3 buckets                    │ │
│ └─────────────────────────────────┘ │
└─────────────────────────────────────┘
            ↓
    Site-to-Site VPN #2
            ↓
┌─────────────────────────────────────┐
│ Branch Office 1                     │
└─────────────────────────────────────┘

Result: Secure access from all locations
```

---

## Step-by-Step Implementation

### Scenario: Hybrid Company Setup

**Requirements:**
- HQ in New York connects to AWS via Site-to-Site VPN
- Seattle office needs access via Site-to-Site VPN
- Remote workers access via Client VPN
- On-premises resources accessible from cloud

### Implementation Guide

**Phase 1: Site-to-Site VPN Setup (HQ)**

```bash
#!/bin/bash

# Variables
VPC_ID="vpc-12345678"
HQ_GATEWAY_IP="203.0.113.12"
HQ_NETWORK="10.0.0.0/8"
HQ_BGP_ASN="65000"

# Step 1: Create CGW for HQ
HQ_CGW=$(aws ec2 create-customer-gateway \
  --type ipsec.1 \
  --bgp-asn $HQ_BGP_ASN \
  --public-ip $HQ_GATEWAY_IP \
  --query 'CustomerGateway.CustomerGatewayId' \
  --output text)

echo "HQ CGW Created: $HQ_CGW"

# Step 2: Create VGW
VGW=$(aws ec2 create-vpn-gateway \
  --type ipsec.1 \
  --amazon-side-asn 64512 \
  --query 'VpnGateway.VpnGatewayId' \
  --output text)

echo "VGW Created: $VGW"

# Step 3: Attach VGW to VPC
aws ec2 attach-vpn-gateway \
  --vpn-gateway-id $VGW \
  --vpc-id $VPC_ID

echo "VGW Attached to VPC"

# Step 4: Create VPN Connection
HQ_VPN=$(aws ec2 create-vpn-connection \
  --type ipsec.1 \
  --customer-gateway-id $HQ_CGW \
  --vpn-gateway-id $VGW \
  --options StaticRoutesOnly=false \
  --query 'VpnConnection.VpnConnectionId' \
  --output text)

echo "HQ VPN Connection Created: $HQ_VPN"

# Step 5: Enable route propagation
RTB=$(aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'RouteTables[0].RouteTableId' \
  --output text)

aws ec2 enable-vgw-route-propagation \
  --route-table-id $RTB \
  --gateway-id $VGW

echo "Route Propagation Enabled"

# Step 6: Download configuration
aws ec2 describe-vpn-connections \
  --vpn-connection-ids $HQ_VPN \
  --query 'VpnConnections[0].CustomerGatewayConfiguration' \
  --output text > hq-vpn-config.xml

echo "Configuration saved to hq-vpn-config.xml"
```

**Phase 2: Site-to-Site VPN Setup (Branch)**

```bash
#!/bin/bash

# Variables
BRANCH_GATEWAY_IP="198.51.100.45"
BRANCH_NETWORK="192.168.0.0/16"
BRANCH_BGP_ASN="65001"
VGW="vgw-0x1y2z3a4b5c6d7e8"

# Create CGW for Branch
BRANCH_CGW=$(aws ec2 create-customer-gateway \
  --type ipsec.1 \
  --bgp-asn $BRANCH_BGP_ASN \
  --public-ip $BRANCH_GATEWAY_IP \
  --query 'CustomerGateway.CustomerGatewayId' \
  --output text)

echo "Branch CGW Created: $BRANCH_CGW"

# Create VPN Connection for Branch
BRANCH_VPN=$(aws ec2 create-vpn-connection \
  --type ipsec.1 \
  --customer-gateway-id $BRANCH_CGW \
  --vpn-gateway-id $VGW \
  --options StaticRoutesOnly=false \
  --query 'VpnConnection.VpnConnectionId' \
  --output text)

echo "Branch VPN Connection Created: $BRANCH_VPN"

# Download branch configuration
aws ec2 describe-vpn-connections \
  --vpn-connection-ids $BRANCH_VPN \
  --query 'VpnConnections[0].CustomerGatewayConfiguration' \
  --output text > branch-vpn-config.xml

echo "Configuration saved to branch-vpn-config.xml"

# Verify connections
aws ec2 describe-vpn-connections \
  --query 'VpnConnections[*].{
    ID: VpnConnectionId,
    CGW: CustomerGatewayId,
    State: State,
    Tunnel1: VgwTelemetry[0].Status,
    Tunnel2: VgwTelemetry[1].Status
  }' \
  --output table
```

**Phase 3: Client VPN Setup**

```bash
#!/bin/bash

# Certificate setup (one time)
cd /opt/easy-rsa

# Build CA and server
./easyrsa init-pki
./easyrsa build-ca nopass
./easyrsa build-server-full server nopass

# Build client certificates
for i in {1..5}; do
  ./easyrsa build-client-full client$i nopass
done

# Import to ACM
SERVER_CERT=$(aws acm import-certificate \
  --certificate fileb://pki/issued/server.crt \
  --certificate-chain fileb://pki/ca.crt \
  --private-key fileb://pki/private/server.key \
  --query 'CertificateArn' \
  --output text)

echo "Server Cert ARN: $SERVER_CERT"

CLIENT_CERT=$(aws acm import-certificate \
  --certificate fileb://pki/issued/client1.crt \
  --certificate-chain fileb://pki/ca.crt \
  --private-key fileb://pki/private/client1.key \
  --query 'CertificateArn' \
  --output text)

echo "Client Cert ARN: $CLIENT_CERT"

# Create Client VPN Endpoint
VPC_ID="vpc-12345678"
SUBNET_ID="subnet-12345678"
SG_ID="sg-0a1b2c3d4e5f6g7h8"

CVPN=$(aws ec2 create-client-vpn-endpoint \
  --client-cidr-block 172.27.0.0/22 \
  --server-certificate-arn $SERVER_CERT \
  --authentication-options Type=certificate-authentication,MutualAuthentication.ClientRootCertificateChainArn=$CLIENT_CERT \
  --connection-log-options CloudwatchLogGroup=/aws/clientvpn,Enabled=true \
  --split-tunnel-options SplitTunnelEnabled=true \
  --vpc-id $VPC_ID \
  --security-group-ids $SG_ID \
  --query 'ClientVpnEndpointId' \
  --output text)

echo "Client VPN Endpoint Created: $CVPN"

# Wait for availability
aws ec2 wait client-vpn-endpoint-available \
  --client-vpn-endpoint-ids $CVPN

# Associate target network
ASSOC=$(aws ec2 associate-client-vpn-target-network \
  --client-vpn-endpoint-id $CVPN \
  --subnet-id $SUBNET_ID \
  --query 'AssociationId' \
  --output text)

echo "Target Network Associated: $ASSOC"

# Wait for association
aws ec2 wait client-vpn-target-network-associated \
  --client-vpn-endpoint-id $CVPN \
  --association-ids $ASSOC

# Add authorization rules
aws ec2 authorize-client-vpn-ingress \
  --client-vpn-endpoint-id $CVPN \
  --target-network-cidr 172.31.0.0/16 \
  --authorize-all-groups \
  --description "VPC Access"

aws ec2 authorize-client-vpn-ingress \
  --client-vpn-endpoint-id $CVPN \
  --target-network-cidr 10.0.0.0/8 \
  --authorize-all-groups \
  --description "On-Premises Access"

echo "✓ Client VPN Setup Complete"
```

---

## Advanced Features

### 1. Split Tunneling vs Full Tunneling

**Full Tunneling (Default):**
- All traffic goes through VPN
- Higher security (all data encrypted)
- Higher latency (all traffic through AWS)

```bash
# Disable split tunnel (full tunnel)
aws ec2 modify-client-vpn-endpoint \
  --client-vpn-endpoint-id cvpn-id \
  --split-tunnel-options SplitTunnelEnabled=false
```

**Split Tunneling:**
- Only specified traffic through VPN
- Lower latency (local traffic direct)
- Mixed security (only specified traffic encrypted)

```bash
# Enable split tunnel
aws ec2 modify-client-vpn-endpoint \
  --client-vpn-endpoint-id cvpn-id \
  --split-tunnel-options SplitTunnelEnabled=true

# Add specific routes to VPN
aws ec2 create-client-vpn-route \
  --client-vpn-endpoint-id cvpn-id \
  --destination-cidr 172.31.0.0/16 \
  --target-vpc-subnet-id subnet-id
```

### 2. Active Directory Integration

```bash
# Use Active Directory for Client VPN authentication

aws ec2 create-client-vpn-endpoint \
  --client-cidr-block 172.27.0.0/22 \
  --server-certificate-arn $SERVER_CERT \
  --authentication-options Type=directory-service-authentication,DirectoryServiceAuthentication.DirectoryId=d-1234567890 \
  --split-tunnel-options SplitTunnelEnabled=true \
  --vpc-id vpc-12345678

# Users authenticate with AD credentials
# Can restrict by AD group
aws ec2 authorize-client-vpn-ingress \
  --client-vpn-endpoint-id cvpn-id \
  --target-network-cidr 172.31.0.0/16 \
  --access-group-id "CN=VPN-Users,OU=Groups,DC=example,DC=com"
```

### 3. Redundancy and High Availability

```bash
# Multi-AZ deployment for Site-to-Site VPN
aws ec2 describe-vpn-connections \
  --query 'VpnConnections[*].VgwTelemetry'

# Output shows two tunnels in different AZs
# {
#   "Status": "UP",
#   "TunnelAddress": "52.1.2.3",    # AZ-A tunnel
#   "OutsideIpAddress": "52.1.2.3"
# },
# {
#   "Status": "UP",
#   "TunnelAddress": "52.2.2.3",    # AZ-B tunnel
#   "OutsideIpAddress": "52.2.2.3"
# }

# Automatic failover if one tunnel fails
# No manual intervention needed
```

### 4. VPN Monitoring and Logging

```bash
# Enable CloudWatch logging
aws ec2 create-client-vpn-endpoint \
  --connection-log-options \
    CloudwatchLogGroup=/aws/clientvpn/endpoint-logs,\
    Enabled=true

# Query connection logs
aws logs filter-log-events \
  --log-group-name /aws/clientvpn/endpoint-logs \
  --start-time $(date -d '1 hour ago' +%s)000 \
  --query 'events[*].[message]' \
  --output text

# Monitor connection count
aws cloudwatch get-metric-statistics \
  --namespace AWS/ClientVPN \
  --metric-name ClientVpnActiveConnections \
  --dimensions Name=ClientVpnEndpointId,Value=cvpn-id \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average,Sum

# Monitor bytes transferred
aws cloudwatch get-metric-statistics \
  --namespace AWS/ClientVPN \
  --metric-name ClientVpnBytesIn \
  --dimensions Name=ClientVpnEndpointId,Value=cvpn-id \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Sum
```

---

## Security Best Practices

### Site-to-Site VPN

```bash
# 1. Use strong encryption algorithms
# In IPsec config:
# ike=aes256-sha256-modp2048     # Phase 1 (key exchange)
# esp=aes256-sha256              # Phase 2 (encryption)
# DPD (Dead Peer Detection) enabled
# Rekey interval: 1 hour

# 2. Enable AWS VPN connection logging
aws ec2 create-vpn-connection \
  --type ipsec.1 \
  --customer-gateway-id cgw-id \
  --vpn-gateway-id vgw-id \
  --options StaticRoutesOnly=false,TunnelOptions=[{DPDTimeoutAction=restart}]

# 3. Restrict security group on VGW attachment
aws ec2 authorize-security-group-ingress \
  --group-id sg-vpn-servers \
  --protocol tcp --port 443 \
  --source-group sg-vpn-clients

# 4. Monitor tunnel status
aws ec2 describe-vpn-connections \
  --query 'VpnConnections[*].VgwTelemetry[*].Status' \
  --output text
```

### Client VPN

```bash
# 1. Use certificate-based authentication
# Avoid username/password alone

# 2. Implement AD group-based access control
aws ec2 authorize-client-vpn-ingress \
  --client-vpn-endpoint-id cvpn-id \
  --target-network-cidr 172.31.0.0/16 \
  --access-group-id "admin-group"

# 3. Enable audit logging
aws ec2 create-client-vpn-endpoint \
  --connection-log-options CloudwatchLogGroup=/aws/clientvpn,Enabled=true

# 4. Use security groups to restrict EC2 access
aws ec2 authorize-security-group-ingress \
  --group-id sg-ec2-app \
  --protocol tcp --port 443 \
  --source-group sg-vpn-endpoint

# 5. Regularly rotate certificates
# Before expiration, generate new certificates and import to ACM
./easyrsa renew-cert client1

# 6. Monitor concurrent connections
aws cloudwatch get-metric-statistics \
  --namespace AWS/ClientVPN \
  --metric-name ActiveClientVpnConnections \
  --dimensions Name=ClientVpnEndpointId,Value=cvpn-id \
  --period 300 \
  --statistics Average
```

---

## Troubleshooting & Monitoring

### Common Issues and Solutions

**Issue 1: VPN Tunnel Down**

```bash
# Check tunnel status
aws ec2 describe-vpn-connections \
  --vpn-connection-ids vpn-id \
  --query 'VpnConnections[0].VgwTelemetry[*].[Status,TunnelAddress]' \
  --output table

# If Status is DOWN:
# 1. Check on-premises VPN device logs
# 2. Verify public IP of customer gateway
# 3. Check pre-shared key (PSK) matches
# 4. Ensure IPsec parameters match (Phase 1 & 2)
# 5. Verify on-premises firewall allows UDP 500, 4500

# Restart tunnel on premises
systemctl restart ipsec  # Linux StrongSwan
systemctl restart openswan  # OpenSwan
```

**Issue 2: Cannot Ping Through VPN**

```bash
# Diagnostic checklist:
# 1. Verify tunnel is UP
aws ec2 describe-vpn-connections \
  --vpn-connection-ids vpn-id

# 2. Check route table has route for on-premises CIDR
aws ec2 describe-route-tables \
  --route-table-ids rtb-id \
  --query 'RouteTables[0].Routes'

# 3. Verify security groups allow ICMP
aws ec2 describe-security-groups \
  --group-ids sg-id \
  --query 'SecurityGroups[0].IpPermissions'

# 4. Check NACLs allow ICMP
aws ec2 describe-network-acls \
  --network-acl-ids nacl-id

# 5. Verify on-premises routing has route back to AWS
# On on-premises gateway:
route -n
# Should show 172.31.0.0/16 -> via VPN
```

**Issue 3: Client VPN Connection Failed**

```bash
# Check endpoint status
aws ec2 describe-client-vpn-endpoints \
  --client-vpn-endpoint-ids cvpn-id \
  --query 'ClientVpnEndpoints[0].Status'

# Check client logs
# Linux: /var/log/openvpn/openvpn.log
# Mac: /Library/Logs/OpenVPN/openvpn.log
# Windows: C:\Program Files\OpenVPN\log\openvpn.log

# Common reasons:
# 1. Certificate expired (regenerate and reimport)
# 2. Authorization rule missing (add with authorize-client-vpn-ingress)
# 3. Security group blocking traffic (add inbound rule)
# 4. Wrong client CIDR block (modify endpoint)

# Verify certificate validity
aws acm describe-certificate \
  --certificate-arn arn:aws:acm:region:account:certificate/id \
  --query 'Certificate.NotAfter'
```

### Monitoring Dashboard

```bash
# Create CloudWatch dashboard for VPN monitoring
aws cloudwatch put-dashboard \
  --dashboard-name VPN-Monitoring \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric",
        "properties": {
          "metrics": [
            ["AWS/ClientVPN", "ClientVpnActiveConnections", {"stat": "Sum"}],
            ["AWS/ClientVPN", "ClientVpnBytesIn", {"stat": "Sum"}],
            ["AWS/ClientVPN", "ClientVpnBytesOut", {"stat": "Sum"}]
          ],
          "period": 300,
          "stat": "Average",
          "region": "us-east-1",
          "title": "Client VPN Metrics"
        }
      }
    ]
  }'
```

---

## Top 5 Interview Questions

### Question 1: Fundamental Difference (Beginner)

**Question:**
"Explain the key difference between Site-to-Site VPN and Client VPN. When would you use each?"

**Model Answer:**

**Site-to-Site VPN:**

**Purpose:** Connect entire networks (on-premises data center to AWS VPC)

**Architecture:**
```
Corporate HQ Network ←→ [Customer Gateway] ←→ [IPsec Tunnel] ←→ [Virtual Private Gateway] ←→ AWS VPC
```

**Characteristics:**
- Network-to-network connectivity
- Requires VPN device on-premises
- Static or dynamic (BGP) routing
- Dual tunnels for redundancy
- Better for stable connections
- Typical use: Branch office permanent connection

**Client VPN:**

**Purpose:** Enable individual users to connect remotely to AWS resources

**Architecture:**
```
Remote User [OpenVPN Client] ←→ [TLS Tunnel] ←→ [Client VPN Endpoint] ←→ VPC Resources
```

**Characteristics:**
- User-to-network connectivity
- Software-based (no appliance needed)
- Certificate or AD authentication
- Multi-AZ automatic failover
- Better for variable connections
- Typical use: Remote workers, contractors

**Side-by-Side Comparison:**

| Aspect | Site-to-Site | Client VPN |
|--------|-------------|-----------|
| Connectivity | Network-to-network | User-to-network |
| Hardware | VPN appliance required | Software client only |
| Users | All users in network | Individual users |
| Scale | Few (1-5 typical) | Thousands concurrent |
| Setup | Complex (hours) | Simple (minutes) |
| Cost | Per connection | Per connection-minute |
| Authentication | PSK (pre-shared key) | Certificates + AD |
| Throughput | High per tunnel | Per-user throttled |

**Decision Tree:**

```
Question: How many locations need connectivity?
  ├─ Few offices/data centers → Site-to-Site VPN
  └─ Individual remote users → Client VPN

Question: Permanent or temporary access?
  ├─ Permanent (always-on) → Site-to-Site VPN
  └─ Temporary/variable → Client VPN

Question: Need to access VPC only or also on-premises?
  ├─ Both → Site-to-Site + Client VPN together
  └─ VPC only → Client VPN sufficient
```

**Real-World Example:**

```
Company Structure:
├─ Headquarters (NYC office) → Site-to-Site VPN
├─ Branch offices (LA, Chicago) → Site-to-Site VPN
├─ Remote workers (WFH) → Client VPN
├─ Contractors (temporary) → Client VPN
└─ On-premises resources accessible through both

Implementation:
1. Set up Site-to-Site VPN for each office
2. Set up Client VPN endpoint for remote access
3. Both connect through same Virtual Private Gateway
4. Route tables handle traffic distribution
```

**Key Insight:**
> "Site-to-Site is like establishing a permanent bridge between two cities, while Client VPN is like giving individual people a ferry ticket to cross."

---

### Question 2: IPsec VPN Configuration (Intermediate)

**Question:**
"Walk me through the IPsec tunnel establishment process for Site-to-Site VPN. What are the two phases?"

**Model Answer:**

**IPsec Overview:**

IPsec (Internet Protocol Security) creates encrypted tunnels with two distinct phases:

**Phase 1: Key Exchange (IKE - Internet Key Exchange)**

**Purpose:** Establish secure channel to exchange encryption keys

**Process:**
```
1. Negotiation
   Customer GW → VGW: "I support IKEv2, AES-256, SHA-256"
   VGW → Customer GW: "Agreement! Using IKEv2, AES-256"

2. Authentication
   Both sides verify identity using Pre-Shared Key (PSK)
   PSK: "my-secret-key-12345"
   
3. Diffie-Hellman Key Exchange
   Both parties generate random numbers
   Exchange public values (not secrets)
   Both independently calculate same shared secret
   This secret becomes ISAKMP SA (Security Association)

4. Result: Secure, authenticated channel established
```

**IKE Parameters:**

```bash
# From AWS configuration file:
ike=aes256-sha256-modp2048

# Breakdown:
# aes256 = Encryption algorithm (AES 256-bit)
# sha256 = Hash algorithm (SHA-256)
# modp2048 = Diffie-Hellman group (2048-bit prime)

# Other options:
# aes128-sha1-modp1024 (older, less secure)
# aes256-sha512-modp4096 (newer, more secure)
```

**Phase 2: Data Encryption (IPsec)**

**Purpose:** Encrypt and transmit actual application traffic

**Process:**
```
1. Agreement
   Customer GW → VGW: "Use AES-256-GCM, SHA-256"
   Both agree on encryption parameters

2. Encryption
   Plaintext packet → Encrypt with agreed algorithm → Send

3. Authentication
   Add authentication tag to verify integrity
   Receiving end verifies tag didn't change in transit

4. Decryption
   Receive encrypted packet → Decrypt → Verify → Deliver

Result: All traffic through tunnel is encrypted
```

**ESP Parameters:**

```bash
# From AWS configuration file:
esp=aes256-sha256

# Breakdown:
# aes256 = Encryption algorithm
# sha256 = Hash algorithm

# Traffic flows:
User Data
  ↓ (Encrypt)
AES-256 encrypted data
  ↓ (Add hash)
AES-256 + SHA-256 MAC
  ↓ (Send)
Over IPsec tunnel
  ↓
Receiving end reverses process
```

**Timeline Example:**

```
Customer GW (10.0.1.1) → VGW (52.1.2.3)

T=0 Phase 1 starts
    Customer GW → VGW: IKE proposal (IKEv2, AES-256-sha256-modp2048)
T=1 VGW → Customer GW: IKE acceptance
T=2 Customer GW ↔ VGW: Diffie-Hellman exchange (DH1, DH2)
T=3 Customer GW ↔ VGW: Authentication (PSK verification)
T=4 Phase 1 complete: ISAKMP SA established

T=5 Phase 2 starts
    Customer GW → VGW: Propose ESP params (AES-256-sha256)
T=6 VGW → Customer GW: Accept ESP params
T=7 Phase 2 complete: IPsec SA established

T=8+ Data flows: All traffic encrypted through tunnel
    Customer: 10.0.1.100 → AWS: 172.31.1.100
    Path: Encrypt → Send over tunnel → Decrypt → Deliver
```

**Tunnel Status:**

```bash
# Check tunnel negotiation status
aws ec2 describe-vpn-connections \
  --vpn-connection-ids vpn-id \
  --query 'VpnConnections[0].VgwTelemetry' \
  --output table

# Output:
# Status: UP (both phases complete)
# Status: DOWN (phase 1 or 2 failed)

# Common failures:
# "No matching encryption" = Phase 1 proposal mismatch
# "PSK mismatch" = Phase 1 authentication failed
# "Rekey failed" = Phase 2 renewal failed
```

**Rekey Process (Periodic):**

```
IKE Rekey (default: 1 hour)
  ├─ New Phase 1 negotiation
  ├─ New ISAKMP SA created
  ├─ Old SA flushed
  └─ Phase 2 continues with new SA

IPsec Rekey (default: 1 hour)
  ├─ New Phase 2 negotiation
  ├─ New encryption keys generated
  ├─ Traffic switches to new keys
  └─ Existing connections not disrupted
```

**Troubleshooting Phase Issues:**

```bash
# Phase 1 failure indicators:
# - "No matching proposals" = Wrong IKE algorithms
# - "Timeout" = Firewall blocking UDP 500

# Check IKE parameters match:
# Customer config: ike=aes256-sha256-modp2048
# AWS config: Must also support aes256-sha256-modp2048

# Fix:
# Update customer gateway config to match AWS IKE params
# Restart IPsec daemon

# Phase 2 failure indicators:
# - "Policy mismatch" = Wrong ESP algorithms
# - "Rekey timeout" = Too aggressive timeout settings

# Check ESP parameters match:
# Customer config: esp=aes256-sha256
# AWS config: Must also support aes256-sha256
```

**Key Insight:**
> "Phase 1 is like shaking hands and agreeing on a secret handshake (IKE), Phase 2 is like using that handshake to unlock the door (encrypt data). Both must happen successfully for tunnel to work."

---

### Question 3: Real-World Hybrid Architecture (Intermediate-Advanced)

**Question:**
"Design a hybrid network where HQ connects to AWS via Site-to-Site VPN, and remote workers connect via Client VPN. How would you route traffic? Show security group and route table configurations."

**Model Answer:**

**Architecture Design:**

```
┌─────────────────────────────────────┐
│ On-Premises HQ (10.0.0.0/16)       │
├─────────────────────────────────────┤
│                                     │
│ ┌──────────────────────────────┐   │
│ │ VPN Gateway Device           │   │
│ │ IP: 203.0.113.12             │   │
│ │ Routes: 10.0.0.0/16 ← local  │   │
│ │         172.31.0.0/16 ← AWS  │   │
│ └──────────────────────────────┘   │
│              ↓                       │
│      IPsec Tunnel                    │
│              ↓                       │
└─────────────────────────────────────┘
          Public Internet
┌─────────────────────────────────────┐
│ AWS VPC (172.31.0.0/16)             │
├─────────────────────────────────────┤
│                                     │
│ ┌───────────────────────────────┐   │
│ │ Virtual Private Gateway       │   │
│ │ (VPN Gateway)                 │   │
│ │ BGP ASN: 64512                │   │
│ │ Routes learned:               │   │
│ │ - 10.0.0.0/16 via S2S VPN    │   │
│ │ - 172.27.0.0/22 via C-VPN    │   │
│ └───────────────────────────────┘   │
│              ↓                       │
│ ┌───────────────────────────────┐   │
│ │ Client VPN Endpoint           │   │
│ │ CIDR: 172.27.0.0/22           │   │
│ │ Auth: Certificates            │   │
│ │ Port: 443 (TLS/OpenVPN)       │   │
│ └───────────────────────────────┘   │
│              ↓                       │
│ ┌───────────────────────────────┐   │
│ │ Private Subnet A              │   │
│ │ 172.31.1.0/24                 │   │
│ │                               │   │
│ │ ┌─────────────────────────┐   │   │
│ │ │ Web Server Tier (sg-web)│   │   │
│ │ │ - HTTP/HTTPS in         │   │   │
│ │ │ - SSH from HQ           │   │   │
│ │ │ - Output to DB tier     │   │   │
│ │ └─────────────────────────┘   │   │
│ └───────────────────────────────┘   │
│              ↓                       │
│ ┌───────────────────────────────┐   │
│ │ Private Subnet B              │   │
│ │ 172.31.2.0/24                 │   │
│ │                               │   │
│ │ ┌─────────────────────────┐   │   │
│ │ │ App Server Tier (sg-app)│   │   │
│ │ │ - Port 8080 from web    │   │   │
│ │ │ - SSH from HQ           │   │   │
│ │ │ - Output to DB tier     │   │   │
│ │ └─────────────────────────┘   │   │
│ └───────────────────────────────┘   │
│              ↓                       │
│ ┌───────────────────────────────┐   │
│ │ Isolated Subnet C             │   │
│ │ 172.31.3.0/24                 │   │
│ │                               │   │
│ │ ┌─────────────────────────┐   │   │
│ │ │ Database Tier (sg-db)   │   │   │
│ │ │ - Port 3306 from app    │   │   │
│ │ │ - SSH from HQ only      │   │   │
│ │ │ - No internet access    │   │   │
│ │ └─────────────────────────┘   │   │
│ └───────────────────────────────┘   │
│                                     │
└─────────────────────────────────────┘
            ↑
    TLS Tunnel (Client VPN)
            ↑
┌─────────────────────────────────────┐
│ Remote Workers                      │
│ - Seattle (WFH)                     │
│ - Toronto (Coffee Shop)             │
│ - New York (Co-working)             │
│ (OpenVPN Client)                    │
└─────────────────────────────────────┘
```

**Route Table Configuration:**

```bash
# Main Route Table (for VPC)
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=vpc-12345678" \
  --query 'RouteTables[?Associations[?Main==true]].Routes' \
  --output table

# Expected routes:
# ┌──────────────────┬──────────┬─────────────────┐
# │ Destination      │ Target   │ Origin          │
# ├──────────────────┼──────────┼─────────────────┤
# │ 172.31.0.0/16    │ local    │ CreateRouteTable│
# │ 10.0.0.0/16      │ vgw-id   │ EnableVgwRoute  │
# │ 172.27.0.0/22    │ vgw-id   │ CreateRoute*    │
# └──────────────────┴──────────┴─────────────────┘

# Manual route for Client VPN traffic
aws ec2 create-route \
  --route-table-id rtb-main \
  --destination-cidr-block 172.27.0.0/22 \
  --gateway-id vgw-id
```

**Security Group Configuration:**

```bash
#!/bin/bash

# Create security groups
SG_WEB=$(aws ec2 create-security-group \
  --group-name sg-web \
  --description "Web tier" \
  --vpc-id vpc-12345678 \
  --query 'GroupId' --output text)

SG_APP=$(aws ec2 create-security-group \
  --group-name sg-app \
  --description "App tier" \
  --vpc-id vpc-12345678 \
  --query 'GroupId' --output text)

SG_DB=$(aws ec2 create-security-group \
  --group-name sg-db \
  --description "Database tier" \
  --vpc-id vpc-12345678 \
  --query 'GroupId' --output text)

SG_VPN=$(aws ec2 create-security-group \
  --group-name sg-vpn-endpoint \
  --description "VPN Endpoint" \
  --vpc-id vpc-12345678 \
  --query 'GroupId' --output text)

echo "Security Groups created"

# ─────────────────────────────────
# Web Tier (sg-web)
# ─────────────────────────────────

# Inbound: HTTP from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id $SG_WEB \
  --protocol tcp --port 80 --cidr 0.0.0.0/0

# Inbound: HTTPS from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id $SG_WEB \
  --protocol tcp --port 443 --cidr 0.0.0.0/0

# Inbound: SSH from HQ (via S2S VPN)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_WEB \
  --protocol tcp --port 22 --cidr 10.0.0.0/16

# Inbound: SSH from Remote Workers (via C-VPN)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_WEB \
  --protocol tcp --port 22 --cidr 172.27.0.0/22

# Outbound: Port 8080 to App tier (implicit egress to app SG)
aws ec2 authorize-security-group-egress \
  --group-id $SG_WEB \
  --protocol tcp --port 8080 \
  --destination-group $SG_APP

# ─────────────────────────────────
# App Tier (sg-app)
# ─────────────────────────────────

# Inbound: Port 8080 from Web tier
aws ec2 authorize-security-group-ingress \
  --group-id $SG_APP \
  --protocol tcp --port 8080 \
  --source-group $SG_WEB

# Inbound: SSH from HQ (via S2S VPN)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_APP \
  --protocol tcp --port 22 --cidr 10.0.0.0/16

# Inbound: SSH from Remote Workers (via C-VPN)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_APP \
  --protocol tcp --port 22 --cidr 172.27.0.0/22

# Outbound: Port 3306 to DB tier
aws ec2 authorize-security-group-egress \
  --group-id $SG_APP \
  --protocol tcp --port 3306 \
  --destination-group $SG_DB

# ─────────────────────────────────
# Database Tier (sg-db)
# ─────────────────────────────────

# Inbound: MySQL from App tier only
aws ec2 authorize-security-group-ingress \
  --group-id $SG_DB \
  --protocol tcp --port 3306 \
  --source-group $SG_APP

# Inbound: SSH from HQ only (NOT from remote workers)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_DB \
  --protocol tcp --port 22 --cidr 10.0.0.0/16

# NO outbound access (locked down)
aws ec2 revoke-security-group-egress \
  --group-id $SG_DB \
  --protocol -1 --cidr 0.0.0.0/0

echo "✓ Security Groups configured"
```

**Routing Decision Matrix:**

```
Source → Destination | Route | Via
─────────────────────────────────────────
HQ (10.0.0.0/16) → Web (172.31.1.0) | 10.0.0.0/16 → VGW | S2S VPN
HQ (10.0.0.0/16) → App (172.31.2.0) | 10.0.0.0/16 → VGW | S2S VPN
HQ (10.0.0.0/16) → DB (172.31.3.0)  | 10.0.0.0/16 → VGW | S2S VPN

Remote (172.27.0.0/22) → Web | 172.27.0.0/22 → VGW | C-VPN
Remote (172.27.0.0/22) → App | 172.27.0.0/22 → VGW | C-VPN
Remote (172.27.0.0/22) → DB  | 172.27.0.0/22 → VGW | C-VPN

Web → App | 172.31.2.0/24 → Local | Direct (same VPC)
App → DB  | 172.31.3.0/24 → Local | Direct (same VPC)

VPC → Internet | 0.0.0.0/0 → IGW | Internet Gateway
VPC → HQ       | 10.0.0.0/16 → VGW | S2S VPN
```

**Traffic Flow Examples:**

**Example 1: HQ user accesses Web server**

```
1. HQ PC (10.0.2.100) opens browser
2. Request: 10.0.2.100 → 172.31.1.10 (Web server)
3. Packet reaches on-premises gateway
4. VPN device checks route table:
   172.31.1.10 → via S2S VPN tunnel
5. Encrypt packet → Send to AWS VGW
6. AWS receives encrypted packet
7. VGW decrypts → 10.0.2.100 → 172.31.1.10
8. Route table: Source=10.0.0.0/16 → Local delivery
9. Web server receives request
10. Web server responds: 172.31.1.10 → 10.0.2.100
11. Reverse process through VPN tunnel
12. HQ PC receives response

Security: SSH from HQ (10.0.0.0/16) to Web SG → ALLOWED
```

**Example 2: Remote worker accesses Database**

```
1. Remote worker (coffee shop, 203.45.67.89)
2. Opens OpenVPN client → connects to CVPN endpoint
3. Gets IP from Client CIDR: 172.27.0.50
4. Query: 172.27.0.50 → 172.31.3.10 (DB)
5. Client VPN endpoint routes to associated subnets
6. Route table: Source=172.27.0.0/22 → Local delivery
7. Packet reaches DB security group
8. DB SG: Port 3306 from 172.27.0.0/22 → ALLOWED
9. Database receives query
10. Response sent back through tunnel
11. Remote worker receives response

Security: SSH from Remote (172.27.0.0/22) to DB SG
         → NOT allowed (only 10.0.0.0/16 allowed)
```

**Implementation Steps:**

```bash
#!/bin/bash

# 1. Create Route Tables
aws ec2 create-route-table --vpc-id vpc-12345678

# 2. Add routes
aws ec2 create-route \
  --route-table-id rtb-id \
  --destination-cidr-block 10.0.0.0/16 \
  --gateway-id vgw-id

aws ec2 create-route \
  --route-table-id rtb-id \
  --destination-cidr-block 172.27.0.0/22 \
  --gateway-id vgw-id

# 3. Associate subnets
aws ec2 associate-route-table \
  --route-table-id rtb-id \
  --subnet-id subnet-web

# 4. Enable VGW route propagation
aws ec2 enable-vgw-route-propagation \
  --route-table-id rtb-id \
  --gateway-id vgw-id

# 5. Verify configuration
aws ec2 describe-route-tables --route-table-ids rtb-id \
  --query 'RouteTables[0].Routes' --output table
```

**Key Design Principles:**

1. **Layered Access:** Database only accessible from App tier (principle of least privilege)
2. **Dual Access Paths:** Both HQ and Remote workers can access Web/App tiers
3. **Restricted DB Access:** HQ SSH allowed to DB; Remote workers NOT allowed
4. **Route Propagation:** VGW routes automatically added to route tables
5. **Encryption:** All traffic through VPN tunnels encrypted
6. **Scalability:** Add new remote users without modifying security groups

**Key Insight:**
> "The route table guides where traffic goes, security groups guard who gets there. Together they create secure multi-layered access."

---

### Question 4: Troubleshooting VPN Connection Issues (Advanced)

**Question:**
"Your Site-to-Site VPN is up but ping between HQ and AWS fails. Walk me through your troubleshooting steps to diagnose the problem."

**Model Answer:**

**Troubleshooting Flowchart:**

```
Symptom: Ping fails (HQ 10.0.0.1 → AWS 172.31.1.100)

Step 1: Verify VPN Tunnel Status
  ├─ Check if tunnel is UP
  │ └─ FAIL → Go to Step 2a (Tunnel Down)
  │ └─ PASS → Continue to Step 3
  │
  └─ Tunnel UP means:
     ├─ IPsec Phase 1 & 2 negotiated ✓
     ├─ Encryption established ✓
     └─ But...

Step 2a: Tunnel is DOWN - Diagnose Phase Failure
  ├─ Check Phase 1 (IKE)
  ├─ Check Phase 2 (ESP)
  └─ Fix and restart

Step 3: Verify Route Tables
  ├─ Does HQ route exist in on-premises table?
  ├─ Does AWS route exist in AWS table?
  └─ Continue if both present

Step 4: Check Security Groups
  ├─ Does AWS SG allow ICMP inbound?
  ├─ Does HQ firewall allow ICMP outbound?
  └─ Continue if both allow

Step 5: Verify Network ACLs
  ├─ Do NACLs allow ICMP?
  ├─ Do NACLs allow ephemeral response ports?
  └─ Continue if all allow

Step 6: Test DNS Resolution
  ├─ Can HQ resolve AWS instance name?
  ├─ Can AWS resolve HQ server name?
  └─ Continue if both resolve

Step 7: Deep Packet Analysis
  ├─ Run tcpdump on HQ VPN device
  ├─ Run tcpdump on AWS instance
  └─ Compare packet flow

Result: Identify exact failure point
```

**Step-by-Step Diagnosis:**

**Step 1: Check Tunnel Status**

```bash
# Check VPN connection status
aws ec2 describe-vpn-connections \
  --vpn-connection-ids vpn-0abc1def2ghi3jkl4 \
  --query 'VpnConnections[0].{
    State: State,
    Tunnel1: VgwTelemetry[0].[Status,TunnelAddress,OutsideIpAddress],
    Tunnel2: VgwTelemetry[1].[Status,TunnelAddress,OutsideIpAddress]
  }' \
  --output table

# Expected output:
# State: available
# Tunnel1: [UP, 52.1.2.3, 52.1.2.3]
# Tunnel2: [UP, 52.2.2.3, 52.2.2.3]

# If either tunnel is DOWN:
# Likely cause: IPsec negotiation failed

# On-premises VPN device logs
# Linux StrongSwan:
tail -f /var/log/auth.log | grep -i ipsec
# Look for: "negotiation failed", "no matching", "timeout"

# Linux OpenSwan:
tail -f /var/log/pluto.log
# Look for: "cannot authenticate", "timeout", "mismatch"
```

**If Tunnel Down - Diagnose:**

```bash
# Issue 1: Public IP mismatch
# Your CGW registered with one IP, actual is different

aws ec2 describe-customer-gateways \
  --customer-gateway-ids cgw-id \
  --query 'CustomerGateways[0].{
    RegisteredIP: PublicIp,
    State: State
  }' \
  --output table

# If actual IP different from registered:
aws ec2 create-customer-gateway \
  --type ipsec.1 \
  --bgp-asn 65000 \
  --public-ip 203.0.113.45  # NEW IP
  --query 'CustomerGateway.CustomerGatewayId'

# Issue 2: PSK mismatch
# Check on-premises config
cat /etc/ipsec.d/aws-vpn.secrets
# Should match AWS config exactly

# Issue 3: IKE/ESP parameters mismatch
# On-premises config:
# ike=aes256-sha256-modp2048
# esp=aes256-sha256

# AWS config (in download file):
# Should be identical

# Fix: Update config and restart
systemctl restart ipsec  # or strongswan / openswan
```

**Step 2: Verify Route Tables**

```bash
# On-premises routing table
# (SSH to VPN device)
ip route show

# Expected:
# default via 203.0.113.1 dev eth0
# 10.0.0.0/16 dev eth0 proto kernel scope link src 10.0.1.1
# 172.31.0.0/16 via 10.0.1.254 dev eth0
#   ↑ This route sends AWS traffic through VPN

# AWS route table
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=vpc-12345678" \
  --query 'RouteTables[0].Routes[*].[
    DestinationCidrBlock,
    GatewayId,
    Origin
  ]' \
  --output table

# Expected:
# 172.31.0.0/16    | local            | CreateRouteTable
# 10.0.0.0/16      | vgw-id           | EnableVgwRoutePropagation
#   ↑ This route sends HQ traffic through VGW

# If AWS route missing:
aws ec2 create-vpn-connection-route \
  --vpn-connection-id vpn-id \
  --destination-cidr-block 10.0.0.0/16

# Or check if using BGP (dynamic routes)
aws ec2 describe-vpn-connections \
  --vpn-connection-ids vpn-id \
  --query 'VpnConnections[0].Options.StaticRoutesOnly'
# If false: Using BGP, routes should auto-populate
# If true: Need manual routes
```

**Step 3: Check Security Groups**

```bash
# Get instance security group
INSTANCE_SG=$(aws ec2 describe-instances \
  --instance-ids i-12345678 \
  --query 'Reservations[0].Instances[0].SecurityGroups[0].GroupId' \
  --output text)

# Check inbound rules
aws ec2 describe-security-groups \
  --group-ids $INSTANCE_SG \
  --query 'SecurityGroups[0].IpPermissions' \
  --output table

# Expected: Rule allowing ICMP (protocol -1 or protocol 1)
# From: 10.0.0.0/16 (HQ CIDR)

# If missing, add it:
aws ec2 authorize-security-group-ingress \
  --group-id $INSTANCE_SG \
  --protocol icmp \
  --port -1 \
  --cidr 10.0.0.0/16

# Check outbound rules
aws ec2 describe-security-groups \
  --group-ids $INSTANCE_SG \
  --query 'SecurityGroups[0].IpPermissionsEgress' \
  --output table

# Expected: Allow all outbound (default)
# Or specifically allow ICMP to 10.0.0.0/16
```

**Step 4: Check Network ACLs**

```bash
# Get instance subnet
SUBNET_ID=$(aws ec2 describe-instances \
  --instance-ids i-12345678 \
  --query 'Reservations[0].Instances[0].SubnetId' \
  --output text)

# Get NACL for subnet
NACL_ID=$(aws ec2 describe-network-acls \
  --filters "Name=association.subnet-id,Values=$SUBNET_ID" \
  --query 'NetworkAcls[0].NetworkAclId' \
  --output text)

# Check NACL rules
aws ec2 describe-network-acls \
  --network-acl-ids $NACL_ID \
  --query 'NetworkAcls[0].Entries' \
  --output table

# Expected inbound rules:
# Rule# 100: ICMP from 10.0.0.0/16 → ALLOW
# Rule# 110: Ephemeral (1024-65535) from 10.0.0.0/16 → ALLOW

# Expected outbound rules:
# Rule# 100: ICMP to 10.0.0.0/16 → ALLOW
# Rule# 110: Ephemeral (1024-65535) to 10.0.0.0/16 → ALLOW

# If missing, add them:
aws ec2 create-network-acl-entry \
  --network-acl-id $NACL_ID \
  --rule-number 100 \
  --protocol icmp \
  --port-range From=-1,To=-1 \
  --cidr-block 10.0.0.0/16 \
  --ingress
```

**Step 5: Test with Traceroute**

```bash
# From HQ, trace path to AWS
traceroute 172.31.1.100

# Expected output:
# 1  10.0.1.1 (gateway)
# 2  10.0.1.254 (VPN device)
# 3  52.1.2.3 (AWS VGW tunnel)
# 4  172.31.1.100 (destination)

# If hop 3 times out: VPN device issue
# If hop 4 times out: AWS instance unreachable

# From AWS, trace back to HQ
traceroute 10.0.2.100

# Should show reverse path
```

**Step 6: Packet Capture Analysis**

```bash
# On HQ VPN device, capture traffic
tcpdump -i eth0 -n 'src 10.0.0.1 or dst 10.0.0.1' -w hq.pcap

# On AWS instance, capture traffic
sudo tcpdump -i eth0 -n 'src 172.31.1.100 or dst 172.31.1.100' -w aws.pcap

# Analyze HQ capture
tcpdump -r hq.pcap -n

# Look for:
# 10.0.2.100 > 172.31.1.100: ICMP echo request
# Should see encrypted packets to VPN device

# Analyze AWS capture
tcpdump -r aws.pcap -n

# Look for:
# 172.31.1.100 < 10.0.2.100: ICMP echo reply
# If no reply seen: Instance didn't receive request

# No packets in AWS capture = Network problem before instance
# Packets in AWS but no response = Security group/NACL issue
```

**Complete Troubleshooting Script:**

```bash
#!/bin/bash

VPN_ID="vpn-0abc1def2ghi3jkl4"
INSTANCE_ID="i-12345678"

echo "=== VPN Troubleshooting Report ==="
echo

echo "1. VPN Tunnel Status"
aws ec2 describe-vpn-connections \
  --vpn-connection-ids $VPN_ID \
  --query 'VpnConnections[0].VgwTelemetry[*].[Status,TunnelAddress]' \
  --output table

echo
echo "2. Route Tables"
aws ec2 describe-route-tables \
  --query 'RouteTables[*].[RouteTableId, Routes[*].[DestinationCidrBlock,GatewayId,Origin]]' \
  --output table

echo
echo "3. Security Groups"
SG=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].SecurityGroups[0].GroupId' \
  --output text)

aws ec2 describe-security-groups \
  --group-ids $SG \
  --query 'SecurityGroups[0].[GroupName, IpPermissions, IpPermissionsEgress]' \
  --output table

echo
echo "4. Network ACLs"
SUBNET=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].SubnetId' \
  --output text)

NACL=$(aws ec2 describe-network-acls \
  --filters "Name=association.subnet-id,Values=$SUBNET" \
  --query 'NetworkAcls[0].NetworkAclId' \
  --output text)

aws ec2 describe-network-acls \
  --network-acl-ids $NACL \
  --query 'NetworkAcls[0].Entries' \
  --output table

echo "=== End Report ==="
```

**Common Root Causes & Solutions:**

| Symptom | Cause | Solution |
|---------|-------|----------|
| Both tunnels DOWN | IKE Phase 1 failure | Verify PSK, IKE params, public IP |
| One tunnel DOWN | One tunnel failure | Check backup tunnel params |
| Routes missing | BGP not propagating | Enable route propagation manually |
| Routes present but no ping | SG rule missing | Add ICMP inbound to SG |
| Tunnel UP but connection timeouts | NACL rules missing | Add ICMP and ephemeral port rules |
| DNS not resolving | No DNS rule in NACL | Allow UDP 53 in NACL |
| Intermittent connectivity | Tunnel flapping | Check DPD settings, network stability |

**Key Insight:**
> "VPN tunnel UP doesn't mean data flows. It's like having a bridge built but locked gates. Check tunnel→routes→security→NACLs methodically."

---

### Question 5: VPN Architecture Design for Multi-Region (Advanced)

**Question:**
"Design a multi-region AWS setup where two regions need to communicate securely, and remote users need access to both regions. Should you use VPN or is there a better solution?"

**Model Answer:**

**Requirements Analysis:**

```
Scenario:
├─ Production in us-east-1 (Virginia)
├─ Disaster Recovery in us-west-2 (Oregon)
├─ Remote workers worldwide
├─ On-premises HQ in New York
├─ Data must be encrypted
└─ RTO: < 15 minutes, RPO: < 1 hour
```

**Solution Options:**

**Option 1: VPN Only (NOT Recommended)**

```
Cons:
├─ Site-to-Site VPN from HQ to us-east-1
├─ Site-to-Site VPN from HQ to us-west-2
├─ Client VPN in both regions
├─ Redundancy complex
├─ Cross-region communication via internet (latency)
└─ Not ideal for inter-region replication

Architecture:
HQ ←→ VPN ←→ us-east-1 VPC
HQ ←→ VPN ←→ us-west-2 VPC
us-east-1 ←→ Internet ←→ us-west-2 (unencrypted or complex tunnel)
```

**Option 2: VPC Peering + VPN (Better)**

```
Pros:
├─ Direct VPC-to-VPC communication (us-east-1 ↔ us-west-2)
├─ Low latency for replication
├─ Automatic failover support
├─ Single S2S VPN to HQ
├─ All remote users via Client VPN
└─ Simple architecture

Cons:
├─ Inter-region peering costs data transfer
├─ Separate Client VPN endpoints in each region
└─ Manual failover between regions

Architecture:
HQ ← S2S VPN → us-east-1 VPC ↔ VPC Peering ↔ us-west-2 VPC
                   ↑                                  ↑
              Client VPN                        Client VPN
              (Region 1)                        (Region 2)
```

**Option 3: AWS Transit Gateway + VPN (RECOMMENDED)**

```
Pros:
├─ Centralized hub-and-spoke architecture
├─ Single S2S VPN attachment
├─ Single Client VPN endpoint can reach both regions (via TGW)
├─ Automatic failover
├─ Simplified routing
├─ Better bandwidth utilization
├─ Support multiple VPCs and on-premises networks
└─ Scales easily

Architecture:

        ┌─────────────────────────────────────┐
        │ AWS Transit Gateway (TGW)           │
        │ (Central Hub)                       │
        │ - ASN: 64512                        │
        │ - CIDR: 100.64.0.0/16               │
        └─────────────────────────────────────┘
                ↑         ↑         ↑
        ┌───────┘         │         └───────┐
        ↓                 ↓                 ↓
   us-east-1 VPC    On-Premises       us-west-2 VPC
   (172.31.0.0/16)    (10.0.0.0/8)     (172.32.0.0/16)
        ↓              (S2S VPN)             ↓
    EC2, RDS                           EC2, RDS
```

**Recommended Architecture: Transit Gateway Setup**

**Step 1: Create Transit Gateway**

```bash
#!/bin/bash

# Create Transit Gateway
TGW=$(aws ec2 create-transit-gateway \
  --description "Multi-Region Hub" \
  --options AmazonSideAsn=64512,DefaultRouteTableAssociation=enable,DefaultRouteTablePropagation=enable \
  --region us-east-1 \
  --query 'TransitGateway.TransitGatewayId' \
  --output text)

echo "Transit Gateway created: $TGW"

# Wait for TGW to be available
aws ec2 wait transit-gateway-available \
  --transit-gateway-ids $TGW \
  --region us-east-1

# Create route table in TGW
TGW_RTB=$(aws ec2 create-transit-gateway-route-table \
  --transit-gateway-id $TGW \
  --region us-east-1 \
  --query 'TransitGatewayRouteTable.TransitGatewayRouteTableId' \
  --output text)

echo "TGW Route Table: $TGW_RTB"
```

**Step 2: Attach VPCs to Transit Gateway**

```bash
# Region 1: us-east-1
VPC_1="vpc-12345678"  # 172.31.0.0/16
SUBNET_1="subnet-11111111"
SUBNET_2="subnet-22222222"

TGW_ATTACH_1=$(aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id $TGW \
  --vpc-id $VPC_1 \
  --subnet-ids $SUBNET_1 $SUBNET_2 \
  --options AssociateWithTransitGateway=true \
  --region us-east-1 \
  --query 'TransitGatewayVpcAttachment.TransitGatewayAttachmentId' \
  --output text)

echo "VPC 1 attached: $TGW_ATTACH_1"

# Region 2: us-west-2
VPC_2="vpc-87654321"  # 172.32.0.0/16
SUBNET_3="subnet-33333333"
SUBNET_4="subnet-44444444"

TGW_ATTACH_2=$(aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id $TGW \
  --vpc-id $VPC_2 \
  --subnet-ids $SUBNET_3 $SUBNET_4 \
  --options AssociateWithTransitGateway=true \
  --region us-west-2 \
  --query 'TransitGatewayVpcAttachment.TransitGatewayAttachmentId' \
  --output text)

echo "VPC 2 attached: $TGW_ATTACH_2"

# Enable resource sharing for cross-region attachment
aws ram create-resource-share \
  --name "TGW-Cross-Region" \
  --resource-arns "arn:aws:ec2:us-east-1:123456789012:transit-gateway/$TGW" \
  --principals "arn:aws:organizations::123456789012:organization/o-1234567890" \
  --region us-east-1
```

**Step 3: Attach Site-to-Site VPN to Transit Gateway**

```bash
# Create customer gateway
CGW=$(aws ec2 create-customer-gateway \
  --type ipsec.1 \
  --bgp-asn 65000 \
  --public-ip 203.0.113.12 \
  --region us-east-1 \
  --query 'CustomerGateway.CustomerGatewayId' \
  --output text)

echo "Customer Gateway: $CGW"

# Create VPN connection to TGW
VPN=$(aws ec2 create-vpn-connection \
  --type ipsec.1 \
  --customer-gateway-id $CGW \
  --transit-gateway-id $TGW \
  --region us-east-1 \
  --query 'VpnConnection.VpnConnectionId' \
  --output text)

echo "VPN Connection: $VPN"

# Download configuration
aws ec2 describe-vpn-connections \
  --vpn-connection-ids $VPN \
  --query 'VpnConnections[0].CustomerGatewayConfiguration' \
  --output text > tgw-vpn-config.xml

echo "VPN Config saved"
```

**Step 4: Create Client VPN with Transit Gateway**

```bash
# Create Client VPN endpoint (attached to TGW)
CVPN=$(aws ec2 create-client-vpn-endpoint \
  --client-cidr-block 172.27.0.0/22 \
  --server-certificate-arn arn:aws:acm:us-east-1:123456789012:certificate/... \
  --authentication-options Type=certificate-authentication,MutualAuthentication.ClientRootCertificateChainArn=arn:aws:acm:... \
  --connection-log-options CloudwatchLogGroup=/aws/clientvpn,Enabled=true \
  --transit-gateway-id $TGW \
  --security-group-ids sg-vpn \
  --tag-specifications ResourceType=client-vpn-endpoint,Tags=[{Key=Name,Value=multi-region-vpn}] \
  --region us-east-1 \
  --query 'ClientVpnEndpointId' \
  --output text)

echo "Client VPN Endpoint: $CVPN"

# No need to separately associate target networks (TGW handles it)
# Client VPN automatically gets access to all VPCs attached to TGW

# Add authorization rules
aws ec2 authorize-client-vpn-ingress \
  --client-vpn-endpoint-id $CVPN \
  --target-network-cidr 172.31.0.0/16 \
  --authorize-all-groups \
  --description "Access to Region 1 (us-east-1)" \
  --region us-east-1

aws ec2 authorize-client-vpn-ingress \
  --client-vpn-endpoint-id $CVPN \
  --target-network-cidr 172.32.0.0/16 \
  --authorize-all-groups \
  --description "Access to Region 2 (us-west-2)" \
  --region us-east-1

aws ec2 authorize-client-vpn-ingress \
  --client-vpn-endpoint-id $CVPN \
  --target-network-cidr 10.0.0.0/8 \
  --authorize-all-groups \
  --description "Access to On-Premises" \
  --region us-east-1
```

**Step 5: Configure Routing in Transit Gateway**

```bash
# Create static routes in TGW for remote networks
aws ec2 create-transit-gateway-route \
  --transit-gateway-route-table-id $TGW_RTB \
  --destination-cidr-block 10.0.0.0/8 \
  --transit-gateway-attachment-id $VPN_ATTACHMENT_ID \
  --region us-east-1

# Enable route propagation
aws ec2 enable-transit-gateway-route-table-propagation \
  --transit-gateway-route-table-id $TGW_RTB \
  --transit-gateway-attachment-id $TGW_ATTACH_1 \
  --region us-east-1

aws ec2 enable-transit-gateway-route-table-propagation \
  --transit-gateway-route-table-id $TGW_RTB \
  --transit-gateway-attachment-id $TGW_ATTACH_2 \
  --region us-east-1

# Verify routes
aws ec2 search-transit-gateway-routes \
  --transit-gateway-route-table-id $TGW_RTB \
  --filters Name=state,Values=active \
  --region us-east-1 \
  --query 'Routes[*].[DestinationCidrBlock,Type,State]' \
  --output table
```

**Final Architecture:**

```
┌───────────────────────────────────────────────────────┐
│ On-Premises HQ (New York)                            │
│ (10.0.0.0/8)                                         │
│ ┌────────────────────────────────────────────────┐   │
│ │ VPN Device (203.0.113.12)                      │   │
│ └────────────────────────────────────────────────┘   │
│                     ↓                                  │
│ ────────── IPsec S2S VPN (Encrypted) ──────────       │
│                     ↓                                  │
└───────────────────────────────────────────────────────┘
                      ↓
┌───────────────────────────────────────────────────────┐
│ AWS Transit Gateway (us-east-1)                       │
│ - Central routing hub                                 │
│ - BGP ASN: 64512                                      │
│ - Auto-scaling, multi-AZ                             │
│                                                       │
│ ┌──────────────────────────────────────────────────┐  │
│ │ TGW Route Table                                  │  │
│ ├──────────────┬─────────────────┬────────────────┤  │
│ │ CIDR         │ Type            │ Attachment     │  │
│ ├──────────────┼─────────────────┼────────────────┤  │
│ │ 172.31.0/16  │ propagated      │ VPC Att. 1    │  │
│ │ 172.32.0/16  │ propagated      │ VPC Att. 2    │  │
│ │ 10.0.0.0/8   │ static          │ VPN Att.      │  │
│ │ 172.27.0/22  │ dynamic         │ Client VPN    │  │
│ └──────────────┴─────────────────┴────────────────┘  │
│                  /        │        \                  │
│    ┌────────────┘         │         └──────────────┐  │
│    ↓                      ↓                        ↓  │
└────┼──────────────────────┼────────────────────────┼──┘
     │                      │                        │
     ↓                      ↓                        ↓
┌─────────────────┐  ┌──────────────────┐  ┌────────────────────┐
│ us-east-1 VPC   │  │ Client VPN       │  │ us-west-2 VPC      │
│ 172.31.0.0/16   │  │ Endpoint         │  │ 172.32.0.0/16      │
├─────────────────┤  │ (Multi-region)   │  ├────────────────────┤
│ ┌─────────────┐ │  │ 172.27.0.0/22    │  │ ┌──────────────┐   │
│ │ Web Tier    │ │  │ ┌──────────────┐ │  │ │ DB Tier      │   │
│ │ EC2, ALB    │ │  │ │ Remote Users │ │  │ │ RDS, Cache   │   │
│ │ App Tier    │ │  │ │ Access all 3:│ │  │ │ Aurora       │   │
│ │ EC2, Batch  │ │  │ │ ✓ HQ via VPN │ │  │ │ Standby DB   │   │
│ │ App Logic   │ │  │ │ ✓ Both VPCs  │ │  │ │ Standby App  │   │
│ └─────────────┘ │  │ └──────────────┘ │  │ └──────────────┘   │
│ ┌─────────────┐ │  │                  │  │ ┌──────────────┐   │
│ │ Primary DB  │ │  │                  │  │ │ Read Replica │   │
│ │ RDS Aurora  │ │  │                  │  │ │ (Standby)    │   │
│ │ Master      │ │  │                  │  │ │ For failover │   │
│ └─────────────┘ │  │                  │  │ └──────────────┘   │
│                 │  │                  │  │                    │
└─────────────────┘  └──────────────────┘  └────────────────────┘
     ↓                                              ↓
 S3 Bucket                              S3 Cross-Region Replication
 (Primary)                                    (Backup)
     ↓                                              ↓
  CloudFront Distribution (Global)
```

**Traffic Flow Examples:**

**Flow 1: Remote Worker → Primary Region**
```
Remote Worker (San Francisco) → OpenVPN Client → Client VPN Endpoint
  → Transit Gateway → us-east-1 VPC → EC2 Instance
  → Response reversed through same path
  Encryption: TLS (Client VPN) end-to-end
```

**Flow 2: HQ → Secondary Region**
```
HQ (New York) → S2S VPN → Transit Gateway → us-west-2 VPC
  → RDS Aurora (read-only standby) → Response reversed
  Encryption: IPsec end-to-end
```

**Flow 3: Region-to-Region Replication**
```
us-east-1 Primary DB → (AWS private backbone) → us-west-2 Secondary
  Path: Through Transit Gateway (optimized AWS network)
  Encryption: AWS managed (in-transit encryption optional)
```

**Benefits of This Architecture:**

```
✓ Single entry point (Transit Gateway)
✓ Redundancy: Multi-AZ, multi-region
✓ Scalability: Auto-scales with demand
✓ Simplicity: Single control plane
✓ Cost-efficient: Consolidated routing
✓ Flexibility: Easy to add new VPCs/regions
✓ Security: All traffic encrypted (VPN + internal AWS)
✓ Performance: AWS backbone for inter-region
✓ Compliance: Meets HIPAA, PCI-DSS requirements
✓ Failover: Automatic switching between regions
```

**Failover Scenario:**

```
Normal:
  Primary (us-east-1) → Active
  Secondary (us-west-2) → Standby (reading from Aurora replica)
  
If us-east-1 fails:
  1. Application detects primary unavailable
  2. Promotes Aurora read replica to primary in us-west-2
  3. Updates Route53 to point to us-west-2 endpoint
  4. All traffic (VPN + Client VPN) routes to us-west-2
  5. Users don't need to reconnect (automatic)
  6. RTO < 5 minutes, RPO < 1 minute (achieved!)
```

**Cost Analysis:**

```
Transit Gateway costs:
├─ VPC attachments: $0.05/hour per attachment
├─ On-premises attachments: $0.05/hour each
├─ Data transfer: $0.02 per GB (inter-region cheaper than NAT)
└─ Total: ~$200-300/month

Alternative (VPC Peering):
├─ VPC Peering: Free (same region) / $0.01/GB (cross-region)
├─ Multiple VPNs: Multiple tunnels (redundant cost)
├─ Client VPN per region: Doubled costs
└─ Total: ~$400-600/month (more expensive)

Savings: ~40% with Transit Gateway
```

**Key Insight:**
> "Transit Gateway is the postal service for AWS networks. Instead of point-to-point roads (VPC Peering), it's a central hub where all mail (traffic) gets sorted and routed efficiently. Perfect for multi-region, multi-VPC at scale."

---

## Summary

AWS VPN provides two complementary solutions:

**Site-to-Site VPN:**
- Network-to-network connectivity (HQ ↔ AWS)
- Uses IPsec encryption
- Requires hardware VPN device
- Best for permanent connections

**Client VPN:**
- Individual user remote access
- Uses TLS/OpenVPN encryption
- Software-based (no appliance)
- Best for flexible, scalable access

**Multi-Region Best Practice:**
- Use AWS Transit Gateway as central hub
- Attach both VPCs and on-premises via TGW
- Single Client VPN endpoint for all regions
- Automatic failover and high availability

**Security & Compliance:**
- All traffic encrypted (IPsec for S2S, TLS for Client)
- Meets HIPAA, PCI-DSS, SOC2 requirements
- Network isolation from public internet
- Audit logging and monitoring available

---

**Last Updated:** December 2, 2025  
**Version:** 2.0  
**Comprehensive Coverage:** All aspects of AWS VPN with real-world examples, CLI commands, architecture patterns, and interview questions.
