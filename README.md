# Deploy Scalable VPC Architecture on AWS Cloud

![AWS-Cloud](https://imgur.com/AXD50yl.png)

### TABLE OF CONTENTS

1. [Goal](#goal)
2. [Pre-Requisites](#pre-requisites)
3. [Directory Structure](#directory-structure)
4. [Pre-Deployment](#pre-deployment)
5. [VPC Deployment](#vpc-deployment)
6. [Validation](#validation)

## Goal

Deploy a Modular and Scalable Virtual Network Architecture with Amazon VPC.

## Pre-Requisites

1. You must be having an [AWS account](https://aws.amazon.com/) to create infrastructure resources on AWS cloud.
2. Source Code is available in the `html-web-app` directory of this repository.

## Directory Structure

- **`VPC Architecture/`**: Contains necessary configuration files and bootstrapping scripts:
  - `script.sh`: User data shell script to automate the configuration of EC2 instances upon launch. It detects the OS, installs dependencies (Apache/HTTPD, AWS CLI, CloudWatch Agent), and fetches web content from an S3 bucket.
  - `flow-logs.json`, `flow-logs-trusted.json`: IAM JSON policies related to setting up and authorizing VPC Flow Logs.
  - `memory_metrics.json`: CloudWatch Agent configuration to track and push custom memory metrics.
  - `s3-policy.json`: IAM policy snippet for allowing EC2 instances secure and restricted access to an S3 bucket.
- **`html-web-app/`**: Contains the source code for the web application (`index.html`, CSS, JS, etc.) that will be hosted on the highly available AWS infrastructure.

## Pre-Deployment

Customize the application dependencies mentioned below on AWS EC2 instance and create the Golden AMI. Alternatively, you can use the provided `script.sh` as User Data to bootstrap instances on launch.

1. AWS CLI
2. Install Apache Web Server
3. Install Git
4. Cloudwatch Agent
5. Push custom memory metrics to Cloudwatch (using `memory_metrics.json`).
6. AWS SSM Agent

## VPC Deployment

1. Build a primary VPC network (e.g., `172.32.0.0/16`) for deploying your Highly Available and Auto Scalable application servers.
2. Build a secondary VPC network (e.g., `192.168.0.0/16`) dedicated to the Bastion Host deployment.
3. Create an Internet Gateway (IGW) and attach it to both VPCs. Update the Public Subnet Route Tables to route default public traffic (`0.0.0.0/0`) to the IGW.
4. Create a NAT Gateway in the Public Subnets and update the Private Subnet Route Tables to route default outbound internet traffic to the NAT Gateway.
5. Create a Transit Gateway and attach both VPCs to it for private communication.
6. Create an IAM role for VPC Flow Logs using the provided `flow-logs-trusted.json` and `flow-logs.json` policies. Create a Cloudwatch Log Group with two Log Streams to store the VPC Flow Logs of both VPCs.
7. Enable Flow Logs for both VPCs and push the Flow Logs to Cloudwatch Log Groups.
8. Create Security Group for bastion host allowing port 22 from your public IP.
9. Deploy Bastion Host EC2 instance in the Public Subnet with EIP associated.
10. Create S3 Bucket to store application specific configuration and upload the `html-web-app` source code.
11. Create Launch Template with below configuration:
    1. Golden AMI (or base AMI with Userdata)
    2. Instance Type – t3.micro
    3. Userdata (from `script.sh`) to pull the code from S3 bucket to document root folder of webserver and start the httpd service.
    4. IAM Role granting access to Session Manager and to S3 bucket (`s3-policy.json`).
    5. Security Group allowing port 22 from Bastion Host and Port 80 from Public.
    6. Key Pair
12. Create Auto Scaling Group with Min: 2 Max: 4 with two Private Subnets associated to multiple AZs for high availability.
13. Create Target Group and associate it with ASG.
14. Create Application Load Balancer (ALB) in Public Subnets for better layer 7 routing and add Target Group as target.
15. Update route53 hosted zone with CNAME/Alias record routing the traffic to ALB.

## Validation

1. As DevOps Engineer login to Private Instances via Bastion Host using ssh agent forwarding.
2. Login to AWS Session Manager and access the EC2 shell from console.
3. Browse web application from public internet browser using domain name and verify that page loaded.
4. Access Amazon CloudWatch Logs to confirm that system logs, custom memory metrics, and VPC Flow Logs are being successfully recorded and monitored properly.
