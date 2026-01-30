# OpenPrime Generated Infrastructure

This is your generated infrastructure package from OpenPrime. It contains everything you need to deploy your AWS-based Kubernetes environment.

## What's Included

- **terraform/** - Infrastructure as Code for AWS and Kubernetes
- **argocd/** - GitOps application manifests
- **helm/** - Kubernetes application value files
- **.github/workflows/** - GitHub Actions CI/CD pipeline
- **.gitlab-ci.yml** - GitLab CI pipeline
- **test-pipeline.sh** - Pipeline testing script
- **CI-CD-SETUP.md** - Detailed CI/CD setup instructions

## Quick Start

### 1. Initialize Your Repository

```bash
# Extract the zip file
unzip your-infrastructure.zip
cd your-infrastructure

# Initialize git (if not already done)
git init
git add .
git commit -m "Initial infrastructure configuration"

# Add your remote repository
git remote add origin https://github.com/your-org/your-repo.git
# OR for GitLab
git remote add origin https://gitlab.com/your-org/your-repo.git

git push -u origin main
```

### 2. Configure CI/CD

Follow the detailed instructions in **[CI-CD-SETUP.md](CI-CD-SETUP.md)** to:
- Set up AWS credentials
- Configure repository secrets
- Enable the CI/CD pipeline

**Quick reference:**

<details>
<summary>GitHub Actions</summary>

Go to **Settings → Secrets and variables → Actions** and add:
- `AWS_ROLE_ARN` or (`AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`)
- `TF_STATE_BUCKET`
- `AWS_REGION`

</details>

<details>
<summary>GitLab CI</summary>

Go to **Settings → CI/CD → Variables** and add:
- `AWS_ACCESS_KEY_ID` (Protected, Masked)
- `AWS_SECRET_ACCESS_KEY` (Protected, Masked)
- `TF_STATE_BUCKET` (Protected)
- `AWS_REGION`

</details>

### 3. Test Before Deployment

```bash
# Make the test script executable
chmod +x test-pipeline.sh

# Test your chosen platform
./test-pipeline.sh github   # For GitHub Actions
# OR
./test-pipeline.sh gitlab   # For GitLab CI
```

### 4. Deploy

```bash
# Commit and push to trigger deployment
git add .
git commit -m "Configure CI/CD"
git push origin main
```

The pipeline will automatically:
1. Validate Terraform configuration
2. Generate and review a plan
3. Apply the infrastructure changes

## Manual Deployment (Optional)

If you prefer to deploy manually without CI/CD:

```bash
cd terraform/aws

# Initialize Terraform
terraform init \
  -backend-config="bucket=your-terraform-state-bucket" \
  -backend-config="region=eu-west-1"

# Review the plan
terraform plan

# Apply the changes
terraform apply
```

## Directory Structure

```
.
├── .github/
│   └── workflows/
│       └── terraform-deploy.yml    # GitHub Actions workflow
├── .gitlab-ci.yml                  # GitLab CI pipeline
├── terraform/
│   ├── aws/                        # AWS infrastructure
│   │   ├── _config.tf              # Terraform & provider config
│   │   ├── _variables.tf           # Variable definitions
│   │   ├── terraform.auto.tfvars   # Variable values
│   │   ├── vpc.tf                  # VPC configuration
│   │   ├── eks.tf                  # EKS cluster
│   │   ├── database.tf             # RDS/Aurora
│   │   └── ...                     # Other AWS resources
│   └── kubernetes/                 # Kubernetes resources
│       ├── _config.tf              # Terraform config
│       └── argocd.tf               # ArgoCD bootstrap
├── argocd/
│   ├── infra-apps/                 # Infrastructure applications
│   │   ├── nginx-ingress-controller.yaml
│   │   ├── cert-manager.yaml
│   │   ├── prometheus-stack.yaml
│   │   └── ...
│   └── example-apps/               # Example applications
├── helm/                           # Helm values files
│   ├── nginx-ingress-controller/
│   ├── cert-manager/
│   ├── prometheus-stack/
│   └── ...
├── CI-CD-SETUP.md                  # Detailed CI/CD instructions
├── test-pipeline.sh                # Pipeline testing script
└── README.md                       # This file
```

## Terraform Components

### AWS Infrastructure (`terraform/aws/`)

The AWS infrastructure includes:

| Component | Description | Conditional |
|-----------|-------------|-------------|
| VPC | Multi-AZ VPC with public/private subnets | Always |
| EKS | Kubernetes cluster with managed node groups | Optional |
| RDS/Aurora | PostgreSQL database | Optional |
| S3 | Object storage buckets | Optional |
| ECR | Container registries | Optional |
| CloudFront | CDN distribution | Optional |
| Route53 | DNS management | Optional |
| Karpenter | Cluster autoscaler | Optional |
| Monitoring | CloudWatch alarms & dashboards | Optional |

### Kubernetes Resources (`terraform/kubernetes/`)

Bootstraps ArgoCD for GitOps-based application management.

## ArgoCD Applications

Infrastructure applications in `argocd/infra-apps/`:
- **nginx-ingress-controller** - Ingress for external traffic
- **cert-manager** - Automatic TLS certificate management
- **aws-load-balancer-controller** - AWS ALB/NLB integration
- **prometheus-stack** - Monitoring and alerting
- **loki** - Log aggregation
- **promtail** - Log collection
- **karpenter** - Node autoscaling

## Customization

### Modifying Terraform Variables

Edit `terraform/aws/terraform.auto.tfvars` to customize your infrastructure:

```hcl
region = "eu-west-1"

global_tags = {
  Environment = "production"
  ManagedBy   = "terraform"
}

# Enable/disable services
enable_eks = true
enable_rds = true
enable_elasticache = false
```

### Updating Helm Values

Modify values in `helm/<component>/values.yaml` to customize application configurations.

### Adding ArgoCD Applications

Create new application manifests in `argocd/infra-apps/`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://charts.example.com
    chart: my-chart
    targetRevision: 1.0.0
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
```

## Troubleshooting

### Pipeline Failures

1. **Check logs** in GitHub Actions or GitLab CI
2. **Verify secrets** are configured correctly
3. **Test locally** using `test-pipeline.sh`
4. **Review Terraform plan** for any configuration issues

### Terraform State Lock

If you see "Error acquiring state lock":

```bash
# List locks
aws dynamodb scan --table-name terraform-state-lock

# Force unlock (use with caution)
cd terraform/aws
terraform force-unlock <LOCK_ID>
```

### AWS Permissions

Ensure your AWS credentials have necessary permissions:
- EC2, VPC, EKS, RDS management
- S3 access for Terraform state
- IAM for service roles

## Support & Documentation

- **CI/CD Setup**: [CI-CD-SETUP.md](CI-CD-SETUP.md)
- **Terraform Docs**: https://www.terraform.io/docs
- **AWS Provider**: https://registry.terraform.io/providers/hashicorp/aws
- **ArgoCD**: https://argo-cd.readthedocs.io
- **OpenPrime**: https://github.com/devopsgroupeu/openprime-infra-templates

## Next Steps

1. ✅ Configure CI/CD secrets
2. ✅ Test pipelines locally
3. ✅ Push to main branch
4. ✅ Monitor deployment
5. ✅ Access your Kubernetes cluster
6. ✅ Configure applications via ArgoCD

---

**Generated by OpenPrime** - Production-ready infrastructure templates
