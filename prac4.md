# Practical 04 — Configure Auto Scaling and Load Balancing

---

## 📌 Objective

Set up an Auto Scaling Group (ASG) with a launch template, configure scaling policies based on CPU utilization, deploy an Application Load Balancer (ALB) to distribute traffic, and test the setup by simulating high traffic.

---

## 🧠 Conceptual Background (Know-How)

### Why Auto Scaling?

Without auto scaling, you must manually add servers when traffic spikes and remove them when traffic drops. This leads to either:
- **Over-provisioning** — wasting money running too many idle servers
- **Under-provisioning** — poor performance or downtime during traffic spikes

**Auto Scaling** automatically adjusts the number of EC2 instances in response to demand, ensuring your application has the right capacity at the right time.

### Auto Scaling Architecture

```
                           User Requests
                                │
                    ┌───────────▼───────────┐
                    │  Application Load     │
                    │  Balancer (ALB)       │
                    │  [Distributes traffic]│
                    └───────┬───────────────┘
                            │
               ┌────────────┼────────────────┐
               │            │                │
        ┌──────▼──┐  ┌──────▼──┐    ┌────────▼──┐
        │  EC2-1  │  │  EC2-2  │    │  EC2-3   │
        │ (AZ-a)  │  │ (AZ-b)  │    │ (AZ-c)   │
        └─────────┘  └─────────┘    └───────────┘
               └────────────┬────────────────┘
                            │
                ┌───────────▼────────────┐
                │  Auto Scaling Group    │
                │  Min: 2  Desired: 2    │
                │  Max: 5                │
                │                       │
                │  Scale Up: CPU > 70%  │◄── CloudWatch Alarm
                │  Scale Down: CPU<30%  │◄── CloudWatch Alarm
                └────────────────────────┘
```

### Key Components

| Component | Role |
|---|---|
| **Launch Template** | Blueprint for new instances (AMI, type, SG, user data) |
| **Auto Scaling Group (ASG)** | Manages the fleet of EC2 instances |
| **Scaling Policy** | Rules that trigger scale-out or scale-in |
| **Application Load Balancer (ALB)** | Distributes HTTP/HTTPS traffic across instances |
| **Target Group** | Collection of instances the ALB routes to |
| **CloudWatch Alarm** | Monitors metrics and triggers scaling actions |
| **Health Check** | Determines if an instance is healthy enough for traffic |

### Scaling Policy Types

```
1. Target Tracking Scaling (recommended)
   → "Keep CPU at 50%"
   → AWS automatically calculates when to scale

2. Step Scaling
   → "If CPU > 70%, add 2 instances"
   → "If CPU > 90%, add 4 instances"

3. Simple Scaling
   → "If CPU > 70%, add 1 instance, then wait 300s"

4. Scheduled Scaling
   → "At 8 AM weekdays, set desired to 10"
   → "At midnight, set desired to 2"
```

### Load Balancer Types in AWS

```
┌─────────────────────────────────────────────────────┐
│            AWS Elastic Load Balancers                │
├──────────────────┬──────────────────┬───────────────┤
│  Application LB  │   Network LB     │  Gateway LB   │
│  (ALB) L7        │   (NLB) L4       │  L3           │
├──────────────────┼──────────────────┼───────────────┤
│ HTTP/HTTPS       │ TCP/UDP/TLS      │ IP packets    │
│ Path/host-based  │ Ultra-low latency│ Third-party   │
│ routing          │ Millions of req/s│ virtual        │
│ WebSockets       │ Static IP        │ appliances    │
│ gRPC             │                  │               │
└──────────────────┴──────────────────┴───────────────┘
↑ Use ALB for web applications (this practical)
```

---

## 🛠️ Step-by-Step Guide

### Prerequisites
- VPC with at least 2 public subnets in different AZs (use setup from Practical 03 or the default VPC)
- EC2 key pair

---

### Step A — Create a Launch Template

A **Launch Template** defines the configuration for every new instance the ASG launches.

1. Go to **EC2 → Launch Templates → Create launch template**
2. Configure:

