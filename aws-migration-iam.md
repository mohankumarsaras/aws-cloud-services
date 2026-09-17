# AWS Migration - Organization, Architecture, and IAM Isolation

This document outlines the AWS migration strategy for the MintoCred API and UI. It explicitly defines the account-level isolation under AWS Organizations, VPC isolation for each environment, and the specific IAM users, roles, and policies required to manage these resources securely.

## 1. AWS Organizations & Account Structure

To ensure maximum security and billing separation, the infrastructure is managed under a central **AWS Organization** with multiple isolated child accounts.

| Account Level | Account Name | Purpose | Authorized Users |
| :--- | :--- | :--- | :--- |
| **Management** | `MintoCred-Org-Root` | Billing, SCP (Service Control Policies) management, and creating child accounts. Hosts no workloads. | Root User, Organization Admins |
| **Workload OU** | `MintoCred-Dev` | Hosts the completely private Development environment. | Dev Admins, Developers |
| **Workload OU** | `MintoCred-Staging` | Hosts the completely private Staging/QA environment. | Staging Admins, QA |
| **Workload OU** | `MintoCred-Production` | Hosts the publicly accessible Production environment. | Prod Admins, SREs |

## 2. Environment & VPC Resource Isolation

Each account contains a single, isolated VPC. The networking rules dictate how resources are exposed.

### Dev and Staging Accounts (`MintoCred-Dev`, `MintoCred-Staging`)
- **VPC Design**: Strictly Private.
- **Subnets**: All resources (ECS Tasks, EC2 Databases, internal ALBs) reside in **Private Subnets**.
- **Access**: There are **no public IP addresses** and no Internet Gateways for inbound traffic. Developers access the API and UI internally via **AWS Client VPN** or a secure **Bastion Host** (EC2) residing in a public subnet only for SSH/Tunneling.

### Production Account (`MintoCred-Production`)
- **VPC Design**: Public/Private Hybrid.
- **Private Subnets**: The backend logic (ECS Tasks) and the Database (EC2) are strictly placed in Private Subnets, isolated from direct internet access.
- **Public Subnets**: Reserved exclusively for the **Application Load Balancer (ALB)** and **NAT Gateways**.
- **Access**: The ALB receives public HTTPS traffic and securely routes it to the private ECS tasks. The UI is hosted on S3 and exposed globally via a public **CloudFront CDN**.

## 3. IAM Users, Roles, and Accessed Resources

Each account requires specific IAM identities with scoped permissions. Cross-account access is denied by default.

### A. Management Account
- **User**: `OrganizationAdmin`
- **Accessed Resources**: AWS Organizations, Billing, AWS SSO (Identity Center).
- **Policies Needed**: `AWSOrganizationsFullAccess`, `Billing`.

### B. Dev & Staging Accounts
- **User/Role**: `EnvironmentAdmin` (e.g., `DevAdmin`)
- **Accessed Resources**: VPC, EC2 (Database & Bastion), ECS, ECR, internal ALBs, S3 (UI Bucket), Route 53 (Private Hosted Zone), ACM.
- **Policies Needed**: See Section 4 (Custom Admin Policy). CloudFront permissions can be restricted since Dev/Staging UIs are typically accessed directly via S3/ALB over VPN, but if CloudFront is used, it requires WAF IP-restrictions.

### C. Production Account
- **User/Role**: `ProdAdmin`
- **Accessed Resources**: VPC, EC2 (Database), ECS, ECR, Public ALBs, S3 (UI Bucket), CloudFront (Public CDN), Route 53 (Public Hosted Zone), ACM, WAF.
- **Policies Needed**: See Section 4. Stricter limits can be applied to prevent manual deletion of Production EC2 databases without MFA.

### D. CI/CD Deployment Role (Across Dev/Staging/Prod)
- **Role**: `GitHubActionsDeployRole` (Assuming GitHub Actions is used for CI/CD).
- **Accessed Resources**: ECR (Push images), ECS (Update Service), S3 (Upload UI files), CloudFront (Create Invalidation).
- **Policies Needed**: Highly scoped custom policy (no VPC/EC2 creation rights).

## 4. List of Services and Necessary IAM Policies

The following details the services used per account and the exact JSON policy required for the `EnvironmentAdmin` (Dev/Staging/Prod Admin) to build and manage the infrastructure.

### Services Used
1. **Amazon EC2 & VPC**: For Database hosting, Bastion Hosts, and networking infrastructure.
2. **Amazon ECS & ECR**: For running and storing the containerized FastAPI backend.
3. **Elastic Load Balancing (ELB)**: Internal ALBs for Dev/Staging; Public ALB for Prod.
4. **Amazon S3 & CloudFront**: For storing and serving the static React/Vue frontend UI.
5. **Amazon Route 53 & ACM**: For DNS routing and SSL certificate provisioning.

### JSON Policy: `MintoCredEnvironmentAdmin`

*This policy should be attached to the Admin user in the respective Dev, Staging, or Prod account.*

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VPCandEC2Management",
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
            "Sid": "DNSAndSSLManagement",
            "Effect": "Allow",
            "Action": [
                "route53:*",
                "route53domains:*",
                "acm:*"
            ],
            "Resource": "*"
        },
        {
            "Sid": "IAMRolePassForECSAndCloudFront",
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

### JSON Policy: `GitHubActionsDeployRole` (CI/CD Only)

*This policy is strictly for the deployment pipeline, ensuring it can only update code, not destroy infrastructure.*

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
                "ecr:GetRepositoryPolicy",
                "ecr:DescribeRepositories",
                "ecr:ListImages",
                "ecr:DescribeImages",
                "ecr:BatchGetImage",
                "ecr:InitiateLayerUpload",
                "ecr:UploadLayerPart",
                "ecr:CompleteLayerUpload",
                "ecr:PutImage"
            ],
            "Resource": "*"
        },
        {
            "Sid": "ECSUpdateService",
            "Effect": "Allow",
            "Action": [
                "ecs:UpdateService",
                "ecs:DescribeServices"
            ],
            "Resource": "*"
        },
        {
            "Sid": "S3UploadUI",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:DeleteObject",
                "s3:ListBucket"
            ],
            "Resource": "arn:aws:s3:::mintocred-ui-bucket-name/*"
        },
        {
            "Sid": "CloudFrontInvalidation",
            "Effect": "Allow",
            "Action": [
                "cloudfront:CreateInvalidation"
            ],
            "Resource": "*"
        }
    ]
}
```
