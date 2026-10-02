# Terraform Folder, Explained — File by File, Module by Module

Friends, this is a **reading guide** for the `terraform/` folder. Open the real file next to this page and follow along. Each section says what the file does, why it exists, and the one thing to remember.

**Good to know:**
- This is a personal reference note, not the main documentation. The main docs are `docs/ARCHITECTURE.md`, `docs/DEPLOYMENT.md`, and `docs/UML.md`.
- It describes `main` with one primary region. If the code changes later, this page can go out of date.
- Companion guide: [`k8s-explained.md`](k8s-explained.md).

**How the folder is organized:**
- **Root `.tf` files** = cross-cutting things (IAM, ArgoCD setup, certificates, CloudFront, DR).
- **`terraform/modules/`** = 8 reusable modules. Each one is a self-contained piece of infrastructure.

**Easy memory trick:** Root files = the glue. Modules = the Lego blocks.

---

## Part 1 — Root-level files

### `versions.tf`
- Sets the **minimum Terraform version**: `>= 1.10.0`. This is needed for native S3 state locking (no DynamoDB table).
- Lists every provider: `aws`, `helm`, `random`, `null`, `tls`, `kubernetes`, and `kubectl`.
- The `kubectl` provider is `gavinbunney/kubectl`, not HashiCorp's. It checks the YAML only at apply time. That matters because `argocd.tf` applies ArgoCD's YAML in the **same apply** that installs ArgoCD's CRDs.
- Holds the `backend "s3"` block. The `bucket` and `region` are **empty on purpose**.
  - Terraform reads the backend before it reads variables, so it cannot use `var.aws_region`.
  - `scripts/init_backend.py` writes the real values in.

### `providers.tf`
- **Three `aws` providers:**
  - the default one (region = `var.aws_region`)
  - `aws.secondary` (region = `var.secondary_region`, used for DR copies)
  - `aws.us_east_1` (hardcoded, because CloudFront certificates must live in `us-east-1`)
- The `helm`, `kubernetes`, and `kubectl` providers log in with `aws eks get-token`. No saved kubeconfig. A fresh token is made each run.
- The `helm` provider keeps its own repo cache in `.terraform/helm/`. So an old Helm repo from another project on your laptop cannot break this one.

### `variables.tf`
Most variables have a validation block. The important ones:
- `aws_region` / `secondary_region` — checked with a regex. `secondary_region` defaults to `us-west-2`.
- `enable_dr_replication` — **off by default**. It used to turn on by accident, because `secondary_region` was never empty. That created copies nobody wanted. Now it is a separate flag.
- `domain`, `github_repo`, `alert_email` — no safe default. `scripts/configure.py` fills them in.
- `primary_alb_dns` / `secondary_alb_dns` — leave empty. `argocd.tf` finds the primary one for you.
- `dr_kms_key_id` — empty by default. Cross-region RDS backup copy needs your own KMS key.
- `monitoring_admin_cidr` — default `0.0.0.0/0` (open). Narrow it for real use.
- `extra_admin_principal_arns` — extra people who get cluster-admin. Whoever runs `apply` is always added.

### `locals.tf`
- The VPC range: `170.20.0.0/16`.
- The subnet map: **8 subnets**. 2 public, 6 private.
  - Private 0–3 = EKS nodes.
  - Private 4–5 = RDS (one per AZ).
- AZ names come from `data.aws_availability_zones.available`. They used to be hardcoded as `us-west-1a/c`, which broke every other region.
- `eks_node_subnet_cidrs` = just the 4 node subnets. The RDS security group uses it, so only nodes can reach the database.

### `data.tf`
Two lookups used everywhere:
- `aws_caller_identity.current` — your account ID.
- `aws_availability_zones.available` — the AZ names for `locals.tf`.

### `main.tf` — the orchestra conductor
It wires the modules together and holds a few resources that belong to no module.

- **`module.network`, `module.security_groups`, `module.rds`** — each passes IDs from the one before it.
- **Per-service DB credentials**
  - A `for_each` over `catalog`, `user`, `order`, `notification`.
  - Makes one random password and one Secrets Manager secret per service.
  - All services share one RDS instance, but each has its own database and DB user.
  - This replaced about 110 lines of four copy-pasted blocks.
