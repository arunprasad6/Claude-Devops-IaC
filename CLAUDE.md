# CLAUDE.md — AWS Infrastructure Specification

This file defines the AWS infrastructure components to be provisioned within an **existing VPC**. Claude (or any AI coding assistant) should use these definitions as the authoritative source of truth when generating Terraform, CDK, CloudFormation, or any IaC code for this project.

---

## General Constraints

- **All resources must be deployed into the existing VPC.** Never create a new VPC.
- Retrieve the VPC ID and subnet IDs via data sources / lookups — do **not** hardcode them.
- All resources must be tagged with at minimum:
  ```
  Environment = var.environment   # e.g. dev | staging | prod
  Project     = var.project_name
  ManagedBy   = "terraform"       # or "cdk" / "cloudformation"
  ```
- Prefer private subnets for all compute and database resources.
- Public subnets are only permitted for load balancers and NAT Gateways.

---

## 1. S3 Bucket

| Parameter | Value |
|---|---|
| **Purpose** | Application artifact storage and data lake |
| **Bucket name** | `${var.project_name}-${var.environment}-data` |
| **Region** | Same as existing VPC region |
| **Versioning** | Enabled |
| **Encryption** | SSE-S3 (AES-256) by default; upgrade to SSE-KMS for prod |
| **Public access** | Block all public access — all four block settings set to `true` |
| **Lifecycle rules** | Transition objects to S3-IA after 30 days; Glacier after 90 days; expire after 365 days |
| **Access logging** | Enabled; logs written to a separate `-logs` bucket |
| **Bucket policy** | Enforce `aws:SecureTransport` (HTTPS only) |
| **VPC endpoint** | Attach a Gateway VPC Endpoint for S3 to the existing VPC route tables |

### Notes for Claude
- Do **not** set `force_destroy = true` in production environments.
- The `-logs` bucket should have its own lifecycle policy (expire after 90 days).
- Enable S3 Object Lock only if compliance mode is explicitly requested.

---

## 2. EC2 Instance

| Parameter | Value |
|---|---|
| **Purpose** | Application / bastion / worker node |
| **AMI** | Latest Amazon Linux 2023 (lookup via `aws_ami` data source) |
| **Instance type** | `t3.medium` (dev) / `m6i.large` (prod) |
| **Subnet** | Private subnet (pulled from existing VPC data source) |
| **Key pair** | Reference existing key pair via `var.key_pair_name` |
| **IAM instance profile** | Attach a least-privilege IAM role with SSM Session Manager access |
| **Security group** | Allow outbound HTTPS (443) and HTTP (80); allow inbound only from VPC CIDR on app port |
| **Root volume** | gp3, 30 GB, encrypted with KMS |
| **User data** | Bootstrap script to install CloudWatch agent and application dependencies |
| **Monitoring** | Detailed monitoring enabled; CloudWatch alarms for CPU > 80% |
| **Auto-recovery** | Enable EC2 auto-recovery via CloudWatch alarm action |

### Notes for Claude
- Never assign a public IP or Elastic IP unless explicitly instructed.
- Use `aws_ssm_parameter` to store and retrieve any secrets needed in user data — never hardcode credentials.
- If an Auto Scaling Group is needed instead of a single instance, ask for clarification before proceeding.

---

## 3. RDS Instance (Relational Database)

| Parameter | Value |
|---|---|
| **Purpose** | Primary relational database |
| **Engine** | PostgreSQL 16.x (latest minor via `var.db_engine_version`) |
| **Instance class** | `db.t3.medium` (dev) / `db.r6g.large` (prod) |
| **Storage** | 100 GB gp3, encrypted with KMS; autoscaling up to 500 GB |
| **Multi-AZ** | `false` (dev) / `true` (prod) |
| **Subnet group** | DB subnet group built from **private** subnets of the existing VPC |
| **Security group** | Allow inbound 5432 only from EC2 and EKS node security groups |
| **Database name** | `var.db_name` |
| **Master username** | `var.db_username` (stored in Secrets Manager) |
| **Master password** | Auto-generated; stored in AWS Secrets Manager |
| **Backup retention** | 7 days (dev) / 30 days (prod) |
| **Backup window** | `03:00-04:00` UTC |
| **Maintenance window** | `sun:04:00-sun:05:00` UTC |
| **Parameter group** | Custom parameter group with `log_connections = 1` and `log_min_duration_statement = 1000` |
| **Deletion protection** | `true` in prod; `false` in dev |
| **Performance Insights** | Enabled (7-day retention in dev; 731-day / paid tier in prod) |

