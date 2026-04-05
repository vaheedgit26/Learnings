###  CREATE OIDC provider and the COMPLETE flow
---
## Step 1: Create OIDC provider 
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
}
```
`Note: Creation of OIDC provider means actually registering the external OIDC provider(GitHub/EKS)` in **`AWS`**    
## Important
It contains mainly 3 inputs
> 1. url              `(In Terraform: url)`
> 2. client-id-list   `(In Terraform: client_id_list)`
> 3. thumbprint-list  `(In Terrform: thumbprint_list)`

**Whenever you create/register an OIDC provider in AWS it does the following:**   
👉 **`AWS`** stores:   
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

## 🔍 Step 4: AWS verification (THIS is where thumbprint is used)  
AWS does:  
✅ **1. Extract issuer from JWT**
```text
iss = https://token.actions.githubusercontent.com
```

✅ **2. Connects to OIDC provider URL** 
AWS makes HTTPS call to:  
```text
https://token.actions.githubusercontent.com
```

✅ **3. TLS handshake happens**  
During this:  
* AWS receives SSL certificate chain  
* Extracts root CA certificate

✅ **4. AWS computes fingerprint**
AWS internally:   
```text
Root CA Certificate → SHA-1 → fingerprint
```

✅ **5. Compare with stored thumbprint**  
```text 
Computed fingerprint == Stored thumbprint ?
```
✔️ If match → trusted connection  
❌ If mismatch → reject  

✅ **6. Validate JWT**  
Now AWS checks:  
* `iss` matches registered provider  
* `aud` matches `sts.amazonaws.com`  
* Signature using JWKs (public keys)

✅ **7. Issue credentials**  
If all valid:  
👉 AWS STS returns temporary credentials  

## 🧠 Key Insight (VERY IMPORTANT)  
> The thumbprint is used to verify the TLS connection to the OIDC provider,
NOT the JWT itself.

## 🔥 Corrected version of your statement  
Here is the perfect version 👇    
> “When we create an OIDC provider in AWS, AWS stores the issuer URL, audience, and the thumbprint of  
> the provider’s root CA certificate. When GitHub sends a JWT to AWS STS, AWS connects to the OIDC issuer URL,  
> verifies the TLS certificate using the stored thumbprint,  
> and then validates the JWT claims like issuer and audience before issuing temporary credentials.”  