- **JWT secret**
  - One shared HS256 secret (64 characters, no special characters, because it goes straight into an env var).
  - `user-service` signs tokens with it. `api-gateway` verifies them.
- **`module.route53`** — gets `local.primary_alb_dns` (found automatically) for the failover records.
- **`module.ecr`** — 6 repos: frontend plus five services.
- **`null_resource.cleanup_eks_networking`** (destroy-time only)
  - EKS makes hidden network resources outside Terraform's view: extra ENIs and a cluster security group.
  - They can block the VPC delete with `DependencyViolation`.
  - `module.eks` has `depends_on` this resource. On destroy, Terraform goes in **reverse**, so EKS is deleted first and **then** this cleanup script runs.
  - It runs `scripts/cleanup_eks_networking.py`.
- **`module.eks`**
  - Cluster `bookstore-eks`, version 1.31, `t3.medium` nodes, min 1 / max 3 / desired 3.
  - `t3.medium` allows only **17 pods per node** (an ENI IP limit). Two nodes filled up once all services plus 2 gateway replicas arrived.
  - Admin list = whoever runs `apply` + `var.extra_admin_principal_arns`.
- **SES email setup (for alerts)**
  - An SES email identity (AWS sends a verification email to `alert_email`).
  - SES starts in sandbox mode, so sender and receiver must both be verified. Using the same address for both means one click.
  - An IAM user with permission to send mail only (`ses:SendRawEmail`).
  - A helper script `derive_ses_smtp_password.py` turns the IAM key into an SMTP password (AWS's HMAC-SHA256 recipe) and saves it to Secrets Manager. The password never touches Terraform state.
- **`aws_eip.monitoring` + `module.monitoring_ec2`**
  - **No** blanket `depends_on = [module.eks_addons]`. It only needs the cluster and one fast secret.
  - **Does** depend on `null_resource.ses_smtp_password`, so the SMTP secret is filled in before the instance boots.
- **`module.eks_addons`**
  - `depends_on = [module.eks, module.network]`.
  - The `network` link keeps the NAT gateway alive while the ALB controller cleans up on destroy.

### `argocd.tf` — two jobs
1. **Apply ArgoCD's own YAML** from `k8s/argocd/`: AppProject, Application, ApplicationSet.
   - Files are read straight from disk, so there is only one copy of the truth.
   - AppProject goes first. ArgoCD rejects an app that names a project that does not exist.
2. **Find the ALB hostname inside the same apply.**
   - `null_resource.wait_for_alb_hostname` runs `scripts/wait_for_alb_hostname.py`.
   - It waits for the `bookstore-ingress` Ingress to appear (ArgoCD creates it later), then waits for the AWS Load Balancer Controller to fill in the hostname.
   - `data.kubernetes_ingress_v1.bookstore` reads it. `local.primary_alb_dns` uses it unless you set a manual override.
   - This removed the old "apply twice" step.

### `iam.tf`
- The GitHub Actions OIDC role `bookstore-github-oidc-role`. CI uses it with no static keys.
- The trust policy uses wildcards: `repo:OWNER*/REPO*:ref:...`. Some GitHub tokens add numeric IDs (`OWNER@id/REPO@id`), and a strict match would reject them.
- Allowed branches: `main`, `improvements`, `observability`.
- The role can only **push images**: `ecr:GetAuthorizationToken` plus push actions on `bookstore-*` repos.
- It has **no** permission to change infrastructure in this file.

### `ingress-cert.tf`
- The ACM certificate the ALB actually uses (same region as the cluster).
- Covers `*.<domain>` **and** `*.bookstore.<domain>`. One wildcard covers only one label, so `api.bookstore.<domain>` needed the second one.
- No certificate ARN is written into any Kubernetes file. The ALB controller finds a matching **issued** certificate by host name. That avoids a chicken-and-egg problem: the cert does not exist until Terraform runs, but the Ingress YAML is deployed by ArgoCD on its own schedule.
- It runs the full DNS validation, because the controller only matches issued certs.

### `cloudfront.tf`
- **Optional, off by default.**
- When on: a certificate in `us-east-1`, plus a CloudFront distribution with the ALB as origin.
- `/static/*` is cached for a long time. Everything else passes through with TTL 0, because this is a dynamic app.

### `dr.tf`
- One resource: `aws_db_instance_automated_backups_replication`.
- Only runs when `dr_kms_key_id` is set. AWS-managed keys cannot be used across regions.

### `observability-rbac.tf`
- A ClusterRole and ClusterRoleBinding so the monitoring EC2's Prometheus can read:
  - kubelet `/metrics/cadvisor` (real pod CPU and memory use. `kube-state-metrics` only knows requests, limits, and status.)
  - each pod's `/metrics` through the API server's pod proxy (pod IPs are not reachable from outside the cluster).
- It binds to a **group** (`monitoring-metrics-readers`), not to a user name. The EC2 session's user name contains the instance ID, which changes if the instance is replaced.

### `outputs.tf`
Everything a script or a person might need: RDS endpoint, DB secret ARNs, ECR URLs, EKS details, the GitHub role ARN, Route 53 nameservers, monitoring URLs, the monitoring SSH key (marked sensitive), the CloudFront domain, and the JWT secret ARN.

---

## Part 2 — Modules (`terraform/modules/`)

### `network`
- **Creates:** VPC `170.20.0.0/16`, subnets, internet gateway, NAT gateway(s), route tables, and an **S3 gateway endpoint**.
- **NAT gateways:** one **per AZ** by default (survives an AZ loss). Set `single_nat_gateway = true` for one shared NAT, which is cheaper but is a single point of failure.
- **S3 endpoint:** free. It keeps ECR image layers and Terraform state reads off the NAT, so you do not pay NAT data fees.
- **No dependencies.** This is the first thing built.
- **Outputs:** `vpc_id`, `public_subnet_ids`, `private_subnet_ids`.

| Index | CIDR | Purpose |
|---|---|---|
| public[0] | `170.20.1.0/24` | ALB, NAT |
| public[1] | `170.20.2.0/24` | ALB, NAT |
| private[0–3] | `170.20.3.0/24` – `170.20.6.0/24` | EKS nodes |
| private[4–5] | `170.20.7.0/24`, `170.20.8.0/24` | RDS subnet group (Multi-AZ) |

> Note: this module's own `README.md` is auto-generated by `terraform-docs` and is out of date. It still mentions VPC Flow Logs, which were removed.

### `security`
- One security group: RDS (`bookstore-rds-sg`).
- One rule: allow **3306/tcp** only from the **4 EKS node subnets**.
- That used to be the whole VPC, so even public subnets could reach the database. That was a real bug and it is fixed.
- No outbound rule. RDS does not start connections.

### `rds`
- **Creates:** a random password, the Secrets Manager secret `/bookstore/db-credentials`, a subnet group, the DB instance, and an optional rotation setup (off by default).
- **The instance:** MySQL 8.0, `db.t3.micro`, 25 GB growing to 100 GB, **Multi-AZ**, 7-day backups, encrypted storage, in subnets private[4] and [5].
- **`db_name = "test"`** is left over from the old monolith. The real databases (`catalog_db` and so on) come from the schema-init Jobs in Kubernetes.
- **Dev settings:** `skip_final_snapshot = true` and `deletion_protection = false`. They make destroy and re-create easy. Change both before using this for something that must stay up.
- **Important bug that was fixed:**
  - `DB_HOST` uses `.address`, **not** `.endpoint`.
  - `.endpoint` is `host:port`. Apps and the `mysql` CLI need only the host name.
  - For a while, no database connection in this project actually worked because of this.
- CloudWatch logs and Enhanced Monitoring were removed.

### `route53`
- **Private zone:** `bookstore.internal` with `db.bookstore.internal` pointing at RDS. Services can use a friendly name.
- **Public zone:** only **looked up** (`data` source), never created or destroyed.
  - If Terraform created it, every destroy and apply would give new nameservers and break your registrar setup.
  - `scripts/init_domain.py` creates it once.
- **Records:**
  - `primary` — apex A/ALIAS, PRIMARY failover, with an HTTPS health check. Used when CloudFront is off.
  - `primary_cf` — apex alias to CloudFront. Used when CloudFront is on.
  - `frontend` and `api` — `bookstore.<domain>` and `api.bookstore.<domain>`. These were missing for a long time, so the site could not be reached by name.
  - `secondary` — SECONDARY failover. Only created when a second-region ALB name is given (never on `main`).
- **ALIAS hosted-zone ID:** the `aws_lb_hosted_zone_id` data source with `load_balancer_type = "application"`. Using the wrong type (Classic or NLB) broke DNS twice. Check `aws elbv2 describe-load-balancers` before changing it.

### `ecr`
- 6 repos: `bookstore-frontend` plus `bookstore-<service>` for each of the five services.
- Image tags are **IMMUTABLE**. Scan on push is on. Encryption is AES256.
- A lifecycle rule keeps the last **10** images.
- Optional replication of every `bookstore-*` repo to the secondary region.

### `eks`
- **KMS key** for Kubernetes Secrets encryption, with rotation on.
- **Cluster:** `bookstore-eks`, version 1.31, public and private API endpoint, secrets encrypted with the KMS key. No control-plane logs are sent anywhere.
- **Launch template for nodes:**
  - 20 GB encrypted gp3 root disk.
  - IMDS tokens required, hop limit 2 (so containers can reach the metadata service).
  - A boot script installs **node-exporter v1.8.2** and **Fluent Bit** as **systemd services**, not DaemonSets. The goal is zero monitoring pods in the cluster.
  - Fluent Bit finds the monitoring EC2's **private IP at boot** (with retries). A baked-in public IP does not route back inside the VPC.
- **Node group:** `t3.medium`, min 1 / max 3 / desired 3, `max_unavailable = 1`. It currently ignores launch-template version changes while a vCPU quota increase is pending.
  - **Remember:** `t3.medium` is limited by IPs per node (17 pods), not by CPU or memory. "Just add replicas" is not free.
- **IAM:**
  - Cluster role: EKS cluster policy, VPC resource controller, a small KMS policy.
  - Node role: worker node policy, CNI policy, ECR read-only, and `ec2:DescribeInstances` (for the Fluent Bit lookup).
- **Admin access entries** for everyone in `admin_principal_arns`. The creator-admin permission only works once, at cluster creation. These entries are the permanent version.
- **OIDC provider** (audience `sts.amazonaws.com`). Every IRSA role trusts it.

### `eks-addons`
All charts install **at the same time**. No fake ordering. If a real apply times out, retry before adding nodes.

- **`external-secrets.tf`**
  - Helm release for External Secrets Operator (900s timeout).
  - IRSA role trusting only `system:serviceaccount:external-secrets:external-secrets-sa`.
  - Policy: read `/bookstore/*` in Secrets Manager only.
- **`aws-load-balancer-controller.tf`**
  - Chart version `3.4.0` in `kube-system`. It replaced `ingress-nginx`, which the Kubernetes project retired on 2026-03-31.
  - Its IAM policy is the official file from the controller project. Re-compare it before bumping versions.
  - `depends_on` the policy and the VPC CNI. Two real bugs: the controller hit `UnauthorizedOperation` without the policy, and new pods got stuck if the CNI vanished first on destroy.
  - Also has `null_resource.delete_ingress_objects` (destroy only). It deletes Ingress objects so the ALB is released, then waits up to 20 minutes for the controller's security group to clear.
- **`ebs-csi.tf`** — the EBS CSI driver add-on.
- **`vpc-cni.tf`**
  - Turns on **`enableNetworkPolicy = "true"`**. Without it, every NetworkPolicy manifest is just stored and **never enforced**.
  - Also installs **metrics-server**. Without it, every HPA showed `<unknown>` and never scaled.
- **`gitops.tf`**
  - ArgoCD (1200s timeout) and Argo Rollouts (300s).
  - Single replica for everything. No HA. This is demo scale.
  - It does not create the ArgoCD apps. `argocd.tf` does.
- **`grafana-secret.tf`, `monitoring-basic-auth-secret.tf`**
  - Random passwords saved in Secrets Manager.
  - The basic-auth one protects Prometheus and Alertmanager. They have no login of their own, so without it anyone in the allowed range could even silence alerts.
- **`observability.tf`** — only a comment now. Monitoring lives on the EC2.

### `monitoring-ec2`
- **AMI:** latest Ubuntu 22.04 from Canonical.
- **Security group:**
  - SSH 22, Grafana 3000, Prometheus 9090, Alertmanager 9093 — only from `admin_cidr_blocks`.
  - Loki 3100 — from the whole VPC (Fluent Bit pushes logs here).
  - Three extra rules on the **EKS cluster's** security group: node-exporter 9100, the API server 443 (for kube-state-metrics), and kubelet 10250 (for cAdvisor). The 443 rule was missing for a long time and every connection silently timed out.
- **IAM:** `ec2:DescribeInstances`, `eks:DescribeCluster`, and `secretsmanager:GetSecretValue` on exactly 3 secrets (Grafana, monitoring basic-auth, Alertmanager SMTP).
- **EKS access:** a read-only access entry in the `monitoring-metrics-readers` group.
- **SSH key:** auto-generated. `make monitoring-key` saves it as `.monitoring-ssh-key.pem` (gitignored).
- **Instance:** `t3.small`, public subnet, Elastic IP. `user_data_replace_on_change = true`, so a script change replaces the whole box. That is fine because it is fully rebuildable. User data is **gzipped** because EC2's 16 KB limit counts the compressed size.

**What `user-data.sh.tftpl` does, in order (about 740 lines):**
1. Installs Docker from Docker's own repo, plus `awscli`, `jq`, and `apache2-utils` (for `htpasswd`).
2. Installs `kubectl` v1.31.0.
3. Gets the Grafana and monitoring passwords from Secrets Manager. Each fetch **retries up to 20 times**, because the IAM policy may not have spread yet on the first boot. User data runs only once, so a single failure used to kill the whole stack.
4. Gets the SES SMTP details the same way.
5. Writes `refresh-kube-token.sh`. It runs now and every 10 minutes. It makes a kubeconfig for kube-state-metrics and a token file for Prometheus, and restarts kube-state-metrics.
6. Writes `update-prom-targets.sh`. It runs now and every 5 minutes. It asks EC2 for the cluster's nodes and writes target files for node-exporter and kubelet.
7. Writes `prometheus.yml`:
   - 10-second scrape and evaluation intervals, so a load test trips alerts in about 2 minutes.
   - 5 jobs: `prometheus`, `kube-state-metrics`, `node-exporter`, `kubelet-cadvisor`, `app-metrics` (pods with `prometheus.io/scrape: "true"`, scraped through the API server proxy).
8. Writes the alert rules — **9 alerts**: `NodeDown`, `HighCPUUsage`, `HighMemoryUsage`, `PodCrashLooping`, `KubeStateMetricsDown`, `HighPodCPUUsage`, `HighPodMemoryUsage`, `HighRequestRate`, `HighErrorRate`.
9. Writes the Alertmanager config:
   - Email through SES.
   - `critical` alerts repeat every 1 hour, others every 6 hours.
   - Two placeholder webhooks to swap for Slack, PagerDuty, or SNS.
   - A rule that mutes `warning` when a matching `critical` is already firing.
10. Writes the Loki config: filesystem storage, single node.
11. Writes the Grafana datasources: Prometheus (default), Loki, Alertmanager.
12. Writes `docker-compose.yml` with 5 services (`prometheus`, `kube-state-metrics`, `loki`, `alertmanager`, `grafana`), all `restart: unless-stopped`, and runs `docker compose up -d`.
13. Starts `import-grafana-dashboards.sh` in the background.

**Where is the Grafana dashboard ID?**
- It is hardcoded in `import-grafana-dashboards.sh`, which is written by `user-data.sh.tftpl`.
- The line is `import_dash 1860 37 "DS_PROMETHEUS"`.
- **1860** = the public grafana.com dashboard "Node Exporter Full". **37** = the pinned revision.
- The script downloads it, points it at this Grafana's Prometheus datasource, and posts it to `/api/dashboards/import`.
- It then sets the `$job` and `$node` variables by hand, because dashboard 1860 saves them empty and every panel would stay blank.
- The other two dashboards (`k8s-cluster-overview.json`, `pod-node-resources.json`) have **no ID**. They are custom files in `terraform/modules/monitoring-ec2/dashboards/` and Grafana loads them from its provisioning folder (folder `Bookstore`).
- A third dashboard (ID 315) was tried and dropped, because its `$Node` variable uses a label that does not exist here.

> Small open gap: `scripts/monitoring_credentials.py` does not print a Loki row, even though `loki_url` is a real output.

---

## Part 3 — Scripts you run by hand

All scripts in `scripts/` are Python 3 using only the standard library. Terraform runs some of them itself. The four below are run **by you**. (They used to be Bash. They were ported to Python so Windows needs no Git Bash or WSL.)

### `scripts/preflight.py` — run this first
- Checks `terraform` (>= 1.10.0), `kubectl`, `aws`, and `python3` (>= 3.8, not the Windows Store stub).
- Checks `aws sts get-caller-identity` works and that `config.env` has its 5 keys.
- Prints a PASS / WARN / FAIL table and install hints per OS.
- **Never installs anything.** `make plan` and `make apply` run it for you.

### `scripts/init_backend.py` — Step 2, once per AWS account
- Region order: command-line argument → `AWS_REGION` in `config.env` → `aws configure get region`. If none gives a region, it stops with an error.
- Bucket name: `bookstore-terraform-state-<ACCOUNT_ID>`.
- Creates the S3 bucket (versioning on, AES256, all four public-access blocks on).
- Writes the bucket and region into `terraform/versions.tf`, keeping the file's line endings exactly.
- Runs `terraform init` (with `-reconfigure` if `.terraform/` exists).

### `scripts/init_domain.py` — Step 3, once per domain
- Looks for an existing **public** hosted zone first. It filters strictly so it never picks the private `bookstore.internal` zone.
- If none exists, creates one and prints the 4 NS values for your registrar.
- Safe to run again. It says `[skip]` if the zone is already there.

### `scripts/simulate_load.py` — a demo helper
- **CPU load:** a throwaway `polinux/stress` pod.
- **Traffic load:** a steady HTTPS loop at the ALB hostname, with the `api.bookstore.<domain>` Host header. The hostname is read from the Ingress status, not from `terraform output`, because the ALB is created by the controller, not by Terraform.
- Checks Prometheus every 15 seconds and prints the alerts that are firing.
- Cleans up after itself, even on Ctrl-C.

---

## Part 4 — Apply order (real dependency order, not file order)

1. **Stage 1 (parallel):** `network`, `ecr`, `iam.tf`.
2. **Stage 2:** `security`.
3. **Stage 3 (parallel):** `rds` and `eks`.
4. **Stage 4 (parallel):** the Helm charts (ESO, ALB controller, ArgoCD, Argo Rollouts).
5. **Stage 5:** ArgoCD's AppProject → Application and ApplicationSet. Then `wait_for_alb_hostname` waits for the real ALB.
6. **Stage 6 (parallel):** `route53` (needs the ALB name) and `monitoring-ec2` (needs only the Grafana secret).
7. **Stage 7:** the ACM certificate validation. It only needs the hosted zone and could start earlier. It is last because DNS spreading is slow.

Independent and optional: `cloudfront.tf` (off by default) and `dr.tf` (off unless `dr_kms_key_id` is set).

## Destroy order (the safety nets)

Destroy goes in reverse, plus two helpers that run **only on destroy**:
- **`null_resource.delete_ingress_objects`** (in `eks-addons`) deletes Ingress objects first, so the ALB's finalizer does not freeze the destroy.
- **`null_resource.cleanup_eks_networking`** (root `main.tf`) removes leftover ENIs and the cluster security group, after EKS is gone and before the VPC.

**Why they exist:** EKS and its controllers create real AWS resources directly through the EC2 API. Terraform cannot see them, so it cannot clean them up alone.

---

## Quick recap table

| Concept | One-line meaning | Real-time example |
|---|---|---|
| Module | A reusable block of infrastructure | `network`, `rds`, `eks` |
| `depends_on` | Force an order Terraform cannot guess | `module.eks` waits for the cleanup helper (reverse on destroy) |
| `for_each` | Make many similar things from one block | One DB secret per service |
| `data` source | Look something up, do not create it | The public Route 53 zone |
| `null_resource` + `local-exec` | Run a script as part of apply or destroy | Find the ALB name, clean EKS leftovers |
| IRSA | A pod gets an IAM role through OIDC | ESO reads `/bookstore/*` secrets |
| `.address` vs `.endpoint` | Host only vs host + port | `mysql2` needs the host only |
| Remote state | Terraform's memory stored in S3 | `init_backend.py` sets it up |

**Easy memory trick:** Terraform = what you want. Modules = how it is split up. `depends_on` = what must come first. Scripts = what Terraform cannot do alone.
