# AWS VPC Architecture - Scalable Cloud Infrastructure

Deploy a **modular, highly available, and auto-scalable VPC architecture** on AWS with security best practices and monitoring integration.

---

## 🏗️ Architecture Overview

| VPC | CIDR Block | Purpose |
|-----|------------|---------|
| **Bastion VPC** | `192.168.0.0/16` | Secure administrative access |
| **Application VPC** | `172.32.0.0/16` | Hosts auto-scalable web servers |

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

- **Bastion Host**: Only SSH from trusted IPs
- **Private Instances**: No direct public access (only via NLB)
- **IAM Roles**: Least privilege (no S3 full access)
- **VPC Flow Logs**: All traffic logged to CloudWatch
- **Session Manager**: Passwordless, auditable EC2 access

---

## 💰 Cost Tips

| Component | Tip |
|-----------|-----|
| EC2 | Use t2.micro for dev; Reserved Instances for production |
| NAT Gateway | Use NAT Instance for low-traffic workloads |
| Auto Scaling | Right-size min/max based on demand |

---

## 🐛 Troubleshooting

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
