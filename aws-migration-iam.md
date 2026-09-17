# AWS Migration - Target Architecture & IAM Permissions

This document outlines the AWS services required for migrating the MintoCred API and UI to a modernized AWS cloud architecture. It defines the strict isolation boundaries across Dev, Staging, and Production environments and provides the necessary IAM permissions to provision these resources.

## 1. Environment & VPC Isolation Strategy

To ensure maximum security and stability, the infrastructure will be segregated into three distinct environments (`dev`, `staging`, and `production`). Each environment will reside in its own isolated AWS Account under **AWS Organizations**, utilizing strict VPC networking rules.

### Dev and Staging Environments (Strictly Private)
- **VPC Design**: Entirely private. These VPCs do not expose any incoming public endpoints.
- **Subnets**: All resources (ECS tasks, EC2 databases, internal ALBs) reside in **Private Subnets**.
- **Access**: Developers and QA access the `dev` and `staging` APIs and UIs internally via **AWS Client VPN**, **Site-to-Site VPN**, or a secure **Bastion Host**.
- **Egress**: Outbound internet access for patching/updates is handled via NAT Gateways.

### Production Environment (Publicly Accessible)
- **VPC Design**: A secure Public/Private hybrid model. 
- **Private Subnets**: The backend logic (ECS tasks) and the database (EC2) are strictly placed in **Private Subnets**, completely isolated from direct internet access.
- **Public Subnets**: Reserved exclusively for the **Application Load Balancer (ALB)** and **NAT Gateways**. The ALB receives public HTTPS traffic and securely routes it to the private ECS tasks.

## 2. Target Architecture Components

| Component | AWS Service | Purpose in Architecture |
| :--- | :--- | :--- |
| **API Application** | **Amazon ECS & ECR** | The Dockerized FastAPI backend is stored in **Amazon ECR** and runs on **Amazon ECS**. In Production, a public ALB distributes traffic; in Dev/Staging, an internal ALB is used. |
| **Database** | **Amazon EC2** | The MySQL database is hosted directly on an **Amazon EC2** instance, utilizing **Amazon EBS** for data storage. Always deployed in Private Subnets across all environments. |
| **Frontend UI** | **Amazon S3 & CloudFront** | Static UI assets are stored in **S3**. In Production, they are served globally via a public **CloudFront** CDN. In Dev/Staging, CloudFront is restricted via AWS WAF or internal routing. |
| **Domain & DNS** | **Amazon Route 53** | Manages internal DNS for Dev/Staging (Private Hosted Zones) and public DNS for Production (Public Hosted Zones). |
| **SSL / HTTPS** | **AWS Certificate Manager (ACM)** | Provisions free SSL certificates for the load balancers and CloudFront distributions across all environments. |

## 3. Account Provisioning (Root Administrator)

The **Root User** (or an Administrator in the Management Account) should perform the following steps:
1. Create three separate AWS Accounts within AWS Organizations: `MintoCred-Dev`, `MintoCred-Staging`, and `MintoCred-Production`.
2. Create an **IAM User** (or Identity Center Role) in each account to act as the infrastructure administrator.
3. Attach the policy outlined below to this IAM User.

## 4. IAM User Permissions

The following JSON policy represents the permissions required for the Infrastructure Admin to provision and manage the architecture in their respective account.

### Policy: `MintoCredCloudArchitect`

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "ComputeAndNetworkingManagement",
            "Effect": "Allow",
            "Action": [
                "ec2:*",
                "elasticloadbalancing:*"
            ],
            "Resource": "*"
        },
        {
            "Sid": "ContainerServicesManagement",
            "Effect": "Allow",
            "Action": [
                "ecs:*",
                "ecr:*",
                "application-autoscaling:*",
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents",
                "logs:DescribeLogGroups"
            ],
            "Resource": "*"
        },
        {
            "Sid": "FrontendHostingManagement",
            "Effect": "Allow",
            "Action": [
                "s3:*",
                "cloudfront:*"
            ],
            "Resource": "*"
        },
        {
            "Sid": "DomainAndSSLManagement",
            "Effect": "Allow",
            "Action": [
                "route53:*",
                "route53domains:*",
                "acm:*"
            ],
            "Resource": "*"
        },
        {
            "Sid": "IAMRoleManagementForECSAndCloudFront",
            "Effect": "Allow",
            "Action": [
                "iam:CreateRole",
                "iam:DeleteRole",
                "iam:GetRole",
                "iam:PassRole",
                "iam:AttachRolePolicy",
                "iam:DetachRolePolicy",
                "iam:PutRolePolicy",
                "iam:DeleteRolePolicy"
            ],
            "Resource": "*"
        }
    ]
}
```

### Explanation of Permissions:
- **ComputeAndNetworkingManagement**: Full access to provision the isolated VPCs (Private/Public subnets), NAT Gateways, EC2 databases, and ALBs.
- **ContainerServicesManagement**: Full access to push images to ECR and deploy the backend to ECS.
- **FrontendHostingManagement**: Full access to S3 (bucket creation) and CloudFront (CDN distribution).
- **DomainAndSSLManagement**: Full access to Route 53 (Internal/Public Hosted Zones) and ACM.
- **IAMRoleManagementForECSAndCloudFront**: Permission to securely pass execution roles to ECS and grant CloudFront access to S3 via Origin Access Controls.
