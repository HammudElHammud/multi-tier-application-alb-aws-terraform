# Multi-Tier Web Application Deployment in a Custom VPC

This project demonstrates how to deploy a multi-tier web application in a custom VPC on AWS. The setup includes creating a VPC, launching EC2 instances, configuring a load balancer, setting up auto-scaling, and deploying an RDS database.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Setup Instructions](#setup-instructions)
    - [1. Create a Custom VPC](#1-create-a-custom-vpc)
    - [2. Launch EC2 Instances](#2-launch-ec2-instances)
    - [3. Create a Target Group](#3-create-a-target-group)
    - [4. Launch an Application Load Balancer](#4-launch-an-application-load-balancer)
    - [5. Configure Auto Scaling](#5-configure-auto-scaling)
    - [6. Launch a Multi-AZ RDS Database](#6-launch-a-multi-az-rds-database)
    - [7. Test the Setup](#7-test-the-setup)
4. [Cleanup](#cleanup)

## Architecture Overview

The architecture consists of:
- A custom VPC with 2 public and 2 private subnets in different availability zones.
- EC2 instances in private subnets serving as web and application tiers.
- A NAT Gateway in each public subnet for internet access for instances in private subnets.
- An Application Load Balancer in public subnets to distribute traffic to EC2 instances.
- An Auto Scaling group to manage the EC2 instances.
- A Multi-AZ RDS instance for database services.

## Prerequisites

- AWS CLI configured with appropriate permissions.
- AWS account with access to create VPCs, subnets, EC2 instances, NAT Gateways, ALBs, Auto Scaling groups, and RDS instances.

## Setup Instructions

### 1. Create a Custom VPC

1. **Create VPC**:
    ```sh
    aws ec2 create-vpc --cidr-block 10.0.0.0/16
    ```

2. **Create Subnets**:
    ```sh
    aws ec2 create-subnet --vpc-id <VPC_ID> --cidr-block 10.0.10.0/24 --availability-zone us-east-1a
    aws ec2 create-subnet --vpc-id <VPC_ID> --cidr-block 10.0.20.0/24 --availability-zone us-east-1b
    aws ec2 create-subnet --vpc-id <VPC_ID> --cidr-block 10.0.100.0/24 --availability-zone us-east-1a
    aws ec2 create-subnet --vpc-id <VPC_ID> --cidr-block 10.0.200.0/24 --availability-zone us-east-1b
    ```

3. **Create and Attach Internet Gateway**:
    ```sh
    aws ec2 create-internet-gateway
    aws ec2 attach-internet-gateway --vpc-id <VPC_ID> --internet-gateway-id <IGW_ID>
    ```

4. **Create Route Tables and Routes**:
    ```sh
    aws ec2 create-route-table --vpc-id <VPC_ID>
    aws ec2 create-route --route-table-id <RT_ID> --destination-cidr-block 0.0.0.0/0 --gateway-id <IGW_ID>
    ```

5. **Associate Route Tables**:
    ```sh
    aws ec2 associate-route-table --route-table-id <RT_ID> --subnet-id <PUBLIC_SUBNET_ID>
    ```

### 2. Launch EC2 Instances

1. **Create Security Group**:
    ```sh
    aws ec2 create-security-group --group-name webSG --description "Web Security Group" --vpc-id <VPC_ID>
    aws ec2 authorize-security-group-ingress --group-id <SG_ID> --protocol tcp --port 22 --cidr 0.0.0.0/0
    aws ec2 authorize-security-group-ingress --group-id <SG_ID> --protocol tcp --port 80 --cidr 0.0.0.0/0
    aws ec2 authorize-security-group-ingress --group-id <SG_ID> --protocol tcp --port 443 --cidr 0.0.0.0/0
    ```

2. **Launch EC2 Instances**:
    ```sh
    aws ec2 run-instances --image-id <AMI_ID> --count 1 --instance-type t2.micro --key-name <KEY_PAIR_NAME> --security-group-ids <SG_ID> --subnet-id <PRIVATE_SUBNET_ID> --user-data file://userdata.sh
    ```

3. **Encrypt EBS Volumes**:
   Ensure that the instances use encrypted EBS volumes by specifying the encryption parameter when creating the volumes.

4. **Create NAT Gateways**:
    ```sh
    aws ec2 create-nat-gateway --subnet-id <PUBLIC_SUBNET_ID> --allocation-id <EIP_ALLOC_ID>
    ```

5. **Update Route Tables for Private Subnets**:
    ```sh
    aws ec2 create-route --route-table-id <PRIVATE_RT_ID> --destination-cidr-block 0.0.0.0/0 --nat-gateway-id <NAT_GATEWAY_ID>
    ```

### 3. Create a Target Group

1. **Create Target Group**:
    ```sh
    aws elbv2 create-target-group --name webTG --protocol HTTP --port 80 --vpc-id <VPC_ID>
    aws elbv2 register-targets --target-group-arn <TG_ARN> --targets Id=<INSTANCE_ID1> Id=<INSTANCE_ID2>
    ```

### 4. Launch an Application Load Balancer

1. **Create Security Group for ALB**:
    ```sh
    aws ec2 create-security-group --group-name ALBSG --description "ALB Security Group" --vpc-id <VPC_ID>
    aws ec2 authorize-security-group-ingress --group-id <ALB_SG_ID> --protocol tcp --port 80 --cidr 0.0.0.0/0
    ```

2. **Create ALB**:
    ```sh
    aws elbv2 create-load-balancer --name webALB --subnets <SUBNET1_ID> <SUBNET2_ID> --security-groups <ALB_SG_ID>
    ```

3. **Create Listener**:
    ```sh
    aws elbv2 create-listener --load-balancer-arn <ALB_ARN> --protocol HTTP --port 80 --default-actions Type=forward,TargetGroupArn=<TG_ARN>
    ```

4. **Update Security Groups**:
    - Update `webSG` to allow inbound traffic from `ALBSG`.

### 5. Configure Auto Scaling

1. **Create Launch Configuration**:
    ```sh
    aws autoscaling create-launch-configuration --launch-configuration-name webLC --image-id <AMI_ID> --instance-type t2.micro --key-name <KEY_PAIR_NAME> --security-groups <SG_ID> --user-data file://userdata.sh
    ```

2. **Create Auto Scaling Group**:
    ```sh
    aws autoscaling create-auto-scaling-group --auto-scaling-group-name webASG --launch-configuration-name webLC --min-size 1 --max-size 4 --desired-capacity 2 --vpc-zone-identifier <SUBNET1_ID>,<SUBNET2_ID> --target-group-arns <TG_ARN>
    ```

3. **Create Scaling Policies**:
    ```sh
    aws autoscaling put-scaling-policy --auto-scaling-group-name webASG --policy-name ScaleUp --scaling-adjustment 1 --adjustment-type ChangeInCapacity
    aws autoscaling put-scaling-policy --auto-scaling-group-name webASG --policy-name ScaleDown --scaling-adjustment -1 --adjustment-type ChangeInCapacity
    ```

### 6. Launch a Multi-AZ RDS Database

1. **Create RDS Subnet Group**:
    ```sh
    aws rds create-db-subnet-group --db-subnet-group-name mydbsubnetgroup --subnet-ids <PRIVATE_SUBNET1_ID> <PRIVATE_SUBNET2_ID>
    ```

2. **Create RDS Instance**:
    ```sh
    aws rds create-db-instance --db-instance-identifier mydbinstance --db-subnet-group mydbsubnetgroup --allocated-storage 20 --db-instance-class db.t2.micro --engine mysql --master-username admin --master-user-password password --vpc-security-group-ids <SG_ID> --multi-az
    ```

### 7. Test the Setup

1. **Test Access**:
    - Access the ALB DNS name in a web browser.
    - Verify that the `index.html` page displays the correct message.

## Cleanup

To delete all resources created by this setup, follow these steps:

1. **Delete Auto Scaling Group**:
    ```sh
    aws autoscaling delete-auto-scaling-group --auto-scaling-group-name webASG --force-delete
    aws autoscaling delete-launch-configuration --launch-configuration-name webLC
    ```

2. **Delete Load Balancer and Target Group**:
    ```sh
    aws elbv2 delete-load-balancer --load-balancer-arn <ALB_ARN>
    aws elbv2 delete-target-group --target-group-arn <TG_ARN>
    ```

3. **Delete EC2 Instances**:
    ```sh
    aws ec2 terminate-instances --instance-ids <INSTANCE_ID1> <INSTANCE_ID2>
    ```

4. **Delete NAT Gateways**:
    ```sh
    aws ec2 delete-nat-gateway --nat-gateway-id <NAT_GATEWAY_ID>
    ```

5. **Delete Subnets, Route Tables, Internet Gateway, and VPC**:
    ```sh
    aws ec2 delete-subnet --subnet-id <SUBNET_ID>
    aws ec2 delete-route-table --route-table-id <RT_ID>
    aws ec2 delete-internet-gateway --internet-gateway-id <IGW_ID>
    aws ec2 delete-vpc --vpc-id <VPC_ID>
    ```

6. **Delete RDS Instance**:
    ```sh
    aws rds delete-db-instance --db-instance-identifier mydbinstance --skip-final-snapshot
    aws rds delete-db-subnet-group --db-subnet-group-name mydbsubnetgroup
    ```

Congratulations! You have successfully deployed and tested a multi-tier web application in a custom VPC on AWS.
