
# Practical 03 — Set Up a Virtual Private Cloud (VPC)

---

## 📌 Objective

Create and configure a custom AWS VPC with public and private subnets, configure an Internet Gateway for the public subnet, and use a NAT Gateway to provide internet access for private subnet instances.

---

## 🧠 Conceptual Background (Know-How)

### What is a VPC?

A **Virtual Private Cloud (VPC)** is a logically isolated section of the AWS cloud where you can launch AWS resources in a network that you define. It gives you complete control over your virtual networking environment, including IP address ranges, subnets, routing tables, and gateways.

Think of a VPC like your own **private data center** inside AWS, with your own IP address space, fully isolated from other customers.

### VPC Core Architecture

```
                        ┌─────────────────────────────────────────┐
                        │           AWS Cloud (us-east-1)          │
                        │                                          │
                        │  ┌──────────────────────────────────┐   │
                        │  │    VPC: 10.0.0.0/16              │   │
                        │  │                                  │   │
   Internet ──────────  │  │  ┌─────────────────────────┐    │   │
                   │    │  │  │  Public Subnet           │    │   │
                   │    │  │  │  10.0.1.0/24 (AZ-a)     │    │   │
                   ▼    │  │  │                          │    │   │
            ┌──────────┐│  │  │  ┌──────────┐           │    │   │
            │ Internet ││  │  │  │  EC2     │           │    │   │
            │ Gateway  ││  │  │  │ (public) │           │    │   │
            └──────────┘│  │  │  └──────────┘           │    │   │
                   │    │  │  │  ┌──────────┐           │    │   │
                   │    │  │  │  │  NAT GW  │           │    │   │
                   │    │  │  │  └────┬─────┘           │    │   │
                   │    │  │  └───────│─────────────────┘    │   │
                   │    │  │          │                       │   │
                   │    │  │  ┌───────▼──────────────────┐   │   │
                   │    │  │  │  Private Subnet           │   │   │
                   │    │  │  │  10.0.2.0/24 (AZ-a)      │   │   │
                   │    │  │  │                           │   │   │
                   │    │  │  │  ┌──────────┐            │   │   │
                   │    │  │  │  │  EC2     │            │   │   │
                   │    │  │  │  │(private) │            │   │   │
                   │    │  │  │  └──────────┘            │   │   │
                   │    │  │  └───────────────────────────┘   │   │
                   │    │  └──────────────────────────────────┘   │
                   │    └──────────────────────────────────────────┘
```

### Key VPC Components

| Component | Description |
|---|---|
| **VPC** | The isolated network container with a CIDR block (e.g., 10.0.0.0/16) |
| **Subnet** | A subdivision of the VPC CIDR range, tied to one Availability Zone |
| **Internet Gateway (IGW)** | Allows resources in public subnets to communicate with the internet |
| **NAT Gateway** | Allows private subnet instances to initiate outbound internet traffic |
| **Route Table** | Rules that determine where network traffic is directed |
| **NACL** | Network Access Control List — stateless firewall at the subnet level |
| **Security Group** | Stateful firewall at the instance level |

### Public vs. Private Subnet

```
Public Subnet:
  ✅ Has a route to Internet Gateway (0.0.0.0/0 → IGW)
  ✅ Instances CAN have public IP addresses
  ✅ Reachable from internet (controlled by security groups)
  Use case: Web servers, load balancers, bastion hosts

Private Subnet:
  ❌ NO direct route to Internet Gateway
  ❌ Instances do NOT have public IPs
  ✅ Can reach internet via NAT Gateway (outbound only)
  Use case: Databases, application servers, internal services
```

### CIDR Notation Crash Course

