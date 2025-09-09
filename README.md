# AWS EKS Cluster (us-east-1 / North Virginia) — Terraform repo README

This repository contains Terraform code and instructions to **create an Amazon EKS cluster in the North Virginia region (`us-east-1`)**.
It’s designed to be a practical starting point for production-ready improvements (VPC sizing, security hardening, CI/CD integration, etc.).

> ⚠️ **Cost reminder:** Creating EKS clusters, node groups, NAT gateways, load balancers, and EC2 instances will incur AWS charges. Destroy resources when not needed: `terraform destroy`.

---

## Repository structure (example)

```
.
├── README.md                    # (this file)
├── modules/
│   ├── vpc/                     # optional - reusable VPC module
│   ├── eks/                     # optional - eks module that wires everything
│   └── node_group/              # optional - managed/node group module
├── env/
│   └── us-east-1/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── terraform.tfvars.example
└── scripts/
    └── kubeconfig.sh            # helper to update kubeconfig
```

> You can adapt this layout to a monolithic root `main.tf` or a modular design — both approaches are supported.

---

## Features included

* EKS cluster creation (`aws_eks_cluster`)
* Managed node group(s) (Amazon EKS Managed Node Groups) using `aws_eks_node_group` **or** self-managed worker group via `aws_autoscaling_group` (choose implementation in the TF code)
* VPC with public/private subnets (recommended) or ability to use an existing VPC
* Security groups for control plane and nodes
* IAM roles & policies for EKS control plane and node role
* Outputs: cluster name, cluster endpoint, cluster CA, node IAM role ARNs
* `kubeconfig` helper to point `kubectl` to the new cluster

---

## Prerequisites

* An **AWS account** with permissions to create: VPCs, Subnets, EKS, EC2 instances, IAM roles, Auto Scaling, Security Groups, ELBs, CloudWatch, and necessary service-linked roles.
* \[Terraform CLI v1.1+] (use latest stable release if possible)
* \[AWS CLI v2]
* \[kubectl] (matching Kubernetes version)
* Optional: \[eksctl] (useful but not required if you use Terraform)
* Configure AWS credentials (via `aws configure`, environment variables, or an assumed role):

  ```bash
  export AWS_PROFILE=your-profile
  export AWS_REGION=us-east-1
  ```

**Minimum IAM permissions:** AWS Managed policies for EKS admin or equivalent custom policy. For local testing you can use an account with `AdministratorAccess` (not recommended for production).

---

## Quick start (example using `env/us-east-1`)

1. Clone the repo

```bash
git clone https://github.com/your-username/your-repo.git
cd your-repo/env/us-east-1
```

2. Inspect `variables.tf` and create `terraform.tfvars` (or copy the example)

```bash
cp terraform.tfvars.example terraform.tfvars
# Edit terraform.tfvars to match your desired config
```

3. Initialize Terraform

```bash
terraform init
```

4. (Optional) Validate & plan

```bash
terraform validate
terraform plan -out plan.tfplan
```

5. Apply

```bash
terraform apply "plan.tfplan"   # if you saved plan
# OR
terraform apply -auto-approve
```

6. Configure kubectl to use the cluster:

```bash
# Using aws cli
aws eks --region us-east-1 update-kubeconfig --name <cluster-name> --alias <optional-alias>

# Or use provided helper
./scripts/kubeconfig.sh <cluster-name> us-east-1
```

7. Verify

```bash
kubectl get nodes
kubectl get pods -A
```

---

## Example `terraform.tfvars.example`

```hcl
# Basic cluster info
cluster_name   = "petsop-eks-cluster"
kubernetes_version = "1.28"        # set desired k8s version supported by EKS
region         = "us-east-1"

# VPC (if creating new VPC)
vpc_cidr       = "10.0.0.0/16"
public_subnets = ["10.0.0.0/24","10.0.1.0/24","10.0.2.0/24"]
private_subnets= ["10.0.10.0/24","10.0.11.0/24","10.0.12.0/24"]

# Node group
node_group_name = "ng-default"
node_instance_type = "t3.medium"
node_min_size   = 2
node_max_size   = 3
node_desired_capacity = 2
node_ami_type   = "AL2_x86_64"     # or AL2_x86_64, AL2_ARM_64, CUSTOM, etc.
```

