# AWS Migration - Organization, Architecture, and IAM Isolation

This document outlines the AWS migration strategy for the MintoCred API and UI, based on the existing infrastructure requirements (Docker, Nginx, MySQL, GitHub Actions, `.env` configurations). It explicitly defines the account-level isolation under AWS Organizations and the specific IAM users, roles, and resource-scoped policies required to manage these resources securely.

## 1. AWS Organizations & Account Structure

To ensure maximum security and billing separation, the infrastructure is managed under a central **AWS Organization** with multiple isolated child accounts.

| Account Level | Account Name | Purpose | Authorized Users |
| :--- | :--- | :--- | :--- |
| **Management** | `MintoCred-Org-Root` | Billing, SCP management, creating child accounts, cross-account IAM. | Root User, Organization Admins |
| **Workload OU** | `MintoCred-Dev` | Hosts the completely private Development environment. | Dev Admins, Developers |
| **Workload OU** | `MintoCred-Staging` | Hosts the completely private Staging/QA environment. | Staging Admins, QA |
| **Workload OU** | `MintoCred-Production` | Hosts the publicly accessible Production environment. | Prod Admins, SREs |

## 2. Environment & VPC Resource Isolation

Each account contains a single, isolated VPC.

### Dev and Staging Accounts (`MintoCred-Dev`, `MintoCred-Staging`)
- **VPC Design**: Strictly Private. No public endpoints.
- **Access**: Developers access the API, UI, and EC2 instances internally via **AWS Client VPN** or **AWS Systems Manager (SSM) Session Manager** (replacing traditional SSH).

### Production Account (`MintoCred-Production`)
- **VPC Design**: Public/Private Hybrid.
- **Private Subnets**: Backend logic (ECS Tasks) and Database (EC2) are placed in Private Subnets. Configuration secrets (`.env`) are fetched securely at runtime from **AWS Systems Manager (SSM) Parameter Store** and **AWS Secrets Manager**.
- **Public Subnets**: Reserved exclusively for the **Application Load Balancer (ALB)** and **NAT Gateways**. The UI is hosted on S3 and exposed globally via a public **CloudFront CDN**.

## 3. Accessed Resource Summary (By Account Level)

The table below defines the naming conventions and ARNs for the maximum breadth of services required across all workload accounts (Dev, Staging, Prod), replacing manual `.env` and SSH management with AWS-native services (SSM, KMS, Secrets Manager).

*(Note: Replace `REGION` with your AWS region, and `ACCOUNT_ID` with the respective 12-digit AWS account ID).*

| Service | Resource Name / Convention | Target ARN for IAM JSON |
| :--- | :--- | :--- |
| **AWS Organizations** | Organization / Accounts | `arn:aws:organizations::ACCOUNT_ID:organization/*` (Read Only for Env Admins) |
| **VPC / Networking** | `mintocred-vpc` | `arn:aws:ec2:REGION:ACCOUNT_ID:vpc/*` (plus subnets/security-groups) |
| **EC2 (Database/App)** | `mintocred-db-instance` | `arn:aws:ec2:REGION:ACCOUNT_ID:instance/*` |
| **ECS Cluster & Tasks** | `mintocred-cluster` | `arn:aws:ecs:REGION:ACCOUNT_ID:cluster/mintocred-*` |
| **ECR Repository** | `mintocred-api-repo` | `arn:aws:ecr:REGION:ACCOUNT_ID:repository/mintocred-*` |
| **SSM Parameter Store** | `/mintocred/dev/*` | `arn:aws:ssm:REGION:ACCOUNT_ID:parameter/mintocred/*` |
| **Secrets Manager** | `mintocred-db-credentials`| `arn:aws:secretsmanager:REGION:ACCOUNT_ID:secret:mintocred-*` |
| **KMS (Key Management)**| `mintocred-encryption-key`| `arn:aws:kms:REGION:ACCOUNT_ID:key/*` |
| **CloudWatch / Logs** | `/ecs/mintocred-api` | `arn:aws:logs:REGION:ACCOUNT_ID:log-group:/ecs/mintocred-*` |
| **S3 (UI Bucket)** | `mintocred-[env]-ui-bucket` | `arn:aws:s3:::mintocred-*-ui-bucket*` |
| **CloudFront** | `mintocred-ui-cdn` | `arn:aws:cloudfront::ACCOUNT_ID:distribution/*` |
| **Route 53** | `mintocred.com` Zones | `arn:aws:route53:::hostedzone/*` |
| **IAM Execution Roles** | `ecsTaskExecutionRole` | `arn:aws:iam::ACCOUNT_ID:role/mintocred-*` |

## 4. Resource-Scoped IAM Policies

The following JSON policies limit the Environment Admins and CI/CD pipelines to exactly the resources defined above, ensuring full AWS-native management (including SSM/Secrets) while maintaining project-level scoping.

### A. JSON Policy: `MintoCredEnvironmentAdmin`

*This policy should be attached to the Admin user in the respective Dev, Staging, or Prod account. It provides comprehensive infrastructure management scoped to the `mintocred` resources.*

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "OrganizationsReadOnly",
            "Effect": "Allow",
            "Action": [
                "organizations:DescribeOrganization",
                "organizations:ListAccounts",
                "organizations:ListOrganizationalUnitsForParent"
            ],
            "Resource": "*"
        },
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
                "ec2:CreateSubnet",
                "elasticloadbalancing:*"
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
            "Sid": "ConfigurationAndSecretsManagement",
            "Effect": "Allow",
            "Action": [
                "ssm:PutParameter",
                "ssm:GetParameter",
                "ssm:DeleteParameter",
                "ssm:StartSession",
                "secretsmanager:CreateSecret",
                "secretsmanager:GetSecretValue",
                "secretsmanager:DeleteSecret",
                "kms:CreateKey",
                "kms:Encrypt",
                "kms:Decrypt",
                "cloudwatch:*",
                "logs:*"
            ],
            "Resource": [
                "arn:aws:ssm:REGION:ACCOUNT_ID:parameter/mintocred/*",
                "arn:aws:secretsmanager:REGION:ACCOUNT_ID:secret:mintocred-*",
                "arn:aws:kms:REGION:ACCOUNT_ID:key/*",
                "arn:aws:logs:REGION:ACCOUNT_ID:log-group:/ecs/mintocred-*"
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
                "arn:aws:s3:::mintocred-*-ui-bucket*",
                "arn:aws:cloudfront::ACCOUNT_ID:distribution/*"
            ]
        },
        {
            "Sid": "DNSAndSSLManagement",
            "Effect": "Allow",
            "Action": [
                "route53:*",
                "route53domains:*",
                "acm:*"
            ],
            "Resource": [
                "arn:aws:route53:::hostedzone/*",
                "arn:aws:acm:REGION:ACCOUNT_ID:certificate/*"
            ]
        },
        {
            "Sid": "IAMRolePassForECS",
            "Effect": "Allow",
            "Action": [
                "iam:PassRole",
                "iam:GetRole",
                "iam:CreateRole",
                "iam:AttachRolePolicy"
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
                "arn:aws:s3:::mintocred-*-ui-bucket*",
                "arn:aws:s3:::mintocred-*-ui-bucket*/*"
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
