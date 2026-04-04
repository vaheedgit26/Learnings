###  IRSA in EKS
---
Q: What is `IRSA`?  
`IRSA  = IAM Roles for Service Account`
>  Using IRSA we bind an IAM Role to kubernetes Service Account via OIDC allowing it to assume the IAM Role ("sts:AssumeRoleWithWebIdentity") without static credentials.

Q: What is `OIDC`?
`OIDC = Open ID Connect`
> OIDC (OpenID Connect) is the identity mechanism that allows your Kubernetes workloads (pods) to securely authenticate with AWS and assume IAM roles — without using static credentials.  
> 👉 OIDC in EKS is the trust bridge that lets Kubernetes pods securely assume AWS IAM roles using identity tokens instead of passwords.