---

## Important variables to review

* `cluster_name` — EKS cluster name
* `region` — must be `us-east-1` for North Virginia (you requested)
* `kubernetes_version` — choose supported EKS version
* `node_instance_type`, `node_min_size`, `node_max_size` — scaling & cost
* VPC CIDR & Subnet CIDRs — must not conflict with your network

---

## Outputs (typical)

Your Terraform `outputs.tf` should include:

```hcl
output "cluster_name" {
  value = aws_eks_cluster.this.name
}

output "cluster_endpoint" {
  value = aws_eks_cluster.this.endpoint
}

output "cluster_ca" {
  value = aws_eks_cluster.this.certificate_authority[0].data
}

output "node_role_arn" {
  value = aws_iam_role.node_role.arn
}
```

Use those outputs to generate kubeconfig or for other automation.

---

## Post-install: Configure kubectl / Helm

After `terraform apply`:

```bash
aws eks --region us-east-1 update-kubeconfig --name petsop-eks-cluster
kubectl get nodes
```

Install Helm (if required) and add repos, then deploy apps:

```bash
helm repo add stable https://charts.helm.sh/stable
helm repo update
```

---

## Recommended Jenkins/GitHub CI integration steps

* Store Terraform state in a **remote backend** (S3 + DynamoDB locking) — do **not** keep state locally for team usage.
* Use an IAM role for CI with least privilege to run Terraform.
* Keep secrets out of TF code; use SSM Parameter Store / Secrets Manager / Vault for secrets.
* Use `terraform fmt` and `terraform validate` in CI before `plan`.

---

## Remote state (example backend config)

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "eks/us-east-1/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

Create the S3 bucket and DynamoDB table before `terraform init` or use a bootstrap script.

---

## Security & best practices

* Use **IAM Roles for Service Accounts (IRSA)** for pods that need AWS API access — do not put AWS keys inside containers.
* Use private subnets for worker nodes where possible, with NAT for outbound.
* Keep control plane logging enabled and integrate with CloudWatch or a centralized logging system.
* Regularly upgrade Kubernetes and node AMIs.
* Use least-privilege IAM policies for node roles and cluster admin accounts.
* Set up automated cluster backups (etcd snapshot solutions) and restore procedures.

---

## Troubleshooting

* `kubectl` cannot connect: ensure `aws eks update-kubeconfig` was run and the cluster exists.
* Node status `NotReady`: check the node bootstrap logs, IAM permissions, and security groups.
* `terraform apply` stuck on IAM resources: ensure you have permission to create service-linked roles for EKS.
* `terraform plan` shows replacements for node group frequently — prefer Managed Node Groups or update strategy to reduce replacements.

---

## Destroying the cluster

When you no longer need the EKS cluster:

```bash
terraform destroy -auto-approve
```

Make sure to delete any external resources created outside Terraform (ALBs, ECR repos, S3 buckets with retained objects) or add them to your Terraform configurations.

---

## Versions & compatibility

* Terraform: v1.1+ (prefer latest v1.x)
* AWS Provider: use a recent provider that supports the EKS resources and latest Kubernetes versions
* kubectl: match the Kubernetes minor version e.g., for `1.28.x` use `kubectl` 1.28.x or 1.27.x (compatible)
* AWS CLI v2 recommended

---

## Example: `scripts/kubeconfig.sh`

```bash
#!/bin/bash
# Usage: ./kubeconfig.sh <cluster-name> <region>
set -e
CLUSTER_NAME=${1:-petsop-eks-cluster}
REGION=${2:-us-east-1}
aws eks --region ${REGION} update-kubeconfig --name ${CLUSTER_NAME}
echo "kubectl context set to ${CLUSTER_NAME} in region ${REGION}"
kubectl get nodes
```

---

## LICENSE

This repository is provided under the **MIT License** — adapt and use it as you need.

---

## Next steps (suggested)

* Add Terraform modules in `modules/` for VPC, EKS, and node groups.
* Add remote state backend (S3 + DynamoDB).
* Add CI job (GitHub Actions / Jenkins) to run `terraform fmt`, `terraform validate`, `terraform plan`.
* Add Helm charts or manifests for app deployment (CI/CD pipeline).

---