```
10.0.0.0/16  →  65,536 IP addresses (10.0.0.0 to 10.0.255.255)
10.0.1.0/24  →     256 IP addresses (10.0.1.0 to 10.0.1.255)
10.0.2.0/24  →     256 IP addresses (10.0.2.0 to 10.0.2.255)

AWS reserves 5 IPs in each subnet:
  10.0.1.0   → Network address
  10.0.1.1   → AWS VPC router
  10.0.1.2   → AWS DNS
  10.0.1.3   → Future use
  10.0.1.255 → Broadcast address
So /24 gives you 256 - 5 = 251 usable IPs
```

---

## 🛠️ Step-by-Step Guide

### Prerequisites
- AWS account with IAM user (admin or VPC-creation permissions)
- Region: **us-east-1** (N. Virginia) — recommended for this practical

---
<img width="853" height="129" alt="image" src="https://github.com/user-attachments/assets/3afa0ff1-bfd0-4812-9bf0-18b982f42e26" />

### Step A — Create a Custom VPC

1. Go to **AWS Console → VPC → Your VPCs → Create VPC**
2. Select **VPC only**
3. Configure:
   ```
   Name tag:           my-custom-vpc
   IPv4 CIDR block:    10.0.0.0/16
   IPv6 CIDR block:    No IPv6 CIDR block
   Tenancy:            Default
   ```
4. Click **Create VPC**

> ✅ You should see your VPC listed with ID `vpc-XXXXXXXX`

**Enable DNS Hostnames (important):**
```
Select your VPC → Actions → Edit VPC settings →
✅ Enable DNS hostnames → Save
```

---

### Create a Public Subnet

1. Go to **VPC → Subnets → Create subnet**
2. Configure:
   ```
   VPC ID:                my-custom-vpc
   Subnet name:           public-subnet-1
   Availability Zone:     us-east-1a
   IPv4 CIDR block:       10.0.1.0/24
   ```
3. Click **Create subnet**
<img width="1609" height="293" alt="image" src="https://github.com/user-attachments/assets/fed56a08-1510-438a-b5fe-1a1e7f5015e7" />

**Enable Auto-assign Public IP for public subnet:**
```
Select public-subnet-1 → Actions → Edit subnet settings →
✅ Enable auto-assign public IPv4 address → Save
```
<img width="1181" height="128" alt="image" src="https://github.com/user-attachments/assets/09629bf2-259e-48b6-b7e8-293e66dc4379" />

---

### Create a Private Subnet

1. Go to **VPC → Subnets → Create subnet**
2. Configure:
   ```
   VPC ID:                my-custom-vpc
   Subnet name:           private-subnet-1
   Availability Zone:     us-east-1a
   IPv4 CIDR block:       10.0.2.0/24
   ```
3. Click **Create subnet**

> Do NOT enable auto-assign public IP for the private subnet
<img width="1556" height="40" alt="image" src="https://github.com/user-attachments/assets/f82bba1b-6920-40d5-89ff-23ab5d4fb69a" />

---

### Step B — Launch EC2 in Public and Private Subnets

**Public EC2 Instance:**
```
Instance name:   public-ec2
AMI:             Amazon Linux 2
Instance type:   t2.micro
VPC:             my-custom-vpc
Subnet:          public-subnet-1
Auto-assign IP:  Enable
Security Group:  Allow SSH (port 22) from My IP, HTTP (80) from anywhere
Key pair:        my-ec2-keypair
```
<img width="670" height="671" alt="image" src="https://github.com/user-attachments/assets/6c419868-6ec0-4f18-a9d7-6bb96abefbc7" />

**Private EC2 Instance:**
```
Instance name:   private-ec2
AMI:             Amazon Linux 2
Instance type:   t2.micro
VPC:             my-custom-vpc
Subnet:          private-subnet-1
Auto-assign IP:  Disable (it won't have a public IP)
Security Group:  Allow SSH (port 22) from 10.0.1.0/24 (only from public subnet)
Key pair:        my-ec2-keypair
```
<img width="1489" height="79" alt="image" src="https://github.com/user-attachments/assets/c3df97be-96c2-48a5-8f2c-378550eaceaa" />

