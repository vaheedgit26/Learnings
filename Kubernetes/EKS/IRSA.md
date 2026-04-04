###  IRSA in EKS
---
Q: What is IRSA?  
A: IRSA stands for IAM Roles for Service Account, using IRSA we bind an IAM Role to kubernetes Service Account via OIDC allowing it to assume the Role ("sts:AssumeRoleWithWebIdentity") without static credentials.