```
Launch template name:     web-server-template
Template version:         1 (auto)
Description:              Web server for auto scaling

AMI:                      Amazon Linux 2023 AMI (free tier)
Instance type:            t2.micro
Key pair:                 my-ec2-keypair
Security Group:           Create new OR select existing
  - Inbound: HTTP (80) from 0.0.0.0/0
  - Inbound: SSH (22) from My IP
  - Outbound: All traffic
```

**User Data (bootstrap script) — paste under "Advanced details → User data":**
```bash
#!/bin/bash
# Update packages
yum update -y

# Install Apache web server
yum install -y httpd

# Create a simple webpage that shows instance ID
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)

cat <<EOF > /var/www/html/index.html
<!DOCTYPE html>
<html>
<head><title>Auto Scaling Demo</title></head>
<body style="font-family: Arial; text-align: center; padding: 50px;">
    <h1>🚀 Auto Scaling is Working!</h1>
    <p><strong>Instance ID:</strong> ${INSTANCE_ID}</p>
    <p><strong>Availability Zone:</strong> ${AZ}</p>
    <p>Refresh to see load balancing distribute to different instances</p>
</body>
</html>
EOF

# Start and enable Apache
systemctl start httpd
systemctl enable httpd
```

3. Click **Create launch template**

---
<img width="1919" height="464" alt="image" src="https://github.com/user-attachments/assets/6c5cfafb-8f02-40da-9d51-963bef1bcdf1" />

### Step B — Create an Auto Scaling Group

1. Go to **EC2 → Auto Scaling Groups → Create Auto Scaling group**

**Step 1 — Name and launch template:**
```
Auto Scaling group name: web-asg
Launch template:         web-server-template (Version: Latest)
```

**Step 2 — Instance launch options:**
```
VPC:      Select your VPC (or default VPC)
Subnets:  Select at least 2 public subnets in different AZs
          (e.g., us-east-1a, us-east-1b)
```
<img width="979" height="447" alt="image" src="https://github.com/user-attachments/assets/6f2be390-f5d5-4fd8-b1ed-414c8bd7491e" />

**Step 3 — Configure advanced options (Load Balancing):**
```
Load balancing: Attach to a new load balancer
  Load balancer type:    Application Load Balancer
  Load balancer name:    web-alb
  Load balancer scheme:  Internet-facing
  
  Listeners and routing:
    Protocol: HTTP  Port: 80
    Default routing: Create a target group
      Target group name: web-target-group

Health checks:
  ✅ Turn on Elastic Load Balancing health checks
  Health check grace period: 300 seconds
```
<img width="1624" height="663" alt="image" src="https://github.com/user-attachments/assets/0e1310de-33a4-439b-8d10-32ff397e3123" />

**Step 4 — Configure group size and scaling:**
```
Group size:
  Desired capacity:  2
  Minimum capacity:  2
  Maximum capacity:  5

Scaling policies:
  Select: Target tracking scaling policy
  Policy name:        scale-on-cpu
  Metric type:        Average CPU utilization
  Target value:       70
  Instance warmup:    300 seconds
<img width="1491" height="605" alt="image" src="https://github.com/user-attachments/assets/f001b2ee-bf40-4c71-b877-a12bc3a93b89" />

```
<img width="847" height="578" alt="image" src="https://github.com/user-attachments/assets/afffd062-dac1-4f12-a181-1566e89b0100" />

d123

**Step 6 — Review and Create**

> ⏳ Wait 3–5 minutes for the instances to launch and become healthy

---

### Step C — Deploy Application Load Balancer (Review)

The ALB was created as part of the ASG setup above. Let's verify and understand it.

**Navigate to: EC2 → Load Balancers → web-alb**


```
DNS Name: web-alb-XXXXXXXX.us-east-1.elb.amazonaws.com
State: Active
Availability Zones: us-east-1a, us-east-1b

Listeners:
  HTTP:80 → Forward to web-target-group

Target Group (web-target-group):
  Targets: (your 2+ EC2 instances)
  Health Status: healthy ✅
```

