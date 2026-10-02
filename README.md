# Bookstore — AWS Three-Tier Microservices Platform

Friends, welcome! This repo is a **real, working bookstore website** that runs on AWS. It is also a great project to learn from, because it uses almost every DevOps tool you study in the course — all in one place.

No need to mug up anything here. Read slowly, one section at a time, and try to understand the logic. Simple only.

**What is inside this project?**
- A **React website** (the part users see).
- **Five small Node.js services** (the part that does the work).
- A **MySQL database** on Amazon RDS.
- **Terraform** to create all the AWS infrastructure from code.
- **Docker** to pack each service into a container.
- **Kubernetes (EKS)** to run those containers.
- **ArgoCD** to deploy automatically from Git (GitOps).
- **GitHub Actions** to test, scan, and build on every push.
- **Prometheus + Grafana + Loki** to watch everything (observability).

**Easy memory trick:** Code → Docker image → Git → ArgoCD → Kubernetes → Users. Everything flows in that order.

---

## Documentation

| Doc | What you will learn |
|---|---|
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | How the whole system fits together, and why it was built this way |
| [`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md) | Step-by-step guide to build everything from zero |
| [`docs/UML.md`](docs/UML.md) | Diagrams of the application: services, database, login, checkout |
| [`docs/LEARNING_PATH.md`](docs/LEARNING_PATH.md) | A 9-step plan to learn this project one small piece at a time |
| [`terraform-explained.md`](terraform-explained.md) | Reading guide for the `terraform/` folder, file by file |
| [`k8s-explained.md`](k8s-explained.md) | Reading guide for the `k8s/` folder, file by file |
| [`CONTRIBUTING.md`](CONTRIBUTING.md) / [`SECURITY.md`](SECURITY.md) | How to help, and how to report a security problem |

**Tip:** New here? Start with `docs/LEARNING_PATH.md`. Or read this README first, then `ARCHITECTURE.md`, then `DEPLOYMENT.md`.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Tech Stack](#2-tech-stack)
3. [Repository Structure](#3-repository-structure)
4. [Prerequisites](#4-prerequisites)
5. [Local Development](#5-local-development)
6. [Building and Pushing Docker Images](#6-building-and-pushing-docker-images)
7. [Infrastructure, Deploy, and CI/CD](#7-infrastructure-deploy-and-cicd)
8. [Secret Management](#8-secret-management)
9. [Security Controls](#9-security-controls)
10. [GitHub Secrets Reference](#10-github-secrets-reference)
11. [Common Beginner Mistakes](#11-common-beginner-mistakes)
12. [License](#12-license)

---

## 1. Architecture Overview

Look at this picture first. It is the whole project in one view.

```
                                   Internet
                                      |
                         Route 53 (public zone, failover routing)
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

**Think of it like a shopping mall:**
- **Route 53** = the street address of the mall.
- **ALB** = the main entrance gate.
- **Frontend** = the shop window.
- **api-gateway** = the security guard + receptionist. Checks your ID, sends you to the right shop.
- **Five services** = the shops inside.
- **RDS** = the warehouse at the back.

**What runs where?**
- Everything runs in **one EKS cluster** called `bookstore-eks` in `us-west-1`.
- The `bookstore` namespace holds the frontend.
- Five more namespaces hold the services: `catalog`, `user`, `order`, `notification`, `gateway`.
- ArgoCD deploys all of them from this Git repo.
- Monitoring is **not** inside the cluster. It runs on its own EC2 machine (the reason is in `ARCHITECTURE.md`).

**Traffic flow, step by step:**
1. The user opens `bookstore.<domain>`. Route 53 → ALB → frontend (React app served by nginx).
2. The React app calls `api.bookstore.<domain>`. Same ALB → `api-gateway`.
3. `api-gateway` checks the login token (JWT) and forwards the request to the right service.
4. Each service reads and writes its **own database** inside the shared RDS instance.
5. Each service shows its health numbers on `/metrics`. Prometheus collects them, Grafana draws them.

**Real-time example:** You click "Add to Cart".
- Browser → `api-gateway` (checks your token) → `order-service` → saves a row in `order_db`.
- You see "added" on screen. Total time: a few milliseconds.

---

## 2. Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 18, Nginx (Alpine) |
| Microservices | Node.js, Express, mysql2, prom-client, helmet, morgan |
| Database | MySQL 8.0 (RDS, Multi-AZ) |
| Container Registry | Amazon ECR |
| Orchestration | Kubernetes 1.31 on Amazon EKS |
| Ingress | AWS Load Balancer Controller (ALB) |
| Progressive Delivery | Argo Rollouts (installed, not used right now) |
| Infrastructure as Code | Terraform ≥ 1.7, AWS provider ~5.0, Helm provider |
| CI/CD | GitHub Actions |
| GitOps | ArgoCD |
| Secret Management | AWS Secrets Manager + External Secrets Operator |
| Observability | Prometheus + Grafana + Loki + Alertmanager on one EC2 (Docker Compose) |
| Security Scanning | Trivy (containers + Terraform), Gitleaks (secrets), SonarCloud (code quality) |
| TLS | ACM wildcard certificate on the ALB |
| Testing | Vitest per service, with a fake (mock) database |
| Disaster Recovery | Backups and images copied to `us-west-2`, Route 53 failover |

**Easy memory trick:** Build = Docker. Run = Kubernetes. Create = Terraform. Deploy = ArgoCD. Check = GitHub Actions. Watch = Grafana.

---

## 3. Repository Structure

```
.
├── terraform/                  # All infrastructure as code
│   ├── main.tf                  # Root config — calls the modules
│   ├── argocd.tf / cloudfront.tf / dr.tf
│   ├── iam.tf / observability-rbac.tf / outputs.tf / providers.tf / variables.tf / versions.tf
│   ├── modules/                  # Reusable building blocks
│   │   ├── ecr/                    # Image storage, one repo per service
│   │   ├── eks/                     # Kubernetes cluster + worker nodes
│   │   ├── eks-addons/               # Helm installs: ALB controller, ESO, ArgoCD, Argo Rollouts
│   │   ├── monitoring-ec2/            # Prometheus/Grafana/Loki/Alertmanager machine
│   │   ├── network/                 # VPC, subnets, NAT gateways
│   │   ├── rds/                     # MySQL database
│   │   ├── route53/                 # DNS zones and records
│   │   └── security/                # Security groups
│   └── environments/             # Example tfvars for dev and staging
│
├── client/                    # React frontend
│   ├── Dockerfile              # Two stages: build → Nginx
│   └── src/
│
├── services/                  # Microservices (Node/Express, Vitest tests)
│   ├── api-gateway/            # Login check + request routing
│   ├── catalog-service/        # Books
│   ├── user-service/           # Register, login, tokens
│   ├── order-service/          # Cart and orders
│   └── notification-service/   # Order notifications
│   # each: Dockerfile, app.js, index.js, __tests__/
│
├── k8s/                        # Kubernetes manifests (Kustomize base + overlays)
│   ├── base/                    # Frontend resources
│   ├── services/                # One folder per microservice
│   ├── overlays/dev/ , overlays/prod/
│   └── argocd/                  # ArgoCD Application manifests
│
├── scripts/                     # Python 3 helper scripts (standard library only)
├── docs/                        # ARCHITECTURE.md, DEPLOYMENT.md, UML.md
└── .github/workflows/           # ci-cd.yml, terraform.yml, terraform-drift.yml
```

**How to read this quickly:**
- Want to change **infrastructure**? Go to `terraform/`.
- Want to change **app code**? Go to `services/` or `client/`.
- Want to change **how it runs in Kubernetes**? Go to `k8s/`.
- Want to change **the pipeline**? Go to `.github/workflows/`.

**Easy memory trick:** terraform = build the land. k8s = arrange the shops. services = the shopkeepers.

---

## 4. Prerequisites

Install these tools on your laptop before you start.

| Tool | Minimum Version | Why we need it |
|---|---|---|
| Node.js | 18 | Run services and frontend locally |
| Docker | 24 | Build images |
| Terraform | 1.10.0 | Create AWS infrastructure (uses native S3 state locking) |
| AWS CLI | 2.x | ECR login, EKS access, helper scripts |
| kubectl | 1.31 | Talk to the cluster |
| Python | 3.8 | All scripts in `scripts/` and every Terraform helper |
| helm | 3.x | Optional, only to look at add-ons |
| kustomize | 5.x | Build manifests locally |

**Good news:** you do not need to check the versions by hand. Run this one command:

```bash
python3 scripts/preflight.py
```

- It checks that `terraform`, `kubectl`, `aws`, and `python3` are installed and new enough.
- It checks that your AWS login works.
- It checks that `config.env` is filled in.
- It **only reports**. It never installs anything.
- `make plan` and `make apply` run it for you automatically.

**Real-time example:** Without preflight, a missing tool fails deep inside a 30-minute `terraform apply`. With preflight, you find out in 5 seconds. Always cheaper!

---

## 5. Local Development

### Run one microservice (example: `order-service`)

```bash
cd services/order-service
npm install

# Create a .env file with your local MySQL details
cat > .env <<EOF
DB_HOST=localhost
DB_USERNAME=root
DB_PASSWORD=yourpassword
DB_PORT=3306
DB_NAME=order_db
APP_PORT=3000
EOF

npm run dev     # auto-reloads when you change code
# or
npm start
```

- Every service also has a `/metrics` page for Prometheus.
- The `.env` file is **never committed** to Git.

### Run the tests (no database needed!)

```bash
cd services/<service-name>
npm test
```

- Tests use **Vitest** and a fake database (`vi.fn()`).
- So you do not need MySQL to test. Nice and easy.

### Run the frontend

```bash
cd client
npm install
npm start          # development server
# or
npm run build      # production build → build/
```

- The frontend calls `http://localhost:3000` by default.
- To point it somewhere else, set `REACT_APP_API_URL` before you start or build.

**Easy memory trick:** `npm start` = run it. `npm test` = check it. `npm run build` = pack it.

---

## 6. Building and Pushing Docker Images

```bash
# Usage — account ID, region and API URL are read from config.env
python3 scripts/build_and_push.py <IMAGE_TAG>

# Example
python3 scripts/build_and_push.py v1.2.0

# Override anything for a one-time run
python3 scripts/build_and_push.py v1.2.0 --account-id 123456789012 --region us-west-1 \
  --api-url https://api.bookstore.your-domain.com
```

- This script builds each image and pushes it to Amazon ECR.
- **You rarely need it.** The CI/CD pipeline does the same thing on every merge to `main`.
- Use the script only for a hotfix or a quick pre-release test.

---

## 7. Infrastructure, Deploy, and CI/CD

**How the project gets built:**
- **Terraform** creates 8 modules plus a few root files (IAM, CloudFront, DR).
- **ArgoCD** deploys the frontend and five services from Git.
- Full step-by-step is in [`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md).
- The "why" behind each module is in [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md).

**The CI/CD pipeline** is in `.github/workflows/ci-cd.yml`. On every push it does this, in order:

1. **Secret scan** — Gitleaks looks for passwords and keys in your code.
2. **Test and validate** — unit tests, `npm audit`, SonarCloud, and a Kubernetes manifest check (kubeconform).
3. **Build and scan** — build each Docker image, scan it with Trivy, push to ECR.
4. **Deploy (only `main`)** — a person approves → CI updates the image tags in Git → ArgoCD syncs the cluster.

**Real-time example:** You push a small fix to `order-service`.
- Gitleaks: no secrets found. ✅
- Tests pass. ✅
- Image built and pushed. ✅
- Your lead clicks "Approve". ✅
- CI commits `chore: bump image tags to ed119808`.
- ArgoCD sees the new commit and updates the pods within about 3 minutes.

**Important:** CI **never runs `kubectl`**. It only edits image tags in Git. ArgoCD does the actual deploying. That is GitOps!

![Bookstore CI/CD Pipeline](CICD_Diagram1.png)

<details>
<summary>Click to see the detailed stage-by-stage diagram</summary>

![Bookstore DevSecOps CI/CD Pipeline detail](CICD_Diagram2.png)

</details>

**Two more workflows:**
- `terraform.yml` — on changes under `terraform/`: scan, format check, validate, plan. On a push to `main` it also **applies** the plan.
- `terraform-drift.yml` — runs every day at 06:00 UTC. It checks if someone changed AWS by hand and fails if the real world no longer matches the code.

**Easy memory trick:** Push → Test → Build → Approve → Bump tag → ArgoCD deploys.

---

## 8. Secret Management

A **secret** is anything private: a password, a key, a token.

| Where | How it is handled |
|---|---|
| Production (EKS) | External Secrets Operator reads AWS Secrets Manager and creates normal Kubernetes Secrets in each namespace |
| CI/CD pipeline | GitHub Secrets only. `AWS_ROLE_ARN`, `AWS_ACCOUNT_ID`, `API_URL`, and the SonarCloud values. No database passwords at all |
| Local development | A `.env` file, never committed (see `.gitignore`) |
| Terraform | Creates secrets in AWS Secrets Manager. Examples: `/bookstore/db-credentials` and `/bookstore/grafana-admin` |

**The golden rule:** never write a password, key, or account ID in a file you commit to Git.

**Real-time example:** A new developer hardcodes a database password and pushes. Gitleaks catches it in Stage 1 and the pipeline stops. Painful, but much better than leaking it!

**Easy memory trick:** Secrets live in AWS. Code only holds the *name* of the secret, never its value.

---

## 9. Security Controls

| Control | How it is done here |
|---|---|
| Secret detection | Gitleaks scans every commit and the full Git history |
| Code quality | SonarCloud Quality Gate (bugs, code smells, security hotspots, coverage) |
| Unit tests | Vitest per service, runs before the audit step |
| Dependency CVEs | `npm audit --omit=dev --audit-level=high` per service. The frontend fails only on critical |
| Container CVEs | Trivy scans every image. Results go to the GitHub Security tab. Today it **reports** findings and does **not** stop the build (see note below) |
| Terraform security | Trivy config scan on every Terraform change. This one **does** fail on CRITICAL/HIGH |
| No static AWS keys | GitHub OIDC → temporary IAM role |
| Secrets in the cluster | External Secrets Operator + AWS Secrets Manager |
| Non-root containers | All pods run as a normal user, not root |
| Read-only filesystem | `readOnlyRootFilesystem: true` on app containers |
| Network rules | Default-deny NetworkPolicy in every namespace, then only the needed paths are opened |
| HTTPS | ACM certificate on the ALB, HTTP is redirected to HTTPS |
| Manual deploy gate | The GitHub `production` environment needs a reviewer to approve |

> **Note for learners:** the comments in `ci-cd.yml` say Trivy is a "hard fail", but each Trivy step is set to `exit-code: "0"`. So findings are shown, not enforced. If you want real blocking, change it to `"1"`. A good first improvement to try!

**Easy memory trick:** Scan the code, scan the image, scan the Terraform, hide the secrets, lock the network.

---

## 10. GitHub Secrets Reference

Set these in GitHub: **Settings → Secrets and variables → Actions**. Do this before you run the pipeline.

| Secret | What it is | Example |
|---|---|---|
| `AWS_ACCOUNT_ID` | Your 12-digit AWS account ID | `123456789012` |
| `AWS_ROLE_ARN` | The IAM role CI assumes through OIDC | `arn:aws:iam::123456789012:role/bookstore-github-oidc-role` |
| `API_URL` | Public URL of the api-gateway (baked into the React build) | `https://api.bookstore.your-domain.com` |
| `SONAR_TOKEN` | SonarCloud token (sonarcloud.io → My Account → Security) | `token...` |
| `SONAR_ORGANIZATION` | SonarCloud organization key | `your-org-key` |
| `SONAR_PROJECT_KEY` | SonarCloud project key | `your-project-key` |

**Optional variable:** `AWS_REGION` (in the *Variables* tab, not Secrets). If you skip it, CI uses `us-west-1`.

---

## 11. Common Beginner Mistakes

- **Forgetting `config.env`.** Copy `config.env.example`, fill it in, then run `python3 scripts/configure.py`.
- **Skipping the state bucket step.** Without `init_backend.py`, Terraform uses local state and plans to create everything again.
- **Wrong region in CI.** If you deploy outside `us-west-1`, set the `AWS_REGION` variable in GitHub too.
- **Pods in `ImagePullBackOff` on day one.** No images exist in ECR yet. Push any commit to `main` and let CI build them.
- **Committing `.env` or `terraform.tfvars`.** Both are gitignored. Keep it that way.
- **Running `terraform apply` "just to see".** It costs real money. Always read the plan first.

---

## 12. License

[MIT](LICENSE)

---

**Final words:** Do not try to learn everything in one day. Run one service locally. Then read one Terraform module. Then watch one pipeline run. Step by step, it all becomes clear. You can do it! 💪
