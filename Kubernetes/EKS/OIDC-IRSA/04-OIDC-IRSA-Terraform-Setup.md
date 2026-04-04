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
```hcl
data "aws_eks_cluster" "cluster" {
  name = var.cluster_name
}

data "aws_eks_cluster_auth" "cluster" {
  name = var.cluster_name
}
```
## 🔐 Step 3: Create OIDC Provider
```hcl
data "tls_certificate" "oidc" {
  url = data.aws_eks_cluster.cluster.identity[0].oidc[0].issuer
}

resource "aws_iam_openid_connect_provider" "oidc_provider" {
  url = data.aws_eks_cluster.cluster.identity[0].oidc[0].issuer

  client_id_list = ["sts.amazonaws.com"]

  thumbprint_list = [data.tls_certificate.oidc.certificates[0].sha1_fingerprint]
}
```
👉 This step is mandatory for IRSA.

## 📜 Step 4: Create IAM Policy
```hcl
resource "aws_iam_policy" "irsa_policy" {
  name = "irsa-s3-policy"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["s3:ListBucket"]
        Resource = "*"
      }
    ]
  })
}
```

## 🔥 Step 5: Create IAM Role with Trust Policy
This is the core of IRSA.
```hcl
resource "aws_iam_role" "irsa_role" {
  name = "eks-irsa-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.oidc_provider.arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "${replace(data.aws_eks_cluster.cluster.identity[0].oidc[0].issuer, "https://", "")}:sub" = "system:serviceaccount:default:my-service-account"
          }
        }
      }
    ]
  })
}
```

## 🔗 Step 6: Attach Policy to Role
```hcl
resource "aws_iam_role_policy_attachment" "attach" {
  role       = aws_iam_role.irsa_role.name
  policy_arn = aws_iam_policy.irsa_policy.arn
}
```

## ☸️ Step 7: Create Kubernetes Service Account
```hcl
resource "kubernetes_service_account" "irsa_sa" {
  metadata {
    name      = "my-service-account"
    namespace = "default"

    annotations = {
      "eks.amazonaws.com/role-arn" = aws_iam_role.irsa_role.arn
    }
  }
}
```

## 🚀 Step 8: Deploy a Test Pod
```hcl
resource "kubernetes_pod" "test_pod" {
  metadata {
    name      = "irsa-test"
    namespace = "default"
  }

  spec {
    service_account_name = kubernetes_service_account.irsa_sa.metadata[0].name

    container {
      name  = "app"
      image = "amazonlinux"

      command = ["sleep", "3600"]
    }
  }
}
```

## 📥 Step 9: Variables
```hcl
variable "region" {
  default = "ap-south-1"
}

variable "cluster_name" {
  description = "EKS cluster name"
}
```

## ▶️ Step 10: Run Terraform
```bash
terraform init
terraform plan
terraform apply
```

## 🧪 Step 11: Verify IRSA
Exec into pod:
```bash
kubectl exec -it irsa-test -- bash
```
Install AWS CLI (inside pod if needed), then:
```bash
aws sts get-caller-identity
```
👉 You should see the IAM role ARN, not node role.

## 🔁 Full Flow Recap
```text
Pod
 ↓
ServiceAccount (annotated)
 ↓
OIDC Token
 ↓
AWS STS (AssumeRoleWithWebIdentity)
 ↓
IAM Role
 ↓
Temporary Credentials
```

## ⚠️ Common Issues (Real Debugging)
❌ Access Denied
- Check trust policy `sub` exactly matches:
  ```text
  system:serviceaccount:<namespace>:<serviceaccount>
  ```

❌ No credentials
- Check annotation exists:
  ```text
  eks.amazonaws.com/role-arn
  ```

## 🧠 Production Tips
* One IAM role per workload
* Never use * in trust policy
* Use namespaces for isolation
* Audit with CloudTrail

## ⚡ Final Summary
👉 Terraform + IRSA in Amazon EKS enables secure, fine-grained AWS access from pods without static credentials, using OIDC and STS.
