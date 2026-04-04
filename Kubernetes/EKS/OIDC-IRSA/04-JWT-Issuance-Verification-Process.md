## END-TO-END FLOW: EKS → POD → IRSA → TEMP AWS CREDENTIALS
---
🧱 1. EKS CLUSTER CREATION (FOUNDATION)
**When you create an EKS cluster:**  
👉 AWS creates:
* Managed Control Plane (AWS-owned)
    * Kubernetes API Server
    * etcd
    * Controller Manager
    * Scheduler  
👉 Important:
* Control plane runs in AWS account (not yours)
* You only interact via API endpoint
