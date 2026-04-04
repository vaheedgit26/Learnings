### Step-by-Step IRSA Setup (EKS)   
---
We’ll set up IRSA (IAM Roles for Service Accounts) using OIDC in Amazon EKS.   

## ✅ Step 1: Check / Associate OIDC Provider  

First, verify if your cluster already has an OIDC provider:  
```bash
aws eks describe-cluster \
  --name <cluster-name> \
  --query "cluster.identity.oidc.issuer" \
  --output text
```
If not associated, create it using eksctl:   
```bash
eksctl utils associate-iam-oidc-provider \
  --cluster <cluster-name> \
  --approve
```
👉 This creates a trust between AWS IAM and your cluster’s OIDC issuer.   

## ✅ Step 2: Create IAM Policy  

Example: allow access to S3  
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "*"
    }
  ]
}
```
Create it:  
```bash
aws iam create-policy \
  --policy-name EKS-S3-Access \
  --policy-document file://policy.json
```

## ✅ Step 3: Create IAM Role (Trust Policy 🔥)  

This is the most important part.  
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/<OIDC_PROVIDER>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "<OIDC_PROVIDER>:sub": "system:serviceaccount:<namespace>:<service-account>"
        }
      }
    }
  ]
}
```
👉 Replace:
* `<namespace>` → e.g. `argocd`
* `<service-account>` → e.g. `argocd-application-controller`
