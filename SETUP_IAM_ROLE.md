# GitHub Actions IAM Role Setup Guide

This guide explains how to set up the least-privilege IAM role for GitHub Actions to deploy EC2 infrastructure.

## Overview

The `IAMRole.yml` CloudFormation template creates:
1. **GitHub OIDC Provider** — Trusts GitHub's OIDC token server
2. **IAM Role** — `GitHubActionsEC2DeployRole` with minimal permissions
3. **Inline Policy** — Permissions scoped to CloudFormation stack operations and EC2 resource creation

## Step 1: Update Parameters

Before deploying, update `IAMRole.yml` with your GitHub details:

```yaml
Parameters:
  GitHubOrg:
    Default: "your-github-org"        # Change to your GitHub org/username
  
  GitHubRepo:
    Default: "AWSProject30052026"     # Repo name
  
  GitHubBranch:
    Default: "main"                   # Branch allowed to deploy (e.g., "main", "develop")
```

## Step 2: Deploy IAM Role Stack

Run this command in your AWS account (one-time setup):

```bash
aws cloudformation create-stack \
  --stack-name github-actions-iam-role \
  --template-body file://IAMRole.yml \
  --parameters \
    ParameterKey=GitHubOrg,ParameterValue=your-github-org \
    ParameterKey=GitHubRepo,ParameterValue=AWSProject30052026 \
    ParameterKey=GitHubBranch,ParameterValue=main \
  --capabilities CAPABILITY_NAMED_IAM
```

Or via AWS Console:
1. Go to **CloudFormation** → **Create Stack**
2. Upload `IAMRole.yml`
3. Enter your GitHub organization and repository details
4. Check "I acknowledge that AWS CloudFormation might create IAM resources"
5. Click **Create Stack**

Wait for the stack to show `CREATE_COMPLETE`.

## Step 3: Verify the Role

The CloudFormation stack outputs the role ARN. You can also retrieve it:

```bash
aws iam get-role --role-name GitHubActionsEC2DeployRole
```

The `Deploy.yml` workflow already uses this role ARN:
```yaml
role-to-assume: arn:aws:iam::084828603933:role/GitHubActionsEC2DeployRole
```

**Note:** Replace `084828603933` with your AWS Account ID if different.

## Step 4: Deploy EC2 Infrastructure

Push your code and trigger the GitHub Actions workflow:

```bash
git add .
git commit -m "Deploy EC2 via CloudFormation"
git push origin main
```

Go to **GitHub → Actions** and manually trigger **"Deploy CloudFormation EC2-Instance"** workflow, or configure it to run on push.

## Permissions Included

The IAM role has least-privilege access to:

- **CloudFormation:** Create/update/delete stacks matching `GitHubActionsCloudStack*`
- **VPC/Subnets:** Create and manage VPCs, subnets, and route tables
- **Internet Gateway:** Create and attach internet gateways
- **Route Tables:** Create routes to the internet
- **Security Groups:** Create security groups and manage ingress/egress rules
- **EC2 Instances:** Launch, terminate, and tag instances
- **EBS Volumes:** Create, delete, attach, and detach volumes
- **IAM PassRole:** Pass EC2 instance roles (if needed)
- **Read-only:** Describe AZs, regions, and account attributes

## Permissions NOT Included (Least Privilege)

The role intentionally does NOT have:
- Access to S3, Lambda, RDS, or other services
- IAM user/role creation (only PassRole for existing roles)
- CloudFormation stack deletion for other stacks
- EC2 key pair creation/deletion

## Troubleshooting

**Error: "User is not authorized to perform: iam:CreateOIDCProvider"**
- The OIDC provider may already exist in your account. Either:
  - Remove the `GitHubOIDCProvider` resource from `IAMRole.yml` and deploy again, or
  - Reuse the existing OIDC provider by updating the trust policy manually

**Error: "AssumeRole failed"**
- Verify `GitHubOrg`, `GitHubRepo`, and `GitHubBranch` match your actual repository
- Confirm the OIDC provider is created and trusts GitHub

**Error: "CloudFormation operation not permitted"**
- Check the inline policy in `IAMRole.yml` has all required permissions
- Verify the stack name matches the pattern `GitHubActionsCloudStack*`

## Next Steps

1. Configure repository secrets (if needed for AWS credentials elsewhere)
2. Test the workflow by triggering a manual deployment
3. Monitor CloudFormation stack events in AWS Console
4. Verify the EC2 instance is created with correct security groups and volumes

## References

- [GitHub Actions OIDC Provider](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
- [AWS CloudFormation Best Practices](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/best-practices.html)
- [IAM Least Privilege](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#grant-least-privilege)