---

### Step C — Configure Internet Gateway

#### Create the Internet Gateway
1. Go to **VPC → Internet Gateways → Create internet gateway**
2. Configure:
   ```
   Name tag: my-igw
   ```
3. Click **Create internet gateway**

#### Attach IGW to VPC
```
Select my-igw → Actions → Attach to VPC → Select my-custom-vpc → Attach
```
<img width="1833" height="400" alt="image" src="https://github.com/user-attachments/assets/83572144-8440-4121-ba46-ccf027a28b3e" />

#### Create a Public Route Table
1. Go to **VPC → Route Tables → Create route table**
2. Configure:
   ```
   Name:  public-rt
   VPC:   my-custom-vpc
   ```
3. Click **Create route table**

#### Add Route to Internet Gateway
```
Select public-rt → Routes tab → Edit routes → Add route:
  Destination: 0.0.0.0/0
  Target:      Internet Gateway → my-igw
→ Save changes
```
<img width="1598" height="549" alt="image" src="https://github.com/user-attachments/assets/c0496725-14e5-46ae-9fe6-b1e62999e5bb" />

#### Associate Public Subnet with Public Route Table
```
Select public-rt → Subnet associations tab → Edit subnet associations →
✅ public-subnet-1 → Save associations
```
<img width="1907" height="563" alt="image" src="https://github.com/user-attachments/assets/6288d483-24cc-4cef-aaa8-53cf387a2875" />

> Now any instance in `public-subnet-1` can reach the internet!

---

### Step D — Configure NAT Gateway

A **NAT Gateway** (Network Address Translation) sits in the public subnet and allows private instances to initiate outbound internet connections (e.g., to download updates), but blocks inbound connections from the internet.

#### Allocate an Elastic IP
```
VPC → Elastic IPs → Allocate Elastic IP address → Allocate
Note the Allocation ID: eipalloc-XXXXXXXXXX
```
<img width="1290" height="143" alt="image" src="https://github.com/user-attachments/assets/641e2330-3822-4ef8-8b04-a0c0709b11d3" />

#### Create NAT Gateway
1. Go to **VPC → NAT Gateways → Create NAT gateway**
2. Configure:
   ```
   Name:               my-nat-gw
   Subnet:             public-subnet-1  ← MUST be in public subnet!
   Connectivity type:  Public
   Elastic IP:         Select the one you just allocated
   ```
3. Click **Create NAT gateway**
4. Wait for status to change from `Pending` to `Available` (~1-2 minutes)
<img width="1607" height="672" alt="image" src="https://github.com/user-attachments/assets/58ad361d-c2f8-476d-8658-d6431a708d08" />


#### Create a Private Route Table
1. Go to **VPC → Route Tables → Create route table**
2. Configure:
   ```
   Name:  private-rt
   VPC:   my-custom-vpc
   ```

#### Add Route to NAT Gateway
```
Select private-rt → Routes → Edit routes → Add route:
  Destination: 0.0.0.0/0
  Target:      NAT Gateway → my-nat-gw
→ Save changes
```
<img width="1611" height="318" alt="image" src="https://github.com/user-attachments/assets/f0831cc6-435a-43a7-a770-042b27e4f33b" />

#### Associate Private Subnet with Private Route Table
```
Select private-rt → Subnet associations → Edit subnet associations →
✅ private-subnet-1 → Save associations
```
<img width="1599" height="558" alt="image" src="https://github.com/user-attachments/assets/242cab3e-2385-4f15-bd3b-cf3fcf4d0b6c" />

---

## 🔍 Verification

### Test Public EC2 Internet Access
```bash
# SSH into public EC2
ssh -i my-ec2-keypair.pem ec2-user@<public-ip>

# Test internet access
ping google.com
curl https://google.com
```
<img width="1038" height="828" alt="image" src="https://github.com/user-attachments/assets/0d08f8cd-3c39-4878-87b4-e3ea6c67321a" />

