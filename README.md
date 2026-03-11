# Deploy Scalable VPC Architecture on AWS Cloud

![AWS-Cloud](https://imgur.com/AXD50yl.png)

### TABLE OF CONTENTS

1. [Goal](#goal)
2. [Pre-Requisites](#pre-requisites)
3. [Pre-Deployment](#pre-deployment)
4. [VPC Deployment](#vpc-deployment)
5. [Validation](#validation)

---

## Goal

Deploy a Modular and Scalable Virtual Network Architecture with Amazon VPC. 

This project demonstrates how to build a highly available, fault-tolerant, and secure infrastructure on AWS. By leveraging multiple Availability Zones, public/private subnets, and isolated VPCS with a Transit Gateway, you ensure that application servers remain secure from direct internet exposure while still being capable of serving web traffic efficiently.

## Pre-Requisites

Before you begin, ensure you have the following:

1. An active **[AWS account](https://aws.amazon.com/)** with administrator privileges to provision VPCs, EC2 instances, S3 buckets, and IAM roles.
2. Basic knowledge of networking concepts (IP addressing, CIDR blocks, routing).
3. The **Source Code** provided in the `html-web-app` directory of this repository, which contains the web application we will host.
4. **AWS CLI** installed locally (optional but recommended for interacting with the environment).

---

## Pre-Deployment

Before building the network, we must prepare the logical components and permissions that our instances will use. This repository contains a `VPC Architecture/` folder with several JSON configurations and a bootstrapping script (`script.sh`) to automate server setup.

### 1. Understanding the Bootstrapping Script (`script.sh`)
Instead of manually installing software on every server, we use a **User Data script** (`script.sh`). When an EC2 instance launches, this script automatically:
- Detects the operating system (Amazon Linux 2 or Ubuntu).
- Installs the Apache Web Server (`httpd` or `apache2`).
- Installs the AWS CLI and the Amazon CloudWatch Agent.
- Fetches the application web files directly from an Amazon S3 bucket (e.g., `s3://ed-web-config-project/index.html`) to the web root directory.
- Configures the CloudWatch Agent to monitor system logs.

### 2. IAM Policies & Roles
For security, EC2 instances run with IAM Roles instead of hardcoded API keys.
- **`s3-policy.json`**: An IAM policy that grants the EC2 instance permission to securely read configurations and web files from our specified S3 bucket.
- **`flow-logs.json` & `flow-logs-trusted.json`**: IAM policies and trust relationships required to authorize VPC Flow Logs to write network traffic logs to Amazon CloudWatch.
- **`memory_metrics.json`**: CloudWatch configuration file used to push custom memory metrics, enabling deeper insight into application performance.

---

## VPC Deployment

This phase involves creating the physical and logical networking boundaries. We separate the architecture into two distinct VPCs for enhanced security: one for the Bastion Host (jump box) and one for the Application servers.

### Step 1: Base Network Setup
1. **Bastion VPC**: Build a VPC network (e.g., `192.168.0.0/16`) for the Bastion Host deployment. This acts as an initial secure entry point.
2. **Application VPC**: Build a VPC network (e.g., `172.32.0.0/16`) for deploying the Highly Available and Auto Scalable application servers.
3. **Internet Gateways (IGW)**: Create an Internet Gateway for each VPC. Update the Route Tables associated with the Public Subnets to route default outward traffic (`0.0.0.0/0`) to the IGW, granting outbound/inbound internet connectivity for public resources.

### Step 2: Private Network & Inter-VPC Routing
4. **NAT Gateway**: Create a NAT Gateway in the Public Subnet of the Application VPC. Update the Private Subnet Route Table to route outgoing internet traffic to the NAT Gateway. This allows private backend servers to download software patches without being directly exposed to the internet.
5. **Transit Gateway**: Create an AWS Transit Gateway and attach both VPCs to it. Update routing tables so that the Bastion VPC can communicate securely with the Application VPC using private IP addresses.

### Step 3: Monitoring & Security
6. **CloudWatch Log Groups**: Create a CloudWatch Log Group with two Log Streams to store the VPC Flow Logs of both VPCs.
7. **Enable VPC Flow Logs**: Enable Flow Logs for both VPCs using the IAM roles configured in the pre-deployment phase. This will record all IP traffic going to and from network interfaces in your VPC, which is critical for security audits and troubleshooting.
8. **Security Groups**: 
   - Create a Bastion Host Security Group allowing inbound Port 22 (SSH) strictly from your trusted public IP address.
   - Create an Application Security Group allowing inbound Port 80 (HTTP) from the Load Balancer, and Port 22 only from the Bastion Host Security Group.

### Step 4: Compute & Scaling Infrastructure
9. **Deploy Bastion Host**: Launch an EC2 instance in the Public Subnet of the Bastion VPC and associate an Elastic IP (EIP).
10. **Application Storage**: Create an S3 Bucket and upload your `html-web-app` files. This is where `script.sh` will fetch the files.
11. **Launch Template**: Create an EC2 Launch Template to standardize new instances.
    - **AMI**: Use a base image like Amazon Linux 2.
    - **Instance Type**: `t3.micro`.
    - **User Data**: Paste the contents of `script.sh`.
    - **IAM Profile**: Attach a role with AmazonSSMManagedInstanceCore and your S3 read policy.
    - **Security Group**: Select the Application Security Group.
12. **Auto Scaling Group (ASG)**: Create an ASG (Min: 2, Max: 4) deployed across two Private Subnets in different Availability Zones to guarantee High Availability. 
13. **Target Group & ALB**: 
    - Create a Target Group that groups instances on Port 80.
    - Create an Application Load Balancer (ALB) across the Public Subnets. The ALB receives public traffic and securely distributes it down into your private Application Subnets via the Target Group.
    - Associate the Target Group with the ASG so new instances are automatically registered.
14. **DNS Routing**: Update your Route 53 hosted zone to create a CNAME or Alias record pointing your custom domain name to the ALB’s DNS name.

---

## Validation

After the deployment stabilizes, you need to verify that the networking rules, routing, and applications are functioning successfully:

1. **Test Bastion Host SSH**: As a DevOps Engineer, use SSH agent forwarding to log into the Bastion Host, and from there, attempt an SSH connection into your Private Application Instances.
2. **AWS Systems Manager**: Validate that you can use AWS Session Manager from the AWS Console to open a secure shell to your private EC2 instances without needing SSH keys.
3. **Web Application Reachability**: Open a web browser (preferably Incognito) and navigate to your ALB domain name. Verify that your HTML web application loads successfully and serves the CSS/JS appropriately.
4. **Log Verification**: Visit Amazon CloudWatch in the AWS Console. Ensure that the VPC Flow logs are populated and that your `script.sh` deployment logs exist in the system logs stream.
