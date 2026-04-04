###  IRSA in EKS
---
Q: What is `IRSA`?  
`IRSA  = IAM Roles for Service Account`
>  Using IRSA we bind an IAM Role to kubernetes Service Account via OIDC allowing it to assume the IAM Role ("sts:AssumeRoleWithWebIdentity") without static credentials.

Q: What is `OIDC`?  
`OIDC = Open ID Connect`
> 👉 OIDC (OpenID Connect) is the identity mechanism that allows your Kubernetes workloads (pods) to securely authenticate with AWS and assume IAM roles — without using static credentials.  
> 👉 OIDC in EKS is the trust bridge that lets Kubernetes pods securely assume AWS IAM roles using identity tokens instead of passwords.

OIDC is an identity layer built on OAuth 2.0 that:

* Issues signed tokens (JWTs)
* Verifies who a workload is
* Lets AWS trust that identity

## ⚙️ How OIDC Works in EKS
When you create an EKS cluster:    
- EKS automatically provides an OIDC issuer URL  
- This issuer is linked to your cluster  
- Kubernetes service accounts get JWT tokens   
- AWS verifies these tokens via OIDC  

## 🔁 Flow with IRSA (Real-world)  
Using IAM Roles for Service Accounts:  
* Pod uses a Kubernetes Service Account
* Service account is linked to an IAM Role
* Pod gets a JWT token from OIDC provider
* Calls AWS STS: **`AssumeRoleWithWebIdentity`**
* AWS validates token via OIDC
* Temporary credentials are returned

## 📊 Architecture View  
```text
Pod → Service Account → OIDC Token → AWS STS → IAM Role → Access AWS
```

## 🧠 Key Components  
1. OIDC Provider
* Created in AWS IAM
* Trusts your EKS cluster
* Example:
```text
https://oidc.eks.region.amazonaws.com/id/XXXXXXXX
```

2. JWT Token
* Issued by Kubernetes
* Contains:
    * service account name
    * namespace
    * cluster identity

3. STS (Security Token Service)
* AWS service that validates token
* Returns temporary credentials

## 🔥 Why OIDC is Important in EKS
**❌ Before (Bad Practice)**
* Store AWS keys inside pods
**✅ With OIDC + IRSA**
* No static credentials
* Fine-grained IAM roles per workload
* More secure and scalable
