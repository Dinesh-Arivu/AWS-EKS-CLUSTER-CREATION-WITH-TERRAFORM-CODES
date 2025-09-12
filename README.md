# 🚀 AWS EKS Cluster Creation with Terraform (North Virginia - `us-east-1`)

This repository contains **Terraform configurations** to deploy an **Amazon Elastic Kubernetes Service (EKS)** cluster in the **North Virginia (us-east-1)** region.
It provisions the following:

* EKS Cluster with default VPC subnets
* IAM roles for EKS control plane and worker nodes
* Managed Node Group with `t2.medium` EC2 instances
* Remote state stored in **S3 backend**

---

<img width="780" height="420" alt="AWS EKS Cluster" src="https://github.com/user-attachments/assets/033e219e-d243-4b3c-9883-c7c5b374c469" />

## 📂 Project Structure

```
.
├── main.tf          # Core Terraform code for EKS
├── provider.tf      # AWS provider configuration
├── backend.tf       # Remote state backend (S3)
├── variables.tf     # Input variables (customize cluster name, size, etc.)
├── outputs.tf       # Outputs after cluster creation
└── README.md        # Documentation
```

---

## ⚙️ Prerequisites

* AWS Account with permissions for EKS, IAM, VPC, and EC2
* [Terraform](https://developer.hashicorp.com/terraform/downloads) v1.3+
* [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) v2
* [kubectl](https://kubernetes.io/docs/tasks/tools/) matching EKS version
* S3 bucket for storing remote Terraform state

---

## 🔧 Configuration

### 1. Setup S3 Backend

Update `backend.tf` with your bucket name:

```hcl
terraform {
  backend "s3" {
    bucket = "north-virginia-terraform-staefile-bucket" # Replace with your S3 bucket
    key    = "EKS/terraform.tfstate"
    region = "us-east-1"
  }
}
```

---

### 2. Initialize Terraform

```bash
terraform init
```

---

### 3. Plan Resources

```bash
terraform plan -out eks-plan
```

---

### 4. Apply Resources

```bash
terraform apply "eks-plan"
```

---

### 5. Configure `kubectl`

Once the cluster is ready, update your kubeconfig:

```bash
aws eks --region us-east-1 update-kubeconfig --name EKS_CLOUD
```

Verify:

```bash
kubectl get nodes
kubectl get pods -A
```

---

## 📜 Outputs

After `terraform apply`, you’ll get values like:

* Cluster name
* Endpoint URL
* Node group details
* IAM Role ARNs

---

## 🔐 Security Notes

* This setup uses the **default VPC** (for simplicity). In production, create dedicated VPCs with private/public subnets.
* Store sensitive state securely (encrypted S3 + DynamoDB for state locking).
* Limit IAM policies (avoid full AdminAccess in real deployments).

---

## 🧹 Cleanup

To destroy all resources:

```bash
terraform destroy -auto-approve
```

---

## 📌 Next Steps

* Add custom VPC module (instead of default VPC)
* Configure **IRSA** (IAM Roles for Service Accounts) for pod-level permissions
* Add monitoring (CloudWatch, Prometheus, Grafana)
* Deploy workloads with Helm or GitOps

---

## 📄 License

This project is licensed under the **MIT License**.

---

