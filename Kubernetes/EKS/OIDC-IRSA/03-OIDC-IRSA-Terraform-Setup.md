### Setting up IRSA using Terraform
---
## 🧭 Overview
We will configure:
* Amazon EKS cluster (assumed already created)
* OIDC provider
* IAM policy
* IAM role (with trust policy)
* Kubernetes Service Account
* Pod using IRSA

## 📦 Prerequisites
Make sure you have:
- Terraform ≥ 1.3
- AWS CLI configured
- kubectl configured for your cluster
- Existing EKS cluster

## 📁 Project Structure
```text
irsa-terraform/
├── main.tf
├── variables.tf
├── outputs.tf
├── provider.tf
```

## ⚙️ Step 1: Configure Providers

`provider.tf`
```text
provider "aws" {
  region = var.region
}

provider "kubernetes" {
  host                   = data.aws_eks_cluster.cluster.endpoint
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.cluster.certificate_authority[0].data)
  token                  = data.aws_eks_cluster_auth.cluster.token
}
```

## 📥 Step 2: Fetch EKS Cluster Data
`main.tf`
