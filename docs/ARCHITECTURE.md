# Architecture

Friends, in this document we will understand how the whole Bookstore system is built — what each piece does, and **why** it was built that way. Everything here describes the real code on `main`, not a dream design.

Take your time. Read one section, then look at the matching folder in the repo. That is the best way to learn.

---

## 1. What is this project?

A bookstore website, built to learn a **production-style three-tier architecture** on AWS.

**The three tiers (layers):**
- **Presentation tier** = the React website users see.
- **Application tier** = the Node.js services and the api-gateway.
- **Data tier** = the MySQL database on RDS.

**Then we cut the app tier into microservices:**
- The old single "backend monolith" is **gone**. It was deleted.
- Now the app tier is five small services.
- The frontend only ever talks to **api-gateway**.

**Easy memory trick:** Monolith = one big shop. Microservices = a mall with many small shops.

**Real-time example:** If the order shop (`order-service`) crashes, people can still browse books from `catalog-service`. In a monolith, one crash would take down everything.

---

## 2. Big picture diagram

```
                                   Internet
                                      |
                         Route53 (public zone, failover routing)
                                      |
                    ┌─────────────────────────────────┐
                    │   CloudFront (optional, off by   │
                    │      default) or direct to ALB   │
                    └─────────────────────────────────┘
                                      |
                        ALB (AWS Load Balancer Controller)
                          ┌───────────┴───────────┐
                 host: bookstore.<domain>   host: api.bookstore.<domain>
                          |                           |
                  frontend Service              api-gateway Service
              (React static, nginx)      (Node/Express, JWT verification)
                                                       |
                          ┌────────────┬──────────────┼──────────────┐
                          |            |              |              |
                  catalog-service  user-service  order-service  notification-service
                          |            |              |              |
                          └────────────┴──────┬───────┴──────────────┘
                                               |
                                        RDS MySQL 8.0
                                        (Multi-AZ, us-west-1, per-service schemas)
```

**Key points:**
- Everything lives in **one EKS cluster**: `bookstore-eks`, region `us-west-1`.
- The `bookstore` namespace has the frontend only.
- Five namespaces hold the services: `catalog`, `user`, `order`, `notification`, `gateway`.
- ArgoCD deploys them from `k8s/overlays/prod` and `k8s/services/*/overlays/prod`.
- **One ALB** serves both hostnames (`bookstore.<domain>` and `api.bookstore.<domain>`).

---

## 3. Two regions: primary and backup

| Region | Role | What lives there |
|---|---|---|
| `us-west-1` | **Primary** | EKS, RDS, monitoring EC2, all real traffic |
| `us-west-2` | **Backup (DR)** | Copies of ECR images, copies of RDS backups, a Route 53 failover record |

**Important to understand:**
- There is **no EKS cluster** in `us-west-2`.
- If `us-west-1` goes fully down, there is no compute there to take over yet.
- So today's DR is **backup-level**, not active-active.
- Backup copying is **off by default**. You turn it on with `enable_dr_replication`.
- The RDS backup copy also needs its own KMS key.
- See `terraform/dr.tf`.

**Easy memory trick:** Backup region = a spare copy of your notes. It saves your data, but it cannot teach the class for you.

---

## 4. Terraform modules

Terraform builds the whole AWS side. The code is split into **modules**, like Lego blocks.

| Module | What it creates | Needs first |
|---|---|---|
| `network` | VPC `170.20.0.0/16`, 2 public + 6 private subnets, internet gateway, NAT gateway per AZ, a free S3 gateway endpoint | — |
| `security` | Security groups: ALB (80/443 from internet), RDS (3306 from the VPC) | `network` |
| `rds` | MySQL 8.0 `db.t3.micro`, Multi-AZ, gp3 storage, admin password in Secrets Manager, optional backup copy | `network`, `security` |
| `route53` | Private zone (for RDS name) + public records with failover | `network`, `rds`, `eks` (needs the ALB name) |
| `ecr` | Image repos: `frontend` plus the five services. Keeps the last 10 images. Optional copy to another region | — |
| `eks` | Kubernetes 1.31 cluster, `t3.medium` nodes (min 1 / max 3 / desired 3), OIDC provider for IRSA, secrets encrypted with KMS | `network`, `security` |
| `eks-addons` | External Secrets Operator, AWS Load Balancer Controller, ArgoCD, Argo Rollouts, plus VPC CNI, EBS CSI, metrics-server | `eks` |
| `monitoring-ec2` | One `t3.small` EC2 running Prometheus, Grafana, Loki, Alertmanager, kube-state-metrics (Docker Compose) | `network`, `eks`, and the Grafana secret |

