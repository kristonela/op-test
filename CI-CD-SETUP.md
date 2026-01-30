# CI/CD Pipeline Setup Guide

This guide explains how to set up automated Terraform deployments using GitHub Actions or GitLab CI.

## Overview

The CI/CD pipelines automatically deploy your infrastructure when changes are pushed to the `main` branch. The pipelines:

- ✅ Validate Terraform configuration
- ✅ Run security scans (GitLab only)
- ✅ Generate and review Terraform plans
- ✅ Apply changes automatically (or manually for production)
- ✅ Support multiple environments (AWS, Kubernetes)

## Prerequisites

### For Both Platforms

1. **Terraform State Backend**: S3 bucket for storing Terraform state
   ```bash
   aws s3 mb s3://your-terraform-state-bucket --region eu-west-1
   ```

2. **AWS Credentials**: IAM user or role with necessary permissions

### AWS IAM Permissions

Create an IAM policy with these permissions (minimum):
- EC2, EKS, RDS, VPC management
- S3 access for state backend
- IAM for service roles
- Route53 for DNS (if used)

## GitHub Actions Setup

### 1. Repository Secrets

Go to **Settings → Secrets and variables → Actions** and add:

| Secret Name | Description | Example |
|------------|-------------|---------|
| `AWS_ROLE_ARN` | AWS IAM Role ARN for OIDC (recommended) | `arn:aws:iam::123456789012:role/GitHubActionsRole` |
| `AWS_ACCESS_KEY_ID` | AWS Access Key (fallback) | `AKIAIOSFODNN7EXAMPLE` |
| `AWS_SECRET_ACCESS_KEY` | AWS Secret Key (fallback) | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY` |
| `AWS_REGION` | AWS Region | `eu-west-1` |
| `TF_STATE_BUCKET` | S3 bucket for Terraform state | `my-terraform-state-bucket` |

### 2. OIDC Configuration (Recommended)

Using OIDC is more secure than static credentials. Set up AWS IAM Identity Provider:

```bash
# Create OIDC provider in AWS
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1

# Create IAM role with trust policy
cat > trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG/YOUR_REPO:*"
        }
      }
    }
  ]
}
EOF

aws iam create-role \
  --role-name GitHubActionsRole \
  --assume-role-policy-document file://trust-policy.json

# Attach necessary policies
aws iam attach-role-policy \
  --role-name GitHubActionsRole \
  --policy-arn arn:aws:iam::aws:policy/PowerUserAccess
```

### 3. Workflow Usage

The pipeline runs automatically on:
- **Push to main**: Runs plan and apply
- **Pull requests**: Runs plan only (with PR comment)
- **Manual trigger**: Choose plan, apply, or destroy

**Manual Trigger:**
```
Actions → Terraform Deploy → Run workflow → Select action
```

### 4. Testing

```bash
# Test locally before committing
cd templates/terraform/aws
terraform init
terraform plan
```

## GitLab CI Setup

### 1. CI/CD Variables

Go to **Settings → CI/CD → Variables** and add:

| Variable Name | Description | Protected | Masked |
|--------------|-------------|-----------|--------|
| `AWS_ACCESS_KEY_ID` | AWS Access Key | ✅ | ✅ |
| `AWS_SECRET_ACCESS_KEY` | AWS Secret Key | ✅ | ✅ |
| `AWS_REGION` | AWS Region | ❌ | ❌ |
| `TF_STATE_BUCKET` | S3 bucket for Terraform state | ✅ | ❌ |

### 2. Pipeline Behavior

| Event | Behavior |
|-------|----------|
| **Merge Request** | Runs validate & plan (manual apply) |
| **Push to main** | Runs validate, plan, auto-apply for AWS |
| **Manual trigger** | Can run destroy (requires manual approval) |

### 3. Pipeline Stages

```
validate → plan → apply → destroy
```

- **validate**: Format check, Terraform validate, security scan (tfsec)
- **plan**: Generate Terraform plan with artifacts
- **apply**: Deploy infrastructure (auto for AWS, manual for K8s)
- **destroy**: Teardown infrastructure (always manual)

### 4. Security Scanning

The pipeline includes `tfsec` for security scanning:
- Detects hardcoded credentials
- Checks for insecure configurations
- Validates encryption settings
- Generates SAST reports in Security Dashboard

### 5. Testing

```bash
# Validate pipeline configuration
gitlab-ci-lint .gitlab-ci.yml

# Test locally with GitLab Runner
gitlab-runner exec docker validate:format
```

## Common Configuration

### Terraform Variables

Both pipelines pass these variables automatically:
```hcl
region = var.region  # From AWS_REGION secret
```

Additional variables can be set in `terraform.auto.tfvars` or passed via pipeline.

### Backend Configuration

The S3 backend is configured automatically:
```hcl
backend "s3" {
  bucket  = "<from TF_STATE_BUCKET>"
  key     = "<environment>/terraform.tfstate"
  region  = "<from AWS_REGION>"
  encrypt = true
}
```

### Multiple Environments

To add more environments, modify:

**GitHub Actions** (`.github/workflows/terraform-deploy.yml`):
```yaml
strategy:
  matrix:
    environment: [aws, kubernetes, staging]
```

**GitLab CI** (`.gitlab-ci.yml`):
```yaml
# Duplicate jobs with different names
plan:staging:
  extends: .terraform_base
  variables:
    TF_ENVIRONMENT: "staging"
```

## Troubleshooting

### Problem: "Error acquiring state lock"

**Solution**: Another pipeline is running or crashed. Remove lock:
```bash
terraform force-unlock <LOCK_ID>
```

### Problem: "Access Denied" errors

**Solution**: Check IAM permissions for the role/user:
```bash
aws sts get-caller-identity
aws iam get-role --role-name GitHubActionsRole
```

### Problem: "Backend initialization failed"

**Solution**: Verify S3 bucket exists and credentials are correct:
```bash
aws s3 ls s3://your-terraform-state-bucket
```

### Problem: Pipeline runs but doesn't apply

**GitHub**: Check that workflow has `id-token: write` permission
**GitLab**: Ensure job has `when: on_success` or is not set to `manual`

## Best Practices

1. **Protected Branches**: Enable branch protection for `main`
2. **Required Reviews**: Require PR approvals before merging
3. **State Locking**: Always enabled by default (uses DynamoDB)
4. **Secrets Management**: Never commit secrets, use CI/CD variables
5. **Plan Review**: Always review Terraform plans before applying
6. **Incremental Changes**: Make small, testable changes
7. **Rollback Plan**: Keep previous Terraform state files

## Pipeline Files Location

After extracting the generated zip:
```
your-repo/
├── .github/
│   └── workflows/
│       └── terraform-deploy.yml
├── .gitlab-ci.yml
├── terraform/
│   ├── aws/
│   └── kubernetes/
├── argocd/
└── helm/
```

## Testing Your Pipeline

Use the included test script:
```bash
./test-pipeline.sh github  # Test GitHub Actions locally
./test-pipeline.sh gitlab  # Test GitLab CI locally
```

## Additional Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitLab CI Documentation](https://docs.gitlab.com/ee/ci/)
- [Terraform Best Practices](https://www.terraform.io/docs/cloud/guides/recommended-practices/index.html)
- [AWS OIDC for GitHub Actions](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)

## Support

For issues or questions:
1. Check the [openprime-infra-templates](https://github.com/devopsgroupeu/openprime-infra-templates) repository
2. Review pipeline logs in your CI/CD platform
3. Open an issue with pipeline logs and error messages
