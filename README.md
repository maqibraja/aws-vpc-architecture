# Deploy Scalable VPC Architecture on AWS Cloud

![AWS-Cloud](https://imgur.com/AXD50yl.png)

### TABLE OF CONTENTS

1. [Goal](#goal)
2. [Project Overview](#project-overview)
3. [Pre-Requisites](#pre-requisites)
4. [Pre-Deployment Setup](#pre-deployment-setup)
5. [VPC Deployment](#vpc-deployment)
6. [Validation](#validation)

---

## Goal

Deploy a Modular and Scalable Virtual Network Architecture with Amazon VPC to serve the highly-available "DevOpsRealtime" static web application.

## Project Overview

This repository is split into two primary components that work together to establish the infrastructure and application:

1. **`html-web-app/`**: Contains the complete source code for the "DevOpsRealtime" corporate website (HTML, CSS, JavaScript, and Images). This is the payload that will be served to the public. 
2. **`VPC Architecture/`**: Contains the critical configuration and automation scripts needed to scaffold out the EC2 instances and AWS services:
   - **`script.sh`**: A Bash script designed to be passed as **User Data** when an EC2 instance launches. It detects the Operating System, installs the Apache web server (`httpd`/`apache2`), installs the AWS CLI and CloudWatch Agent, and downloads the web application directly from your S3 bucket.
   - **`s3-policy.json`**: An IAM policy required to grant your private instances secure, read-only access to download the web app from S3.
   - **`flow-logs.json` & `flow-logs-trusted.json`**: IAM trust and permission policies required to accurately log network traffic (VPC Flow Logs) into CloudWatch.
   - **`memory_metrics.json`**: An IAM policy that allows instances to push custom memory metrics up to CloudWatch for closer server monitoring.

## Pre-Requisites

1. You must be having an [AWS account](https://aws.amazon.com/) to create infrastructure resources on AWS cloud.
2. Create an Amazon S3 Bucket (e.g., `ed-web-config-project`) in your AWS environment.

## Pre-Deployment Setup

Before creating the VPC network, prepare the application payload and the required IAM Roles:

1. **Upload Application Data:** Upload the entire contents of the `html-web-app/` directory into your previously created Amazon S3 Bucket.
2. **Configure S3 IAM Role:** Use the `s3-policy.json` file (ensuring you update `"arn:aws:s3:::your-bucket-name"` to match your created bucket) to create an IAM role for your application instances.
3. **Configure Logging IAM Roles:** Use `flow-logs-trusted.json` and `flow-logs.json` to create the role required for VPC Flow Logs. 
4. **Prepare User Data Execution:** Open `VPC Architecture/script.sh`. Notice the AWS CLI command: `aws s3 cp s3://ed-web-config-project/index.html /var/www/html/`. If your bucket name is different, or if you need to sync the entire folder rather than just `index.html`, update this command before deployment (e.g., `aws s3 sync s3://your-bucket-name/ /var/www/html/`).

## VPC Deployment

Once your IAM roles and User Data scripts are ready, proceed to build the network framework:

1. Build a primary VPC network (e.g., `172.32.0.0/16`) for deploying the Highly Available and Auto Scalable application servers as per the architecture shown above.
2. Build a secondary VPC network (e.g., `192.168.0.0/16`) dedicated strictly for the Bastion Host deployment.
3. Create a **NAT Gateway** in the Public Subnet of your application VPC. Update the Private Subnet Route Tables accordingly to route the default traffic (`0.0.0.0/0`) to the NAT Gateway for outbound internet connections (needed for `script.sh` to download packages).
4. Create an **AWS Transit Gateway** and associate both VPCs to it. Configure route propagation for secure, private communication between the Bastion VPC and the Application VPC.
5. Create an **Internet Gateway (IGW)** for each VPC. Associate Route Tables with the Public Subnets, routing default traffic to the IGW for inbound/outbound internet connections.
6. Create a CloudWatch Log Group with two Log Streams to store the VPC Flow Logs of both VPCs.
7. Enable Flow Logs for both VPCs, pushing the network logs to the CloudWatch Log Groups you just created using the IAM Role configured in Pre-Deployment.
8. Create a Security Group for the Bastion Host allowing `Port 22` from your public IP address.
9. Deploy the Bastion Host EC2 instance in the Bastion VPC's Public Subnet, and associate an Elastic IP (EIP) with it.
10. Create an **EC2 Launch Template** with the following configuration:
    1. Base AMI (e.g., Amazon Linux 2 or Ubuntu)
    2. Instance Type – `t3.micro`
    3. **User Data**: Paste the contents of customized `VPC Architecture/script.sh` to automate the Apache setup and pull your specific web app from S3.
    4. **IAM Role**: Attach the role granting SSM Session Manager access and your S3 bucket read access (`s3-policy.json`).
    5. **Security Group**: Allow Port 22 *only* from the Bastion Host Security Group, and Port 80 *only* from the Application Load Balancer Security Group.
    6. Select a secure Key Pair.
11. Create an **Auto Scaling Group (ASG)** (Min: 2 Max: 4) deploying into the two Private Subnets across multiple Availability Zones for high availability. Select the Launch Template you created.
12. Create a Target Group (Port 80) and associate it with the ASG.
13. Create an **Application Load Balancer (ALB)** spanning the Public Subnets for effective Layer 7 routing, and add the Target Group as the destination.
14. Update your Route 53 hosted zone with a CNAME or Alias record, routing the public DNS traffic to the ALB.

## Validation

1. As a DevOps Engineer, login to your Private Instances via SSH Agent Forwarding through your Bastion Host to ensure the backend networks are successfully isolated.
2. Login via AWS Session Manager to access the EC2 shell directly from the console without keys.
3. Navigate to CloudWatch to ensure your memory metrics and VPC flow logs are successfully transmitting.
4. **Final Check:** Open a public internet browser and browse to the ALB domain name (or Route 53 CNAME) to verify that the *DevOpsRealtime* web page loads flawlessly representing your successful setup!
