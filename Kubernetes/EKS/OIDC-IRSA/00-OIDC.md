###  CREATE OIDC provider and the COMPLETE flow
---
## Create OIDC provider 
**Manually (Example for Github)**     
```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```
**`or` with terraform (Example for EKS cluster)**
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
`Note: Creation of OIDC provider means actually registering the external OIDC provider(GitHub/EKS)` in **`AWS`**    
## Important
It contains mainly 3 inputs
> 1. url              `(In Terraform: url)`
> 2. client-id-list   `(In Terraform: client_id_list)`
> 3. thumbprint-list  `(In Terrform: thumbprint_list)`

**Whenever you create an OIDC provider in AWS it does the following:**   
👉 Amazon Web Services stores:   
* Issuer URL (`https://token.actions.githubusercontent.com`)  
* Allowed audience (`sts.amazonaws.com`)  
* Thumbprint (of CA certificate)

## 🔑 Step 2: GitHub issues JWT  
GitHub generates a token like:  
```json
{
  "iss": "https://token.actions.githubusercontent.com",
  "aud": "sts.amazonaws.com",
  "sub": "repo:org/repo:ref:refs/heads/main"
}
```

## 🚀 Step 3: GitHub calls AWS STS  
It calls:  
```text
AssumeRoleWithWebIdentity
```
and sends: **`JWT token`**