**Root-level files** (things that do not belong to a module):
- `iam.tf` — the GitHub OIDC role for CI.
- `ingress-cert.tf` — the wildcard ACM certificate (there is **no** `acm` module).
- `cloudfront.tf` — optional CDN.
- `dr.tf` — cross-region backup settings.
- `argocd.tf` — applies the ArgoCD project and apps, and finds the ALB hostname.
- `main.tf` — also creates the per-service DB passwords and the JWT secret.

### The real dependency order

The order you *write* the modules in `main.tf` is **not** the order Terraform *runs* them. Terraform runs anything in parallel unless one needs the other's output.

```
network ──┬─→ security ──┬─→ rds ──→ route53 ──→ ingress-cert.tf
          │              └─→ eks ──┬─→ eks-addons ─────┐
          │                        └─→ monitoring-ec2 ←┘ (needs eks + the
ecr  (independent)                                        Grafana secret only)
iam.tf (independent)
```

- `ecr` and `iam.tf` do not need the network, so they run right away.
- `rds` and `eks` both need only `network` + `security`. So they build **at the same time**.
- A full build takes about `max(RDS time, EKS time)`, not the sum.
- `monitoring-ec2` used to wait for all Helm charts. That wait was removed because it only needs one fast secret.

**Easy memory trick:** Terraform is smart. If two things do not need each other, it builds them together.

### Things we removed on purpose
- **No CloudWatch, CloudTrail, or GuardDuty** (removed 2026-08-23). Prometheus, Grafana, Loki, and Alertmanager already cover metrics, logs, and alerts. GuardDuty had no alerting connected, and nobody read CloudTrail.
- VPC Flow Logs, EKS control-plane logs, and RDS Enhanced Monitoring were dropped too. They only sent data to CloudWatch.

### The destroy helper
- `terraform destroy` can get stuck because EKS makes **hidden network resources** (extra ENIs and a security group) that Terraform does not know about.
- So a helper `null_resource.cleanup_eks_networking` runs a Python script on destroy.
- It deletes those leftovers **after** the cluster is gone and **before** the VPC is deleted.
- Without it, the VPC delete fails with `DependencyViolation`.

---

## 5. Network layout (subnets)

```
VPC 170.20.0.0/16 (us-west-1)

public[0]   170.20.1.0/24   us-west-1a   — IGW, ALB
public[1]   170.20.2.0/24   us-west-1c   — IGW, ALB
private[0]  170.20.3.0/24   us-west-1a   — EKS nodes
private[1]  170.20.4.0/24   us-west-1c   — EKS nodes
private[2]  170.20.5.0/24   us-west-1a   — EKS nodes
private[3]  170.20.6.0/24   us-west-1c   — EKS nodes
private[4]  170.20.7.0/24   us-west-1a   — RDS
private[5]  170.20.8.0/24   us-west-1c   — RDS
```

- **Public subnets** hold the load balancer and the NAT gateways.
- **Private subnets** hold the worker nodes and the database. No direct internet access.
- **NAT gateway per AZ** by default. If one AZ fails, only that AZ loses outbound internet.
- Want to save money? Set `single_nat_gateway = true`. One shared NAT, but it becomes a single point of failure.

**Real-time example:** Your EKS node needs to pull a package from the internet. It goes out through the NAT gateway. But nobody on the internet can start a connection *in* to the node. That is the NAT rule.

**Easy memory trick:** Public subnet = shop front. Private subnet = back room.

---

## 6. Why monitoring runs on EC2, not in the cluster

**The first idea:** put `kube-prometheus-stack` inside EKS.

**What went wrong:**
- One `t3.medium` node did not have enough memory.
- It starved the other pods.
- The Helm install never became `Ready` in time.

**The fix:**
- Move Prometheus, Grafana, Loki, and Alertmanager to **one separate EC2** with Docker Compose.
- The cluster runs **zero monitoring pods**.
- `node-exporter` and Fluent Bit run as **systemd services** on each node (set up in the node launch template), not as DaemonSets.
- `kube-state-metrics` runs as a Docker container on the monitoring EC2. It reads the cluster over the network using a read-only EKS access entry.

