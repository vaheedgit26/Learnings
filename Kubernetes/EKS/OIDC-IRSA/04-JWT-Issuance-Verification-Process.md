### END-TO-END FLOW: EKS → POD → IRSA → TEMP AWS CREDENTIALS
---
## 🧱 1. EKS CLUSTER CREATION (FOUNDATION)
**When you create an EKS cluster:**    
👉 **AWS creates:**
* Managed Control Plane (AWS-owned)
    * Kubernetes API Server
    * etcd
    * Controller Manager
    * Scheduler
       
👉 **Important:**
* Control plane runs in AWS account (not yours)
* You only interact via API endpoint

👉 **You configure:**
* VPC, Subnets
* Node Groups (EC2 or Fargate)
* IAM Role for cluster

👉 **CRITICAL: OIDC PROVIDER CREATION**
When you run:
```bash
eksctl utils associate-iam-oidc-provider --cluster my-cluster --approve
```
**(or)**
```hcl
# OIDC Provider for IRSA
data "tls_certificate" "eks" {
  url = aws_eks_cluster.main.identity[0].oidc[0].issuer
}

resource "aws_iam_openid_connect_provider" "eks" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.tls_certificate.eks.certificates[0].sha1_fingerprint]
  url             = aws_eks_cluster.main.identity[0].oidc[0].issuer

  tags = {
    Name    = "${var.project}-${var.env}-oidc-provider"
    Env     = var.env
    Project = var.project
  }
}
```
## What happens internally:
* EKS exposes an OIDC issuer URL like:
```text
https://oidc.eks.<region>.amazonaws.com/id/XXXXXXXX
```
* AWS IAM stores this as a trusted identity provider  
👉 `This is the bridge between Kubernetes & AWS IAM`

This creates in IAM:  
```text
IAM → Identity Provider → OIDC Provider
```
Contains:
* Issuer URL
* Thumbprint (TLS cert hash)
* Audience (sts.amazonaws.com)
 
🔑 Who Creates the Key Pair for ServiceAccount JWTs?  
Answer: The Kubernetes API Server (managed by AWS in EKS) creates the key pair.  
* Private key → used to sign JWT tokens for ServiceAccounts.
  * Stored inside the EKS managed control plane.
  * You cannot access it; AWS keeps it secure.
* Public key → used to verify JWT tokens.
  * Exposed via the cluster’s OIDC endpoint, e.g.:
```text
https://oidc.eks.<region>.amazonaws.com/id/<cluster-id>/.well-known/jwks.json
```
