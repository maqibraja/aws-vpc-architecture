# AWS VPC Architecture - Scalable Cloud Infrastructure

![AWS](https://img.shields.io/badge/AWS-Cloud-orange)
![VPC](https://img.shields.io/badge/Service-VPC-blue)
![EC2](https://img.shields.io/badge/Service-EC2-lightgrey)
![CloudWatch](https://img.shields.io/badge/Monitoring-CloudWatch-orange)
![Auto Scaling](https://img.shields.io/badge/Feature-Auto%20Scaling-green)

Deploy a **modular, highly available, and auto-scalable VPC architecture** on AWS with security best practices and monitoring integration.

![Architecture](images/architecture-diagram.png)

---

## 🏗️ Architecture Overview

| VPC | CIDR Block | Purpose |
|-----|------------|---------|
| **Bastion VPC** | `192.168.0.0/16` | Secure administrative access |
| **Application VPC** | `172.32.0.0/16` | Hosts auto-scalable web servers |

### High-Level Architecture Diagram

![Network Flow](images/network-flow.png)

```
                              ┌─────────────────────────────────────────────────────────┐
                              │                      AWS Cloud                          │
                              │                                                         │
    ┌─────────────────────┐   │   ┌──────────────────┐      ┌────────────────────────┐  │
    │   Public Internet   │───┼───►  Internet Gateway │      │   Bastion VPC          │  │
    │                     │   │   │   (IGW)          │      │   192.168.0.0/16       │  │
    └─────────────────────┘   │   └──────────────────┘      │                        │  │
                              │                              │  ┌──────────────────┐  │  │
                              │                              │  │ Public Subnet    │  │  │
                              │                              │  │                  │  │  │
    ┌─────────────────────┐   │                              │  │  ┌────────────┐  │  │  │
    │   Route53 DNS       │───┼──────────────────────────────┼──┼──│  Bastion   │  │  │  │
    │ app.yourdomain.com  │   │                              │  │  │   Host     │  │  │  │
    └─────────────────────┘   │                              │  │  └────────────┘  │  │  │
                              │                              │  │                  │  │  │
                              │                              │  └──────────────────┘  │  │
                              │                              │                        │  │
                              │   ┌──────────────────────────┴────────────────────────┘  │
                              │   │ Transit Gateway (TGW)                                │
                              │   └──────────────────────────┬───────────────────────────┘
                              │                              │
                              │   ┌──────────────────────────▼───────────────────────────┐
                              │   │         Application VPC                              │
                              │   │         172.32.0.0/16                                │
                              │   │                                                      │
                              │   │  ┌────────────────────┐    ┌──────────────────────┐  │
                              │   │  │  Public Subnet     │    │   Private Subnets    │  │
                              │   │  │                    │    │   (Multi-AZ)         │  │
                              │   │  │ ┌────────────────┐ │    │                      │  │
                              │   │  │ │ Network Load   │ │    │  ┌────────────────┐  │  │
                              │   │  │ │   Balancer     │─┼────┼──│ Auto Scaling   │  │  │
                              │   │  │ │     (NLB)      │ │    │  │    Group       │  │  │
                              │   │  │ └────────────────┘ │    │  │  (EC2 x 2-4)   │  │  │
                              │   │  │ ┌────────────────┐ │    │  └────────────────┘  │  │
                              │   │  │ │   NAT Gateway  │ │    │                      │  │
                              │   │  │ └────────────────┘ │    │  ┌────────────────┐  │  │
                              │   │  │ ┌────────────────┐ │    │  │  Target Group  │  │  │
                              │   │  │ │ Internet GW    │ │    │  │   (Port 80)    │  │  │
                              │   │  │ └────────────────┘ │    │  └────────────────┘  │  │
                              │   │  └────────────────────┘    └──────────────────────┘  │
                              │   │                                                      │
                              │   └──────────────────────────────────────────────────────┘
                              │
                              │   ┌──────────────────────────────────────────────────────┐
                              │   │              CloudWatch                              │
                              │   │  • VPC Flow Logs                                     │
                              │   │  • Custom Memory Metrics                             │
                              │   │  • System Logs                                       │
                              │   └──────────────────────────────────────────────────────┘
                              └─────────────────────────────────────────────────────────┘
```

### Key Components

- ✅ **Multi-AZ Deployment** – High availability across availability zones
- ✅ **Auto Scaling** – Min: 2, Max: 4 EC2 instances
- ✅ **Load Balancing** – Network Load Balancer (NLB)
- ✅ **Bastion Host** – Secure SSH access point
- ✅ **Monitoring** – CloudWatch Logs & Custom Metrics
- ✅ **VPC Peering** – Transit Gateway for private VPC communication

---

## 📁 Project Structure

```
aws-vpc-architecture/
├── README.md
├── html-web-app/              # Sample web application
│   ├── index.html
│   ├── css/, js/, images/
│   └── WEB-INF/
└── VPC Architecture/
    ├── script.sh              # EC2 bootstrap/userdata script
    ├── s3-policy.json         # S3 access IAM policy
    ├── flow-logs.json         # VPC Flow Logs IAM policy
    ├── flow-logs-trusted.json # VPC Flow Logs trust policy
    └── memory_metrics.json    # CloudWatch metrics policy
```

---

## 📋 Prerequisites

- AWS Account with appropriate IAM permissions
- AWS CLI v2 installed and configured
- SSH key pair for EC2 access

---

## 🚀 Deployment Steps

### 1. VPC Network Setup

```bash
# Create Bastion VPC
aws ec2 create-vpc --cidr-block 192.168.0.0/16

# Create Application VPC
aws ec2 create-vpc --cidr-block 172.32.0.0/16
```

**Subnets to create:**
- Public subnets (for Bastion, NLB, NAT Gateway)
- Private subnets (for EC2 instances in Multi-AZ)

### Network Flow Diagram

```
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│   Internet   │ ───► │ Internet GW  │ ───► │  Public      │
│  (Users)     │      │   (IGW)      │      │  Subnet      │
└──────────────┘      └──────────────┘      └──────┬───────┘
                                                   │
                    ┌──────────────────────────────┼──────────────────────────────┐
                    │                              ▼                              │
                    │                    ┌──────────────────┐                     │
                    │                    │  Network Load    │                     │
                    │                    │   Balancer       │                     │
                    │                    │    (NLB)         │                     │
                    │                    └────────┬─────────┘                     │
                    │                             │                               │
                    │                             ▼                               │
                    │                    ┌──────────────────┐                     │
                    │                    │  Target Group    │                     │
                    │                    │   (Port 80)      │                     │
                    │                    └────────┬─────────┘                     │
                    │                             │                               │
                    │              ┌──────────────┴──────────────┐                │
                    │              ▼                             ▼                │
                    │    ┌──────────────────┐        ┌──────────────────┐         │
                    │    │ Private Subnet   │        │ Private Subnet   │         │
                    │    │ AZ-1a            │        │ AZ-1b            │         │
                    │    │ ┌──────────────┐ │        │ ┌──────────────┐ │         │
                    │    │ │  EC2 Instance│ │        │ │  EC2 Instance│ │         │
                    │    │ │  (Apache)    │ │        │ │  (Apache)    │ │         │
                    │    │ └──────────────┘ │        │ └──────────────┘ │         │
                    │    └────────┬─────────┘        └──────────────────┘         │
                    │             │                                               │
                    │             ▼                                               │
                    │    ┌──────────────────┐                                     │
                    │    │   NAT Gateway    │ ◄─── Outbound Internet Access       │
                    │    └──────────────────┘                                     │
                    └─────────────────────────────────────────────────────────────┘

                    ┌─────────────────────────────────────────────────────────────┐
                    │                    Transit Gateway                          │
                    │         (Connects Bastion VPC ↔ Application VPC)            │
                    └─────────────────────────────────────────────────────────────┘
```

### 2. Routing & Connectivity

- **Internet Gateway** – Attach to each VPC for public access
- **NAT Gateway** – Deploy in public subnet for private subnet outbound traffic
- **Transit Gateway** – Connect both VPCs for private communication

### 3. Monitoring Setup

```bash
# Create CloudWatch Log Group
aws logs create-log-group --log-group-name /aws/vpc/flowlogs

# Enable VPC Flow Logs
aws ec2 create-flow-logs --resource-type VPC --resource-ids vpc-xxx
```

Use provided IAM policies:
- `flow-logs.json` – Permissions for CloudWatch Logs
- `memory_metrics.json` – Custom memory metrics

### Monitoring Architecture

![Monitoring](images/cloudwatch-dashboard.png)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CloudWatch Monitoring                        │
│                                                                     │
│   ┌──────────────────┐    ┌──────────────────┐    ┌──────────────┐ │
│   │   VPC Flow Logs  │    │  Memory Metrics  │    │ System Logs  │ │
│   │                  │    │                  │    │              │ │
│   │ /aws/vpc/        │    │ /aws/memory/     │    │ /var/log/    │ │
│   │ flowlogs         │    │ metrics          │    │ messages     │ │
│   │                  │    │                  │    │              │ │
│   │ • Network traffic│    │ • RAM usage      │    │ • OS events  │ │
│   │ • Security audit │    │ • Performance    │    │ • Debugging  │ │
│   │ • Troubleshooting│    │ • Alerting       │    │ • Compliance │ │
│   └────────┬─────────┘    └────────┬─────────┘    └──────┬───────┘ │
│            │                       │                      │        │
│            └───────────────────────┼──────────────────────┘        │
│                                    ▼                                │
│                        ┌──────────────────────┐                     │
│                        │   CloudWatch Dashboard │                   │
│                        │   • Metrics & Graphs   │                   │
│                        │   • Alarms & Alerts    │                   │
│                        │   • Log Insights       │                   │
│                        └──────────────────────┘                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 4. Bastion Host

```bash
# Launch EC2 in Public Subnet
aws ec2 run-instances --image-id ami-xxx --instance-type t2.micro

# Associate Elastic IP
aws ec2 associate-address --instance-id i-xxx
```

**Security Group:** Allow SSH (port 22) from trusted IPs only.

### 5. Application Infrastructure

**Golden AMI should include:**
- Apache Web Server
- AWS CLI v2
- CloudWatch Agent
- AWS SSM Agent

**Create:**
1. **S3 Bucket** – Store application config (`ed-web-config-project`)
2. **IAM Role** – S3 read access + CloudWatch + Session Manager
3. **Launch Configuration** – Golden AMI, t2.micro, userdata script
4. **Auto Scaling Group** – Min: 2, Max: 4, Multi-AZ
5. **Target Group** – Health checks on port 80
6. **Network Load Balancer** – Public subnet, forwards to Target Group

### 6. DNS Configuration

```bash
# Route53 CNAME record pointing to NLB
app.yourdomain.com → App-NLB-xxx.elb.amazonaws.com
```

---

## ✅ Validation

### Access Private Instances via Bastion

```bash
ssh -i key.pem ec2-user@<bastion-ip>
ssh -i key.pem ec2-user@<private-instance-ip>
```

### Session Manager Access

```bash
aws ssm start-session --target i-xxx
```

### Test Web Application

```bash
curl http://app.yourdomain.com
# Should load DevOpsRealtime homepage
```

### Verify Components

```bash
# Check ASG status
aws autoscaling describe-auto-scaling-groups

# Check target health
aws elbv2 describe-target-health

# Verify S3 access from EC2
aws s3 ls s3://ed-web-config-project/
```

---

## 🔒 Security Best Practices

### Security Architecture

![Security](images/security-groups.png)

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Security Layers                                │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Layer 1: Network Security                                   │   │
│  │  • VPC isolation (separate CIDR blocks)                      │   │
│  │  • Public/Private subnet segmentation                        │   │
│  │  • Security Groups (stateful firewalls)                      │   │
│  │  • NACLs (stateless network ACLs)                            │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Layer 2: Access Control                                     │   │
│  │  • Bastion Host (single SSH entry point)                     │   │
│  │  • IAM Roles (no hardcoded credentials)                      │   │
│  │  • Session Manager (passwordless access)                     │   │
│  │  • MFA for privileged users                                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Layer 3: Monitoring & Audit                                 │   │
│  │  • VPC Flow Logs (all network traffic)                       │   │
│  │  • CloudTrail (API activity logging)                         │   │
│  │  • CloudWatch Alarms (anomaly detection)                     │   │
│  │  • Config Rules (compliance checking)                        │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘

Access Flow:
────────────
Internet ──► NLB (Port 80) ──► EC2 (Private)
                │
                └──► Health Checks ──► Target Group

Admin ──► Bastion Host (SSH Port 22) ──► Private EC2
          (Public IP, restricted SG)
```

### Security Group Rules

| Component | Inbound Rules | Outbound Rules |
|-----------|--------------|----------------|
| **Bastion SG** | SSH (22) from trusted IPs | All traffic |
| **App SG** | HTTP (80) from NLB, SSH (22) from Bastion | All traffic |
| **NLB SG** | HTTP (80) from Internet | All traffic |

---

## 💰 Cost Tips

### Cost Optimization Architecture

![Cost](images/cost-optimization.png)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Cost Optimization Strategies                     │
│                                                                     │
│  ┌─────────────────────┐  ┌─────────────────────┐                  │
│  │   Compute (EC2)     │  │   Network           │                  │
│  │                     │  │                     │                  │
│  │  ✓ t2.micro for dev │  │  ✓ NAT Instance     │                  │
│  │  ✓ Reserved Instances│  │    (low traffic)    │                  │
│  │  ✓ Spot Instances   │  │  ✓ Same region      │                  │
│  │    (fault-tolerant) │  │    (no x-region     │                  │
│  │  ✓ Auto Scaling     │  │     data transfer)  │                  │
│  │    (right-size)     │  │                     │                  │
│  └─────────────────────┘  └─────────────────────┘                  │
│                                                                     │
│  ┌─────────────────────┐  ┌─────────────────────┐                  │
│  │   Storage (S3/EBS)  │  │   Monitoring        │                  │
│  │                     │  │                     │                  │
│  │  ✓ S3 Lifecycle     │  │  ✓ Log retention    │                  │
│  │    policies         │  │    policies         │                  │
│  │  ✓ EBS gp2 vs gp3   │  │  ✓ Metric filtering │                  │
│  │  ✓ Delete unused    │  │  ✓ Alarm thresholds │                  │
│  │    snapshots        │  │    (reduce noise)   │                  │
│  └─────────────────────┘  └─────────────────────┘                  │
└─────────────────────────────────────────────────────────────────────┘

Estimated Monthly Cost (us-west-2):
───────────────────────────────────
Bastion (t2.micro)        : ~$7.50
App EC2 x2 (t2.micro)     : ~$15.00
NAT Gateway               : ~$32.40 + data processing
NLB                       : ~$16.20 + LCU charges
Transit Gateway           : ~$36.00 + data processing
CloudWatch Logs           : ~$5.00 (varies by volume)
───────────────────────────────────
Total (approx.)           : ~$112+/month
```

---

## 🐛 Troubleshooting

### Troubleshooting Flowchart

![Troubleshooting](images/troubleshooting.png)

```
                        ┌─────────────────┐
                        │  Issue Reported │
                        └────────┬────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
              ▼                  ▼                  ▼
    ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
    │ Website Down    │ │ Can't SSH       │ │ High Latency    │
    └────────┬────────┘ └────────┬────────┘ └────────┬────────┘
             │                   │                   │
             ▼                   ▼                   ▼
    ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
    │ Check NLB       │ │ Check Bastion   │ │ Check CPU/Mem   │
    │ Target Health   │ │ SG & EIP        │ │ CloudWatch      │
    └────────┬────────┘ └────────┬────────┘ └────────┬────────┘
             │                   │                   │
             ▼                   ▼                   ▼
    ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
    │ Verify Apache   │ │ Check IAM       │ │ Check ASG       │
    │ Service on EC2  │ │ Permissions     │ │ Scale Out       │
    └─────────────────┘ └─────────────────┘ └─────────────────┘
```

### Common Issues & Solutions

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| **503 Service Unavailable** | Target Group unhealthy | Check security groups, verify Apache running |
| **Connection Timeout** | Route table misconfigured | Verify IGW/NAT Gateway routes |
| **SSH Access Denied** | Wrong SG or key | Check Bastion SG allows your IP |
| **S3 Access Denied** | IAM policy missing | Attach S3 read policy to EC2 role |
| **No CloudWatch Logs** | Agent not running | `sudo systemctl status amazon-cloudwatch-agent` |

---

## 📸 Screenshots & Visuals

### Architecture Deployment

![VPC Dashboard](images/vpc-dashboard.png)
![Transit Gateway](images/transit-gateway.png)
![Auto Scaling](images/auto-scaling.png)
![Load Balancer](images/nlb-targets.png)
![CloudWatch Metrics](images/cloudwatch-metrics.png)

---

## 🐛 Quick Troubleshooting Commands

**Instances not launching?**
```bash
aws autoscaling describe-launch-configurations
# Check /var/log/cloud-init-output.log on EC2
```

**Website not accessible?**
```bash
# Check Security Groups
aws ec2 describe-security-groups

# Verify Apache
sudo systemctl status httpd
```

**S3 access denied?**
```bash
# Verify IAM policy
aws iam list-attached-role-policies --role-name EC2-App-Role
```

---

## 📞 Support

- **Issues**: Open on GitHub
- **Contact**: support@DevOpsRealtime.com

---

**Happy Deploying! 🚀**
