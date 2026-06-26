# Three-Tier To-Do App on AWS EKS

A production-grade, three-tier To-Do web application containerized with Docker and deployed on **Amazon EKS (Elastic Kubernetes Service)** using `eksctl`. Infrastructure is fully managed via **AWS CloudFormation** with an **AWS Load Balancer Controller** routing public traffic.

---

## Project Screenshots

### CloudFormation Stacks — All CREATE_COMPLETE
![CloudFormation Stacks Final](<img width="959" height="473" alt="Screenshot 2026-06-26 044310" src="https://github.com/user-attachments/assets/5837312f-cdc0-4597-84be-1a8867e485c2" />)
> All 3 stacks successfully provisioned: EKS cluster, managed node group, and the IAM service account for the AWS Load Balancer Controller.

---

### Live App — Accessible via AWS Load Balancer
![To-Do App Live](todo-app-live.png)
> The To-Do app running live, served through an AWS Elastic Load Balancer. Tasks can be added, checked off, and deleted in real time.

---

### Debugging — Node Group Rollback (Troubleshooting Phase)
![CloudFormation Rollback](cloudformation-rollback.png)
> During initial deployment, the managed node group hit a `ROLLBACK_COMPLETE` while the cluster itself was `CREATE_COMPLETE`. This was debugged and resolved by re-provisioning the node group with corrected parameters.

---

### EKS Cluster Dashboard
![EKS Cluster](eks-cluster.png)
> The `three-tier-cluster` running Kubernetes v1.34 in `us-east-1` (N. Virginia) — Active status, 0 node health issues, 0 cluster health issues.

---

## Architecture Overview

```
Internet
    │
    ▼
AWS Elastic Load Balancer  (AWS Load Balancer Controller)
    │
    ▼
EKS Cluster — three-tier-cluster (us-east-1)
    │
    ├── Frontend Pod(s)   →  React / HTML/CSS To-Do UI
    ├── Backend Pod(s)    →  REST API server
    └── Database Pod(s)  →  Persistent data layer
```

The cluster spans **multiple availability zones** and uses a **managed node group** for the worker nodes, all orchestrated automatically via `eksctl` and CloudFormation.

---

## AWS Infrastructure (CloudFormation Stacks)

| Stack Name | Status | Description |
|---|---|---|
| `eksctl-three-tier-cluster-cluster` | CREATE_COMPLETE | Core EKS cluster with dedicated IAM |
| `eksctl-three-tier-cluster-nodegroup-ng-*` | CREATE_COMPLETE | EKS Managed Node Group (EC2 workers) |
| `eksctl-three-tier-cluster-addon-iamserviceaccount-kube-system-aws-load-balancer-controller` | ✅ CREATE_COMPLETE | IAM role for the AWS Load Balancer Controller service account |

---

## Tech Stack

| Layer | Technology |
|---|---|
| **Container Orchestration** | Amazon EKS (Kubernetes 1.34) |
| **Infrastructure as Code** | AWS CloudFormation (via `eksctl`) |
| **Load Balancing** | AWS Load Balancer Controller |
| **Node Management** | EKS Managed Node Group |
| **Region** | us-east-1 (N. Virginia) |
| **App** | Three-tier To-Do (Frontend + Backend + DB) |

---

## Deployment Steps

### 1. Create the EKS Cluster
```bash
eksctl create cluster \
  --name three-tier-cluster \
  --region us-east-1 \
  --nodegroup-name ng \
  --node-type t3.medium \
  --nodes 2
```

### 2. Install AWS Load Balancer Controller
```bash
# Associate OIDC provider
eksctl utils associate-iam-oidc-provider \
  --cluster three-tier-cluster \
  --approve

# Create IAM service account
eksctl create iamserviceaccount \
  --cluster three-tier-cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# Install via Helm
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=three-tier-cluster \
  --set serviceAccountName=aws-load-balancer-controller
```

### 3. Deploy the Application
```bash
kubectl apply -f k8s/
```

### 4. Get the Load Balancer URL
```bash
kubectl get ingress -n <namespace>
# Access the app at the ADDRESS shown
```

---

## Troubleshooting — Node Group Rollback

During the first attempt, the managed node group stack hit `ROLLBACK_COMPLETE`. The EKS cluster itself was healthy (`CREATE_COMPLETE`), so the fix was:

1. Delete the failed node group stack from CloudFormation.
2. Correct the node group config (instance type / subnet / IAM permissions).
3. Re-run `eksctl create nodegroup` — resulting in a clean `CREATE_COMPLETE`.

---

## Notes

- The EKS cluster is on **Kubernetes 1.34** — standard support ends **December 2, 2026**. Plan an upgrade before then.
- The console warning about IAM principal access to Kubernetes objects can be resolved by creating an **access entry** in the EKS Access tab for your IAM user/role.
- The app URL is served over HTTP (not HTTPS) — adding an ACM certificate to the load balancer is recommended for production.

---

*Deployed on AWS EKS · us-east-1 · June 2026*
