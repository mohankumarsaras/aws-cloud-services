# AWS Migration - Organization, Architecture, and IAM Isolation

This document outlines the AWS migration strategy for the MintoCred API and UI. It explicitly defines the account-level isolation under AWS Organizations, VPC isolation for each environment, and the specific IAM users, roles, and resource-scoped policies required to manage these resources securely.

## 1. AWS Organizations & Account Structure

To ensure maximum security and billing separation, the infrastructure is managed under a central **AWS Organization** with multiple isolated child accounts.

| Account Level | Account Name | Purpose | Authorized Users |
| :--- | :--- | :--- | :--- |
| **Management** | `MintoCred-Org-Root` | Billing, SCP management, and creating child accounts. | Root User, Organization Admins |
| **Workload OU** | `MintoCred-Dev` | Hosts the completely private Development environment. | Dev Admins, Developers |
| **Workload OU** | `MintoCred-Staging` | Hosts the completely private Staging/QA environment. | Staging Admins, QA |
| **Workload OU** | `MintoCred-Production` | Hosts the publicly accessible Production environment. | Prod Admins, SREs |

## 2. Environment & VPC Resource Isolation

Each account contains a single, isolated VPC.

### Dev and Staging Accounts (`MintoCred-Dev`, `MintoCred-Staging`)
- **VPC Design**: Strictly Private. No public endpoints.
- **Access**: Developers access the API and UI internally via **AWS Client VPN** or a secure **Bastion Host**.

### Production Account (`MintoCred-Production`)
- **VPC Design**: Public/Private Hybrid.
- **Private Subnets**: Backend logic (ECS Tasks) and Database (EC2) are strictly placed in Private Subnets.
- **Public Subnets**: Reserved exclusively for the **Application Load Balancer (ALB)** and **NAT Gateways**. The UI is hosted on S3 and exposed globally via a public **CloudFront CDN**.

## 3. Accessed Resource Summary (By Account Level)

To adhere to the principle of least privilege, IAM policies must target specific resources rather than using the wildcard `*`. The table below defines the naming conventions and ARNs for the resources across all workload accounts (Dev, Staging, Prod).

*(Note: Replace `REGION` with your AWS region, e.g., `us-east-1`, and `ACCOUNT_ID` with the respective 12-digit AWS account ID).*

| Service | Resource Name / Convention | Target ARN for IAM JSON |
| :--- | :--- | :--- |
| **VPC / Networking** | `mintocred-vpc` | `arn:aws:ec2:REGION:ACCOUNT_ID:vpc/*` (plus subnet/route table ARNs) |
| **EC2 (Database/Bastion)** | `mintocred-db-instance` | `arn:aws:ec2:REGION:ACCOUNT_ID:instance/*` |
| **ECS Cluster** | `mintocred-cluster` | `arn:aws:ecs:REGION:ACCOUNT_ID:cluster/mintocred-*` |
| **ECS Service / Task** | `mintocred-api-service` | `arn:aws:ecs:REGION:ACCOUNT_ID:service/mintocred-*/*` <br> `arn:aws:ecs:REGION:ACCOUNT_ID:task/mintocred-*/*` |
| **ECR Repository** | `mintocred-api-repo` | `arn:aws:ecr:REGION:ACCOUNT_ID:repository/mintocred-*` |
| **S3 (UI Bucket)** | `mintocred-[env]-ui-bucket` | `arn:aws:s3:::mintocred-*-ui-bucket` <br> `arn:aws:s3:::mintocred-*-ui-bucket/*` |
| **CloudFront** | `mintocred-ui-cdn` | `arn:aws:cloudfront::ACCOUNT_ID:distribution/*` |
| **Route 53** | `mintocred.com` Zones | `arn:aws:route53:::hostedzone/*` |
| **IAM Execution Roles** | `ecsTaskExecutionRole` | `arn:aws:iam::ACCOUNT_ID:role/mintocred-*` |

## 4. Resource-Scoped IAM Policies

The following JSON policies limit the Environment Admins and CI/CD pipelines to exactly the resources defined above, preventing them from interacting with or deleting unrelated AWS resources.