**How Prometheus finds the nodes:**
- A cron script asks the EC2 API for the cluster's nodes every 5 minutes.
- It writes the IPs to `file_sd` target files.
- So when a node is replaced, Prometheus follows automatically.
- Another cron job refreshes the kube access token every 10 minutes.

**Why no `ServiceMonitor` files?**
- They used to be in `k8s/base/monitoring/`. They were removed on 2026-08-29.
- Nothing in the cluster installs the Prometheus Operator that reads them.
- Worse, an unknown resource type makes the **whole ArgoCD sync fail**.
- So please do not add them back.

**Easy memory trick:** Small cluster, big appetite? Give monitoring its own house.

---

## 7. The database

- One **RDS MySQL 8.0** instance, `db.t3.micro`, **Multi-AZ**.
- It lives in two private subnets set aside only for RDS (subnets 4 and 5).
- The admin login is stored in Secrets Manager at `/bookstore/db-credentials`.
- Each service gets **its own database and its own user**:

| Service | Database | DB user |
|---|---|---|
| catalog | `catalog_db` | `catalog_user` |
| user | `user_db` | `user_service_user` |
| order | `order_db` | `order_service_user` |
| notification | `notification_db` | `notification_service_user` |

- Each user can only do `SELECT, INSERT, UPDATE, DELETE` on its own database. Nothing else.
- Tables are created by a small **schema-init Job** that ArgoCD runs *before* each deploy (a PreSync hook). Not by the app code.
- This is **schema-level isolation**, not one RDS per service. Separate RDS instances is a future step.

**Real-time example:** If `notification_service_user` is ever hacked, it still cannot read the `user_db` passwords. Different door, different key.

**History note:** An old in-cluster MySQL design was deleted on 2026-08-14. RDS is the only real database.

---

## 8. Secrets flow

```
AWS Secrets Manager (/bookstore/*)
        |
        | IRSA (IAM Role for Service Account)
        v
External Secrets Operator (external-secrets-sa, namespace external-secrets)
        |
        | ClusterSecretStore "aws-secretsmanager" (cluster-wide, shared)
        v
ExternalSecret (one per namespace, e.g. order-db-secret)
        |
        v
Kubernetes Secret → pod environment variables
```

**In plain words:**
1. Passwords are stored in AWS Secrets Manager.
2. The External Secrets Operator (ESO) is allowed to read `/bookstore/*` through an IAM role.
3. In each namespace, an `ExternalSecret` says "please copy this secret here".
4. ESO creates a normal Kubernetes Secret.
5. The pod reads it as an environment variable.

**The bug that was fixed:**
- ESO's ServiceAccount had **no IAM role**, even though the store expected one called `external-secrets-sa`.
- So no ExternalSecret could read from AWS. Everything silently failed.
- Fixed in `modules/eks-addons/external-secrets.tf`.
- After you deploy, always check the ClusterSecretStore is `Ready` (see `DEPLOYMENT.md`, Step 7).

**One shared role, on purpose:**
- Every service uses the same ClusterSecretStore and the same IAM role, limited to `/bookstore/*`.
- Fully separate secrets per service would need one SecretStore per namespace. That is more work than this project needs right now.

**Easy memory trick:** AWS holds the key. ESO fetches the key. The pod just uses it.

---

## 9. GitOps and deployment flow

```
git push → GitHub Actions CI
  1. secret-scan (Gitleaks)
  2. test (Vitest + coverage, npm audit, SonarCloud)   ┐
     validate (kubeconform on k8s manifests)            ├─ run in parallel
  3. build-and-push (Docker build → Trivy scan → ECR push)
  4. deploy (main branch only, manual approval gate):
       kustomize edit set image ... → commit → push
                    |
                    v
            ArgoCD polls the repo every 3 minutes
                    |
                    v
        kustomize build → apply to the cluster
        (auto-prune, self-heal)
```

**What these words mean:**
- **Auto-sync** — ArgoCD applies changes by itself.
- **Prune** — if you delete a file in Git, ArgoCD deletes it in the cluster too.
- **Self-heal** — if someone changes the cluster by hand, ArgoCD changes it back to match Git.

**Key rule:** CI **never** runs `kubectl`. It only edits image tags in Git. ArgoCD does the real apply.

**Real-time example:** Someone runs `kubectl scale deployment frontend --replicas=10` by hand. A minute later ArgoCD sets it back to what Git says. Git is the boss!

