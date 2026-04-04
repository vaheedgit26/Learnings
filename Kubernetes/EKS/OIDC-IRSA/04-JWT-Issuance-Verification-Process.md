### END-TO-END FLOW: EKS → POD → IRSA → TEMP AWS CREDENTIALS
---
## 🧱 STEP 1. EKS CLUSTER CREATION (FOUNDATION)
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
---
## 🔑 Who Creates the Key Pair for ServiceAccount JWTs?  
Answer:  The Kubernetes API Server (managed by AWS in EKS) creates the key pair.  
* Private key → used to sign JWT tokens by kubernetes api server for ServiceAccounts.
  * Stored inside the EKS managed control plane.
  * You cannot access it; AWS keeps it secure.
* Public key → used to verify JWT tokens.
  * Exposed via the cluster’s OIDC endpoint, e.g.:
```text
https://oidc.eks.<region>.amazonaws.com/id/<cluster-id>/.well-known/jwks.json
```

## STEP 2. CREATE IAM ROLE FOR POD (IRSA ROLE)
Trust Policy (CRITICAL)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/oidc.eks.region.amazonaws.com/id/EXAMPLE"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.region.amazonaws.com/id/EXAMPLE:sub": "system:serviceaccount:default:my-sa"
        }
      }
    }
  ]
}
```
🔍 Important Condition Explained:
```text
sub = system:serviceaccount:<namespace>:<serviceaccount-name>
```
This binds:   
👉 ONLY that ServiceAccount can assume the role  

## STEP 3. CREATE KUBERNETES SERVICE ACCOUNT
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-sa
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/MyIRSARole
```
🔥 **What this annotation does:**
It tells Kubernetes:  
👉 “When a Pod uses this ServiceAccount, inject AWS identity logic”

## STEP 4. SERVICE ACCOUNT TOKEN (JWT CREATION)
Now we go deep.  
---
🧬 JWT STRUCTURE (ACTUAL FORMAT)   
A Kubernetes ServiceAccount token is a JWT:
```text
HEADER.PAYLOAD.SIGNATURE
```
🧾 **HEADER:**
```json
{
  "alg": "RS256",
  "kid": "EXAMPLE_KEY_ID"
}
```
* `RS256` → RSA SHA256 signing
* `kid` → key ID for public key lookup

📦 **PAYLOAD (VERY IMPORTANT):**
```json
{
  "iss": "https://oidc.eks.region.amazonaws.com/id/EXAMPLE",
  "sub": "system:serviceaccount:default:my-sa",
  "aud": ["sts.amazonaws.com"],
  "kubernetes.io": {
    "namespace": "default",
    "serviceaccount": {
      "name": "my-sa",
      "uid": "uuid"
    }
  },
  "exp": 1710000000,
  "iat": 1709990000
}
```

✍️ **SIGNATURE:**
```text
RSA-SHA256(
  base64url(header) + "." + base64url(payload),
  PRIVATE_KEY
)
```

🔐 **WHO SIGNS THE TOKEN?**
👉 Kubernetes API Server
How?
* Uses private key stored in control plane
* Public key exposed via OIDC endpoint:
```text
https://oidc.eks.region.amazonaws.com/id/EXAMPLE/.well-known/jwks.json
```

## 📦 STEP 5. TOKEN DELIVERY TO POD
This is where people get confused  
🧩 Old Way (deprecated)  
   * Token stored as secret  
   * Long-lived
     
✅ **New Way (Projected Volume)**  
Kubernetes injects token into Pod:  
```text
/var/run/secrets/eks.amazonaws.com/serviceaccount/token
```
YAML inside Pod:  
```yaml  
spec:
  serviceAccountName: my-sa
```
🔥 Important:
👉 Token is mounted via Projected Volume
👉 Token is rotated automatically
👉 Token audience = sts.amazonaws.com

## 🔄 STEP 6: POD USES JWT → CALLS AWS STS
Inside the Pod:

AWS SDK detects:
```text
AWS_ROLE_ARN
AWS_WEB_IDENTITY_TOKEN_FILE
```

Then calls:
```text
sts:AssumeRoleWithWebIdentity
```

API Request:
```text
POST https://sts.amazonaws.com
```

With:
  * RoleArn
  * WebIdentityToken (JWT)
  * RoleSessionName

## 🔍 STEP 7: AWS VALIDATION PROCESS (CRITICAL)
AWS STS performs:
1. Validate OIDC Provider
  * Matches issuer URL
  * Verifies TLS thumbprint

2. Fetch Public Key
From:
```text
jwks.json
```

3. Verify JWT Signature
```text
Verify using RSA public key
```

4. Validate Claims
 
| Claim | Checked                     |   
| ----- | --------------------------- |   
| iss   | matches OIDC provider       |   
| aud   | must be `sts.amazonaws.com` |   
| sub   | must match IAM trust policy |   

## 🎯 STEP 8: TEMPORARY CREDENTIALS ISSUED
If valid:

AWS returns:
```json
{
  "AccessKeyId": "...",
  "SecretAccessKey": "...",
  "SessionToken": "...",
  "Expiration": "..."
}
```

## 🔁 STEP 9: POD USES TEMP CREDENTIALS  
Now Pod can:  
 * Access S3  
 * Access DynamoDB  
 * Call any AWS service allowed by IAM role

```text
+-------------------+
|   Pod (App)       |
|-------------------|
| JWT Token         |
| Service Account   |
+--------+----------+
         |
         | (1) AssumeRoleWithWebIdentity
         v
+------------------------+
| AWS STS                |
+------------------------+
         |
         | (2) Validate JWT
         v
+------------------------------+
| OIDC Provider (EKS)          |
| - issuer URL                 |
| - public keys (JWKS)         |
+------------------------------+
         |
         | (3) Signature Verified
         |
         | (4) Check Conditions:
         |     sub == serviceaccount
         |
         v
+------------------------+
| IAM Role               |
+------------------------+
         |
         | (5) Issue Temp Creds
         v
+------------------------+
| Pod gets:              |
| - Access Key           |
| - Secret Key           |
| - Session Token        |
+------------------------+
```
## ⚠️ COMMON INTERVIEW TRAPS (VERY IMPORTANT)
❌ Myth: “API server sends JWT to AWS” (WRONG)   
✔ Pod sends JWT to AWS STS     

❌ Myth: “IAM trusts Kubernetes” (WRONG)     
✔ IAM trusts OIDC provider      

❌ Myth: “Token is static” (WRONG)        
✔ Token is short-lived & rotated         

## 🔥 ULTRA IMPORTANT CONCEPTS TO MEMORIZE
* JWT signed by Kubernetes API Server
* AWS verifies using OIDC public key
* Trust defined via IAM Role trust policy
* Pod uses AssumeRoleWithWebIdentity
* NO static credentials anywhere

## 🧠 MENTAL MODEL (LOCK THIS IN)
```text
Kubernetes = Identity Provider
OIDC = Bridge
AWS STS = Verifier + Credential Issuer
Pod = Client
```