### Notes for Claude
- Never output or log the master password.
- Provision a **read replica** only when `var.create_read_replica = true`.
- The RDS instance must **not** be publicly accessible (`publicly_accessible = false`).
- Use `aws_db_instance` (single) or `aws_rds_cluster` (Aurora) based on `var.use_aurora` flag.

---

## 4. EKS Cluster

| Parameter | Value |
|---|---|
| **Purpose** | Managed Kubernetes cluster for containerised workloads |
| **Cluster name** | `${var.project_name}-${var.environment}-eks` |
| **Kubernetes version** | `1.30` (update `var.eks_k8s_version` to override) |
| **Endpoint access** | Private endpoint enabled; public endpoint disabled in prod |
| **VPC** | Existing VPC — subnets must span **at least 2 AZs** |
| **Subnet placement** | Control plane in private subnets; nodes in private subnets |
| **Cluster security group** | Allow inbound 443 from VPC CIDR; egress unrestricted |
| **IAM** | Dedicated cluster role (`AmazonEKSClusterPolicy`) and node role (`AmazonEKSWorkerNodePolicy`, `AmazonEC2ContainerRegistryReadOnly`, `AmazonEKS_CNI_Policy`) |
| **Logging** | Enable all control plane log types: `api`, `audit`, `authenticator`, `controllerManager`, `scheduler` |
| **Add-ons** | `vpc-cni` (latest), `coredns` (latest), `kube-proxy` (latest), `aws-ebs-csi-driver` (latest) |

### Managed Node Group

| Parameter | Value |
|---|---|
| **Node group name** | `${var.project_name}-${var.environment}-ng` |
| **AMI type** | `AL2_x86_64` (or `AL2_ARM_64` for Graviton) |
| **Instance types** | `["t3.medium"]` (dev) / `["m6i.large", "m6a.large"]` (prod) |
| **Capacity type** | `ON_DEMAND` (prod) / `SPOT` (dev) |
| **Scaling** | min=2, desired=2, max=5 — enable Cluster Autoscaler |
| **Disk size** | 50 GB per node |
| **Update config** | `max_unavailable = 1` |
| **Labels** | `role=worker`, `env=${var.environment}` |

### Notes for Claude
- Enable **IRSA** (IAM Roles for Service Accounts) — set `enable_irsa = true` on the cluster.
- Install the **AWS Load Balancer Controller** after cluster creation to manage ALB/NLB Ingress.
- Install **Cluster Autoscaler** with the appropriate IAM policy via IRSA.
- Do **not** use `kubernetes_provider` resources in the same Terraform root module as the cluster — use a separate module or `null_resource` with `local-exec` for post-cluster bootstrapping.
- Tag private subnets with `kubernetes.io/role/internal-elb = 1` and public subnets with `kubernetes.io/role/elb = 1`.

---

## Variable Reference

```hcl
# Required — must be set in tfvars or CI/CD environment
variable "project_name"       {}   # e.g. "myapp"
variable "environment"        {}   # dev | staging | prod
variable "aws_region"         {}   # e.g. "us-east-1"
variable "vpc_id"             {}   # existing VPC ID
variable "private_subnet_ids" {}   # list of private subnet IDs
variable "public_subnet_ids"  {}   # list of public subnet IDs (LB only)
variable "key_pair_name"      {}   # existing EC2 key pair name
variable "db_name"            {}
variable "db_username"        {}

# Optional with defaults
variable "db_engine_version"  { default = "16" }
variable "use_aurora"         { default = false }
variable "create_read_replica"{ default = false }
variable "eks_k8s_version"    { default = "1.30" }
```

---

## Security Checklist

Before applying infrastructure, Claude must verify:

- [ ] No resource has a public IP unless explicitly required
- [ ] All storage (EBS, RDS, S3) is encrypted at rest
- [ ] All secrets are in AWS Secrets Manager or Parameter Store — never in code
- [ ] Security groups follow least-privilege (no `0.0.0.0/0` inbound)
- [ ] IAM roles follow least-privilege (no `*` actions in prod)
- [ ] Deletion protection is enabled on RDS in prod
- [ ] S3 public access block is fully enabled
- [ ] CloudTrail is enabled (assumed pre-existing in the account)
