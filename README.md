# AWS-Immersion-Day for CNF Partner/Customer in Telco

### Please keep in mind..
* You only have to use 'us-east-1' Region not the one closest to you.

## 1. Please download CFN template for this session
* CFN repo: https://github.com/crosscom/AWS-Immersion-Day/blob/main/template/aws-immersion-infra.yaml
* This creates VPC, public/private subnets, subnet route tables, IGW, NAT-GW, Security Groups, EKS Cluster and Bastion Instance

## 2. Log in to AWS Event Engine 

* https://dashboard.eventengine.run/dashboard.
* Please put Event Hash (will be given at the session from your instructor). 
* Please use OTP authentication with your company email.
* Click "Set Team Name" (Team is equivalent to the account)
    * Please put TEAM name to be your name.  
* Click "SSH key" 
    * Download **ee-default-keypair** to your PC (just in case, copy key material to notepad as well).
* Click "AWS Console"
    * Copy credentials (export AWS_DEFAULT_REGION=..) 
    * Click "Open AWS Console".
![Dashboard](Lab1/images/dashboard.png)

## What we will learn today? 
* **[Lab1](https://github.com/crosscom/AWS-Immersion-Day/tree/main/Lab1)**: EKS environment creation (VPC, Subnets, EKS cluster, Multus-ready worker node) for CNF deployment.
* **[Lab2](https://github.com/crosscom/AWS-Immersion-Day/tree/main/Lab2)**: Creating dummy multi-homed Pod application (multus) from the image from ECR.
