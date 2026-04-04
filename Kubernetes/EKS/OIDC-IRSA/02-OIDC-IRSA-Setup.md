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

**Attach policy to role:**  
```bash
aws iam attach-role-policy \
  --role-name EKS-IRSA-Role \
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/EKS-S3-Access
```

## ✅ Step 4: Create Kubernetes Service Account  

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/EKS-IRSA-Role
```
Apply it:  
```bash
kubectl apply -f sa.yaml
```

## ✅ Step 5: Use in Pod  

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  serviceAccountName: my-service-account
  containers:
  - name: app
    image: amazonlinux
    command: ["sleep", "3600"]
```
## 🔁 What Happens Behind the Scenes
```text
Pod → ServiceAccount → OIDC JWT Token
    → AWS STS (AssumeRoleWithWebIdentity)
    → IAM Role → Temporary Credentials
```
---
## 🧠 Trust Policy Deep Explanation (Interview Gold)  
This line is critical: 
```json
"<OIDC_PROVIDER>:sub": "system:serviceaccount:default:my-service-account"
```
👉 Meaning:  
* Only this exact service account can assume the role
* Prevents privilege escalation
---
## 🔥 Real-World Example (Argo CD)
---
With Argo CD:
* Namespace: `argocd`
* ServiceAccount: `argocd-application-controller`
```json
"sub": "system:serviceaccount:argocd:argocd-application-controller"
```

## ⚠️ Common Mistakes  
❌ Forgetting OIDC provider setup  
❌ Wrong namespace in trust policy  
❌ Missing annotation on service account  
❌ Using wildcard (*) in trust policy (security risk)  

🚀 Pro Tips (From Production Experience)
* Use least privilege policies
* Use separate IAM roles per workload
* Audit via CloudTrail
* Avoid sharing roles across namespaces