### A. JSON Policy: `MintoCredEnvironmentAdmin`

*This policy should be attached to the Admin user in the respective Dev, Staging, or Prod account. It replaces broad `*` access with resource-specific ARNs.*

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VPCandEC2Management",
            "Effect": "Allow",
            "Action": [
                "ec2:RunInstances",
                "ec2:StartInstances",
                "ec2:StopInstances",
                "ec2:TerminateInstances",
                "ec2:Describe*",
                "ec2:CreateVpc",
                "ec2:CreateSubnet"
            ],
            "Resource": [
                "arn:aws:ec2:REGION:ACCOUNT_ID:instance/*",
                "arn:aws:ec2:REGION:ACCOUNT_ID:vpc/*",
                "arn:aws:ec2:REGION:ACCOUNT_ID:subnet/*",
                "arn:aws:ec2:REGION:ACCOUNT_ID:security-group/*",
                "arn:aws:ec2:REGION:ACCOUNT_ID:volume/*"
            ]
        },
        {
            "Sid": "ContainerServicesManagement",
            "Effect": "Allow",
            "Action": [
                "ecs:CreateCluster",
                "ecs:DeleteCluster",
                "ecs:CreateService",
                "ecs:UpdateService",
                "ecs:DeleteService",
                "ecs:RegisterTaskDefinition",
                "ecs:Describe*",
                "ecr:CreateRepository",
                "ecr:DeleteRepository",
                "ecr:*"
            ],
            "Resource": [
                "arn:aws:ecs:REGION:ACCOUNT_ID:cluster/mintocred-*",
                "arn:aws:ecs:REGION:ACCOUNT_ID:service/mintocred-*/*",
                "arn:aws:ecs:REGION:ACCOUNT_ID:task-definition/mintocred-*",
                "arn:aws:ecr:REGION:ACCOUNT_ID:repository/mintocred-*"
            ]
        },
        {
            "Sid": "FrontendHostingManagement",
            "Effect": "Allow",
            "Action": [
                "s3:*",
                "cloudfront:*"
            ],
            "Resource": [
                "arn:aws:s3:::mintocred-*-ui-bucket",
                "arn:aws:s3:::mintocred-*-ui-bucket/*",
                "arn:aws:cloudfront::ACCOUNT_ID:distribution/*"
            ]
        },
        {
            "Sid": "IAMRolePassForECS",
            "Effect": "Allow",
            "Action": [
                "iam:PassRole",
                "iam:GetRole"
            ],
            "Resource": "arn:aws:iam::ACCOUNT_ID:role/mintocred-*"
        }
    ]
}
```

### B. JSON Policy: `GitHubActionsDeployRole` (CI/CD Only)

*This policy is strictly for the deployment pipeline, scoped to exact resource ARNs.*

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "ECRPushPull",
            "Effect": "Allow",
            "Action": [
                "ecr:GetAuthorizationToken",
                "ecr:BatchCheckLayerAvailability",
                "ecr:GetDownloadUrlForLayer",
                "ecr:InitiateLayerUpload",
                "ecr:UploadLayerPart",
                "ecr:CompleteLayerUpload",
                "ecr:PutImage"
            ],
            "Resource": "arn:aws:ecr:REGION:ACCOUNT_ID:repository/mintocred-api-repo"
        },
        {
            "Sid": "ECSUpdateService",
            "Effect": "Allow",
            "Action": [
                "ecs:UpdateService",
                "ecs:DescribeServices"
            ],
            "Resource": "arn:aws:ecs:REGION:ACCOUNT_ID:service/mintocred-*/mintocred-api-service"
        },
        {
            "Sid": "S3UploadUI",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:DeleteObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::mintocred-*-ui-bucket",
                "arn:aws:s3:::mintocred-*-ui-bucket/*"
            ]
        },
        {
            "Sid": "CloudFrontInvalidation",
            "Effect": "Allow",
            "Action": [
                "cloudfront:CreateInvalidation"
            ],
            "Resource": "arn:aws:cloudfront::ACCOUNT_ID:distribution/*"
        }
    ]
}
```