**ALB Routing Concepts:**
```
ALB can route based on:
  Path:  /api/* → backend-tg
         /      → frontend-tg

  Host:  api.example.com → api-tg
         www.example.com → web-tg

  Headers, query strings, source IP
```

**Access your application:**
```
Open browser: http://web-alb-XXXXXXXX.us-east-1.elb.amazonaws.com
```
<img width="1677" height="502" alt="image" src="https://github.com/user-attachments/assets/01df3784-7590-4dfa-910b-f89d767c627b" />

You should see your custom webpage with the instance ID. Refresh several times to see it switch between instances!

---

### Step D — Test Auto Scaling by Simulating High Traffic

#### Method 1 — CPU Stress Test (from inside the instance)
```bash
# SSH into one of your instances
ssh -i my-ec2-keypair.pem ec2-user@<instance-public-ip>

# Install stress tool
sudo yum install -y stress

# Stress the CPU for 5 minutes (300 seconds)
stress --cpu 4 --timeout 300

# In another terminal, watch CPU usage
top
# or
watch -n 1 'uptime'
```
<img width="960" height="232" alt="image" src="https://github.com/user-attachments/assets/82ca3850-78da-4d86-b774-02e118505699" />



#### Monitoring the Scaling Event
```
EC2 → Auto Scaling Groups → web-asg → Activity tab
→ Watch for "Launching a new EC2 instance" events

EC2 → Auto Scaling Groups → web-asg → Monitoring tab
→ Watch CPU utilization graph

CloudWatch → Alarms
→ Watch "scale-on-cpu" alarm change from OK → ALARM → OK
```

---

## 📊 Auto Scaling Lifecycle

```
┌──────────────────────────────────────────────────────────┐
│              EC2 Auto Scaling Lifecycle                  │
│                                                          │
│  Scale-Out (Add Instances):                              │
│                                                          │
│  CloudWatch Alarm → ASG → Launch Template →              │
│  Pending → Pending:Wait → Pending:Proceed →              │
│  InService ← (register with target group)                │
│                                                          │
│  Scale-In (Remove Instances):                            │
│                                                          │
│  CloudWatch Alarm → ASG → Termination Policy →           │
│  InService → Terminating → Terminating:Wait →            │
│  Terminating:Proceed → Terminated                        │
│                                                          │
│  Default Termination Policy:                             │
│  1. Terminate in AZ with most instances                  │
│  2. Oldest launch configuration first                    │
│  3. Closest to next billing hour                         │
└──────────────────────────────────────────────────────────┘
```

---

## 💰 Cost Awareness

```
ALB Pricing:
  $0.008/hour (~$5.76/month)
  + $0.008 per LCU (Load Balancer Capacity Unit)

Auto Scaling itself: FREE
You pay for:
  - The EC2 instances it creates
  - The ALB
  - Data transfer

Cost saving tip:
  Set min=0 for non-production environments when not testing
  → But note: ALB will still charge even with 0 instances
```

---

## ✅ Learning Outcomes

After completing this practical, you should be able to:

- [ ] Create an EC2 Launch Template with user data scripts
- [ ] Configure an Auto Scaling Group with min/desired/max capacity
- [ ] Set up Target Tracking scaling policies
- [ ] Deploy an Application Load Balancer
- [ ] Understand target groups and health checks
- [ ] Simulate high traffic and observe auto scaling behavior
- [ ] Explain the ASG instance lifecycle

---

## 🔐 Security Best Practices

1. The **ALB Security Group** should accept HTTP/HTTPS from `0.0.0.0/0`
2. The **EC2 Security Group** should ONLY accept traffic from the ALB's Security Group (not directly from the internet)
3. Enable **ALB Access Logs** to S3 for audit and debugging
4. Use **HTTPS (port 443)** with ACM (AWS Certificate Manager) certificates on the ALB in production

---

## 📚 Further Reading

- [AWS Auto Scaling Documentation](https://docs.aws.amazon.com/autoscaling/ec2/userguide/)
- [Application Load Balancer Guide](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/)
- [Launch Templates](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-launch-templates.html)
- [Scaling Policy Types](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scale-based-on-demand.html)