**Easy memory trick:** Git = the truth. ArgoCD = the worker who keeps the cluster the same as the truth.

---

## 10. The microservices platform

All five services are live. **Every** call from the frontend goes through `api-gateway`.

```
frontend (React static files, served via bookstore-ingress)
    |  every API call
api-gateway (Node/Express + http-proxy-middleware, JWT check) — only public door
    ├── /books          → catalog-service        (GET public; POST needs login; PUT/DELETE need admin)
    ├── /auth, /users   → user-service           (register, login, profile)
    ├── /orders, /cart  → order-service          (cart, checkout, order history)
    └── (internal only) → notification-service   (called by order-service, never by the browser)
```

**What each service has:**
- Its own ECR image repo.
- Its own Kubernetes namespace.
- Its own Deployment, Service, HPA (auto-scaler), and PDB (keeps pods alive during upgrades).
- Its own database inside the shared RDS.
- Its own `/metrics` page with a `service="<name>"` label.

**One ApplicationSet deploys all five** (`k8s/argocd/applicationset-microservices.yaml`). It uses a list generator, so adding a service is just adding one list item.

**Pod safety settings (on all six app deployments):**
- **Topology spread** over zones, then nodes, with `ScheduleAnyway`. It is a *soft hint*. It spreads pods when possible but never blocks a pod from starting.
- Runs as non-root, read-only filesystem, all Linux capabilities dropped, no service-account token mounted.

**Network rules (NetworkPolicy):**
- Every namespace starts with **default-deny**.
- `api-gateway` accepts traffic from the VPC range (the ALB uses target-type `ip`, so NetworkPolicy can only match by CIDR). It can talk out to the `catalog`, `user`, and `order` namespaces and DNS.
- `catalog`, `user`, and `order` accept traffic **only from the `gateway` namespace**.
- `order-service` may also call the `notification` namespace.

**Ingress:**
- `k8s/base/ingress/ingress.yaml` sends `bookstore.<domain>` → frontend.
- `k8s/services/api-gateway/base/ingress.yaml` is the **only** owner of `api.bookstore.<domain>`.
- Both share one ALB through the same `group.name: bookstore`.
- The gateway ingress sets `healthcheck-path: /health`. Without it, the ALB checks `/`, gets 404, and marks every pod unhealthy.

**Not done yet (on purpose):** service mesh / mTLS, async messaging (SQS), one RDS per service, distributed tracing.

**No canary right now:**
- The old backend's Argo Rollout (10% → 25% → 50% → 100%) was deleted with the monolith.
- Each service now uses a plain rolling update.

**Real-time example:** When `order-service` updates, Kubernetes starts a new pod, waits until `/health` is OK, then stops the old pod. Users notice nothing.

---

## 11. Traffic flow (the real one)

**Loading the website:**
```
Internet → Route53 (bookstore.<domain>) → (CloudFront, optional) → ALB
    → frontend Service (static React via nginx)
```

**Every API call after that:**
```
Internet → Route53 (api.bookstore.<domain>) → (CloudFront, optional) → ALB
    → api-gateway (JWT check on protected routes)
    → catalog / user / order / notification service
    → RDS :3306 (own database, shared instance)
```

---

## 12. Quick recap table

| Concept | One-line meaning | Real-time example |
|---|---|---|
| Three-tier | Web layer, app layer, data layer | React → services → RDS |
| Microservices | Many small services, one job each | `order-service` only handles cart and orders |
| api-gateway | The single public door of the backend | Checks your token, then forwards |
| Terraform module | A reusable block of infrastructure | `network`, `rds`, `eks` |
| ArgoCD | Keeps the cluster equal to Git | Reverts a manual `kubectl` change |
| ESO | Copies secrets from AWS into Kubernetes | `order-db-secret` appears in `order` namespace |
| NetworkPolicy | Firewall between pods | Only `gateway` can call `catalog` |
| Monitoring EC2 | Monitoring outside the cluster | Grafana still works if the cluster is struggling |
| DR region | Backup copy in `us-west-2` | ECR images and RDS backups are copied there |

---

## Related docs

- [`DEPLOYMENT.md`](DEPLOYMENT.md) — how to build all this from zero
- [`UML.md`](UML.md) — diagrams of the app: login, checkout, data model
- [`../README.md`](../README.md) — tech stack, repo structure, local development