### Test Private EC2 Via Bastion (Jump Host)
The public EC2 acts as a **bastion host** (jump server) to reach the private EC2.

**Method 1 — SSH Agent Forwarding:**
```bash
# On your local machine
ssh-add my-ec2-keypair.pem          # Add key to SSH agent

# SSH to public EC2 with agent forwarding
ssh -A -i my-ec2-keypair.pem ec2-user@<public-ec2-ip>

# From inside the public EC2, SSH to private EC2
ssh ec2-user@<private-ec2-private-ip>
```
<img width="916" height="768" alt="image" src="https://github.com/user-attachments/assets/53c4ec55-f6c9-4a40-adda-8f3a66f1e127" />



### Test Private EC2 Can Reach Internet (via NAT)
```bash
# From inside the private EC2
ping google.com           # Should work (via NAT)
curl ifconfig.me          # Will show the NAT Gateway's Elastic IP, not your instance's private IP
```
<img width="809" height="343" alt="image" src="https://github.com/user-attachments/assets/cb2f73fd-38af-4402-84d9-eb66891508a3" />
>

---

## 📊 Route Table Summary

```
Public Route Table (public-rt):
┌────────────────┬──────────────────────────┐
│  Destination   │          Target          │
├────────────────┼──────────────────────────┤
│  10.0.0.0/16   │  local (VPC internal)    │
│  0.0.0.0/0     │  Internet Gateway (IGW)  │
└────────────────┴──────────────────────────┘

Private Route Table (private-rt):
┌────────────────┬──────────────────────────┐
│  Destination   │          Target          │
├────────────────┼──────────────────────────┤
│  10.0.0.0/16   │  local (VPC internal)    │
│  0.0.0.0/0     │  NAT Gateway             │
└────────────────┴──────────────────────────┘
```

---

## 💰 Cost Awareness

```
NAT Gateway Pricing:
  ~$0.045/hour  (≈$32.40/month)
  + $0.045 per GB data processed

⚠️ DELETE the NAT Gateway when not in use — it charges by the hour
   even when no traffic flows through it!

Internet Gateway: FREE (only pay for data transfer)
Elastic IP: FREE when attached to a running instance
            $0.005/hr when not attached or instance is stopped
```

---

## 🔐 Security Best Practices

1. **Principle of Defense in Depth:**
   ```
   Internet → IGW → Security Group → NACL → Instance
   ```
2. **Security Group vs NACL:**
   ```
   Security Group:
     ✅ Stateful (return traffic automatically allowed)
     ✅ Applied at instance level
     ✅ Only ALLOW rules
   
   NACL (Network ACL):
     ✅ Stateless (must explicitly allow return traffic)
     ✅ Applied at subnet level
     ✅ Both ALLOW and DENY rules
     ✅ Rules processed in number order
   ```
3. **Private instances should never have public IPs**
4. **Use bastion host or SSM Session Manager to access private instances**
5. **Enable VPC Flow Logs** for network traffic monitoring

---

## ✅ Learning Outcomes

After completing this practical, you should be able to:

- [ ] Explain what a VPC is and why it is needed
- [ ] Create a VPC with custom CIDR ranges
- [ ] Design public and private subnet architecture
- [ ] Understand and configure Internet Gateways
- [ ] Understand and configure NAT Gateways
- [ ] Create and configure route tables
- [ ] Use a bastion host to access private instances
- [ ] Explain the difference between Security Groups and NACLs

---

## 📚 Further Reading

- [AWS VPC User Guide](https://docs.aws.amazon.com/vpc/latest/userguide/)
- [VPC Pricing](https://aws.amazon.com/vpc/pricing/)
- [NAT Gateway vs. NAT Instance](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-comparison.html)
- [VPC Peering](https://docs.aws.amazon.com/vpc/latest/peering/what-is-vpc-peering.htm
