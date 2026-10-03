# Deployment

Friends, in this document we will build the whole Bookstore project **from zero**, on a fresh AWS account, step by step. Follow the steps in order. Do not skip any. Simple only.

**Before anything else — one golden habit:**
- Always check AWS directly to see what exists. Do not trust your old `kubectl` setup.
- A stale kubeconfig can say a cluster is there when it was deleted weeks ago.
- Quick check: `aws eks describe-cluster --name bookstore-eks`. If it says `ResourceNotFoundException`, nothing is running.

> All Terraform files live inside the `terraform/` folder. Run raw `terraform` commands from there (`cd terraform` first). Or use `make plan` / `make apply` / `make destroy` from the repo root. The Makefile does `terraform -chdir=terraform` for you.

---

## Contents

- [Before you start](#before-you-start)
- [Step 1 — Config file](#step-1--fill-in-your-config-and-generate-terraformtfvars)
- [Step 2 — Terraform state bucket](#step-2--bootstrap-terraform-state-once-per-aws-account)
- [Step 3 — Domain](#step-3--bootstrap-the-domain-once-per-domain-ever)
- [Step 4 — One apply](#step-4--one-apply-everything)
- [Step 5 — GitHub secrets](#step-5--configure-github-secrets-and-variables)
- [Step 6 — Import old entries](#step-6--import-known-conflicting-entries-only-if-re-deploying)
- [Step 7 — Check secrets sync](#step-7--check-that-secret-syncing-works)
- [Step 8 — Watch apps come up](#step-8--watch-all-apps-come-up)
- [Ongoing deploys](#ongoing-deploys)
- [Monitoring access](#monitoring-access)
- [Tearing it down](#tearing-it-down)
- [Troubleshooting cheat sheet](#troubleshooting-cheat-sheet)

---

## Before you start

### What you need

- An **AWS account** and working credentials. Test with `aws sts get-caller-identity`.
- Permissions to create VPCs, EKS clusters, RDS, IAM roles, and more.
- These four tools on your `PATH`: `terraform` (1.10.0 or newer), `kubectl`, `aws` CLI, and `python3`.
- A **domain name** you control. ACM needs it to validate the certificate.

**Why Python?**
- Every helper step in Terraform (`local-exec`) runs a real Python script from `scripts/`.
- It runs the same on Windows, macOS, and Linux. No special bash needed.
- You do **not** need the `helm` CLI. Terraform's Helm provider talks to Helm directly.

### Run the preflight check

```bash
python3 scripts/preflight.py
```

- Checks that all four tools exist and are new enough.
- Checks your AWS login works.
- Checks `config.env` has its 5 required keys.
- **Only reports.** It never installs anything.
- `make plan` and `make apply` run it for you.

### One-time GitHub OIDC provider

This is **not** created by Terraform. You do it once per AWS account.

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

- Check first with `aws iam list-open-id-connect-providers`.
- If it already exists, running it again fails with `EntityAlreadyExists`. Harmless, but skip it.

### Time and money

> ⚠️ Expect **20 to 30 minutes**, and **real money** as soon as RDS, EKS, and the monitoring EC2 exist. Never run the full `terraform apply` "just to see what happens".

- RDS takes 10–15 minutes. EKS takes 15–20 minutes. They build **at the same time**.
- The Helm add-ons also install at the same time now.
- If a Helm release times out, read the error to see which chart failed, then run `apply` again. Helm releases here are safe to retry.

---

## Step 1 — Fill in your config and generate `terraform.tfvars`

```bash
cp config.env.example config.env
# edit config.env: AWS_ACCOUNT_ID, AWS_REGION, DOMAIN, GITHUB_REPO, ALERT_EMAIL
# (GITHUB_BRANCH and SECONDARY_REGION are optional — they default to "main" and "us-west-2")
python3 scripts/configure.py
```

**Do this first**, before Step 2. The state-bucket script reads your region from `config.env`.

**Any AWS region works,** not just `us-west-1`. Zones, ECR address, and the secret store region all come from `AWS_REGION`.
- If you use a different region, also add a GitHub **Variable** called `AWS_REGION` (Settings → Secrets and variables → Actions → **Variables** tab).
- If you forget, CI falls back to `us-west-1` and builds in the wrong region.

**What `configure.py` does:**
1. Writes `terraform.tfvars` with the four values that have no safe default: `aws_region`, `domain`, `github_repo`, `alert_email`.
2. Replaces placeholder text (`YOUR_DOMAIN_HERE.com`, `ACCOUNT_ID`, and so on) in five checked-in files:
   - `k8s/base/ingress/ingress.yaml`
   - `k8s/services/api-gateway/base/configmap.yaml` (the CORS `FRONTEND_URL`)
   - `k8s/argocd/application.yaml`
   - `k8s/overlays/prod/kustomization.yaml`
   - `k8s/base/secrets/external-secret.yaml` (the secret store region)

**Do not hand-write `terraform.tfvars`.** Always generate it with this script.

Leave `primary_alb_dns` and `secondary_alb_dns` out of `config.env`:
- `primary_alb_dns` is found automatically during the apply.
- `secondary_alb_dns` stays empty until a second-region cluster exists (it does not yet).

### Commit and push those five files

ArgoCD deploys from **Git**, not from your laptop. If you skip this, the first sync deploys the placeholder text instead of your real domain.

```bash
git add k8s/base/ingress/ingress.yaml k8s/services/api-gateway/base/configmap.yaml \
        k8s/argocd/application.yaml k8s/overlays/prod/kustomization.yaml \
        k8s/base/secrets/external-secret.yaml
git commit -m "chore: configure for <your-domain>"
git push
```

`config.env` and `terraform.tfvars` are both gitignored. **Never commit them.**

**Easy memory trick:** config.env → configure.py → five stamped files → push them.

---

## Step 2 — Bootstrap Terraform state (once per AWS account)

```bash
python3 scripts/init_backend.py
```

- Creates the S3 bucket that stores the Terraform state.
- Writes the real bucket name and region into `versions.tf`.
- Runs `terraform init`.
- Locking is native S3 locking (`use_lockfile = true`). No DynamoDB table needed.

> ⚠️ If you skip this, Terraform quietly uses **local state**. Then `terraform plan` thinks nothing exists and wants to create everything again. If a plan ever shows a suspiciously large "to add" count, check `terraform state list` and confirm the backend is configured.

**Which region does it use?** In this order:
1. A region you pass on the command line (`python3 scripts/init_backend.py us-west-2`).
2. `AWS_REGION` from `config.env`.
3. `us-west-1` as the last fallback.

**Why a script?** The backend block in `versions.tf` cannot use variables. Terraform reads it before it reads variables. So the script writes the literal value in.

---

## Step 3 — Bootstrap the domain (once per domain, ever)

```bash
python3 scripts/init_domain.py
```

- Creates the public Route 53 hosted zone for your `DOMAIN` if it does not exist.
- Prints **four NS (nameserver) values**.
- Go to your registrar (GoDaddy, Namecheap, and so on) and set those four NS values.

**Do this now,** before Step 4:
- Terraform only **looks up** this zone. It never creates or destroys it.
- So you set the registrar once for the lifetime of the domain.
- DNS needs a few minutes to spread. Doing it early lets that time overlap with the build.
- If `aws_acm_certificate_validation.ingress` hangs in Step 4, the NS records have not spread yet. Just wait.

**Real-time example:** Same idea as the Route 53 class — the domain is bought on GoDaddy, and you delegate it to AWS with NS records.

---

## Step 4 — One apply, everything

```bash
cd terraform
terraform plan -out=tfplan
# read the plan! On a fresh account expect about 145 resources:
#   VPC, subnets, NAT, IGW, S3 endpoint, security groups, ACM certificates,
#   RDS, private Route 53 zone, ECR repos, EKS cluster + nodes + OIDC,
#   eks-addons (ESO, ALB controller, ArgoCD, Argo Rollouts),
#   monitoring EC2 + EIP, GitHub OIDC role,
#   ArgoCD AppProject + Application + ApplicationSet
terraform apply tfplan
```

**Why one apply and not two?**
- Before, Terraform could not create the DNS record until the load balancer existed.
- So you had to apply, copy the ALB name by hand, paste it, and apply again.
- Now `argocd.tf` finds the ALB name **inside the same apply**.
- A helper script first waits for the `bookstore-ingress` Ingress to appear (ArgoCD creates it, on its own schedule). Then it waits for the AWS Load Balancer Controller to fill in the ALB hostname.
- Then Terraform reads that hostname and creates the Route 53 records.

**`argocd.tf` also applies three manifests for you:**
- `k8s/argocd/appproject.yaml`
- `k8s/argocd/application.yaml`
- `k8s/argocd/applicationset-microservices.yaml`

So **no manual `kubectl apply -f k8s/argocd/...`** is needed. They wait for `eks-addons` (they need ArgoCD's CRDs). The Application and ApplicationSet also wait for the AppProject, because ArgoCD rejects an app that names a project that does not exist.

**If it seems stuck:** the certificate step waits for DNS. Make sure Step 3's NS values are set at your registrar.

**Easy memory trick:** Step 4 = plan, read, apply. Never apply blind.

---

## Step 5 — Configure GitHub secrets and variables

Nothing in `.github/workflows/` works until these exist. Without them, every push fails at the AWS login step.

**Why now?** The `AWS_ROLE_ARN` role is created by Step 4 using *your* local AWS login. CI cannot create its own role, because it needs that role to log in at all. After this, CI uses OIDC with no static keys.

In GitHub: **Settings → Secrets and variables → Actions**.

**Secrets tab:**

| Secret | Value |
|---|---|
| `AWS_ACCOUNT_ID` | `aws sts get-caller-identity --query Account --output text` |
| `AWS_ROLE_ARN` | `terraform output -raw github_oidc_role_arn` (only exists after Step 4) |
| `API_URL` | `https://api.bookstore.<your-domain>` |
| `SONAR_TOKEN` | sonarcloud.io → My Account → Security(access token) |
| `SONAR_ORGANIZATION` | your SonarCloud org key(devopstrininghub) |
| `SONAR_PROJECT_KEY` | your SonarCloud project key |

**Variables tab** (not secret — skip it if you use `us-west-1`):

| Variable | Value |
|---|---|
| `AWS_REGION` | Must match `AWS_REGION` in `config.env` |

> Until all secrets exist, expect **exactly two failures** on each push. Both are normal on a fresh repo:
> - `Terraform CI/CD` fails at "Configure AWS credentials (OIDC)" — no `AWS_ROLE_ARN` yet.
> - `DevSecOps Pipeline` fails at the SonarCloud step — no `SONAR_TOKEN` yet.
>
> It is not a code problem. Just missing secrets.

---

## Step 6 — Import known-conflicting entries (only if re-deploying)

**Do you need this?** Only if the account is **not** truly fresh. For example, you changed the state bucket between sessions.

**What goes wrong:** Terraform's current state does not know about things AWS already has. The apply then fails with `AlreadyExistsException`. The usual ones:
- Three Secrets Manager secrets: `/bookstore/db-credentials`, `/bookstore/grafana-admin`, `/bookstore/jwt-secret`.
- The SES email identity for your `ALERT_EMAIL`.

```bash
make import
```

- Safe on a fresh account. Each line ends with `|| echo already in state`, so nothing breaks.

**Easy memory trick:** Fresh account = skip. Re-deploy with lost state = `make import`.

---

## Step 7 — Check that secret syncing works

This bug once broke **every** secret sync in the cluster without any loud error. So please do not skip this check. (The story is in [`ARCHITECTURE.md`](ARCHITECTURE.md), section 8.)

```bash
kubectl get serviceaccount external-secrets-sa -n external-secrets -o jsonpath='{.metadata.annotations}'
# should contain: "eks.amazonaws.com/role-arn":"arn:aws:iam::<account>:role/bookstore-external-secrets"

kubectl get clustersecretstore aws-secretsmanager -o jsonpath='{.status.conditions[0]}'
# should show "status":"True","type":"Ready"
# If this is not Ready, every ExternalSecret will fail,
# because they all use this one store.

kubectl get externalsecret admin-db-secret -n catalog
# STATUS should be SecretSynced, not an error.
# Every service namespace has its own admin-db-secret; catalog is just the first to check.
```

**What each check means:**
- ServiceAccount has the role annotation → ESO is allowed to talk to AWS.
- ClusterSecretStore is `Ready` → the connection to Secrets Manager works.
- ExternalSecret is `SecretSynced` → the password really arrived in the namespace.

---

## Step 8 — Watch all apps come up

Terraform already applied `application.yaml` (the frontend and shared resources) and the ApplicationSet (the five services). **Nothing to `kubectl apply` here.** Just watch ArgoCD work. It reconciles within about 3 minutes.

```bash
kubectl get applications -n argocd
kubectl get applicationsets -n argocd
kubectl get pods -n bookstore
kubectl get pods -n catalog
kubectl get pods -n user
kubectl get pods -n order
kubectl get pods -n notification
kubectl get pods -n gateway
```

**What happens during each sync:**
- For catalog, user, order, and notification, ArgoCD first runs a **schema-init Job** (a PreSync hook). It creates the database, the tables, and that service's own DB user.
- Each Job reads the admin login from an `admin-db-secret` ExternalSecret. That secret copies `/bookstore/db-credentials` into the service's namespace.
- `api-gateway` has **no** schema-init Job, because it has no database.

**Watch a hook run (it only exists briefly):**

```bash
kubectl get jobs -n catalog
kubectl logs job/catalog-schema-init -n catalog
```

### Test the gateway

```bash
curl -s https://api.bookstore.<domain>/health
```

Right after apply, DNS might not be ready. Bypass it with port-forward:

```bash
kubectl port-forward -n gateway svc/gateway-service 8082:80
curl -s http://localhost:8082/health
```

### Pods stuck in `ImagePullBackOff`? Relax, it is expected

On a **fresh account**, no images exist in ECR yet. So all six app pods (frontend + 5 services) wait in `ImagePullBackOff`. Nothing is broken.

**Fix 1 (easiest):** push any commit to `main`. CI builds and pushes the images.

**Fix 2 (by hand):**

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGISTRY="${ACCOUNT_ID}.dkr.ecr.us-west-1.amazonaws.com"
aws ecr get-login-password --region us-west-1 | docker login --username AWS --password-stdin "$REGISTRY"

# Frontend
docker build -t "$REGISTRY/bookstore-frontend:manual" client/
docker push "$REGISTRY/bookstore-frontend:manual"
(cd k8s/overlays/prod && kustomize edit set image bookstore-frontend="$REGISTRY/bookstore-frontend:manual")

# Each service follows the same pattern — just swap the name:
docker build -t "$REGISTRY/bookstore-catalog-service:manual" services/catalog-service/
docker push "$REGISTRY/bookstore-catalog-service:manual"
(cd k8s/services/catalog-service/overlays/prod && kustomize edit set image bookstore-catalog-service="$REGISTRY/bookstore-catalog-service:manual")
# ...repeat for user-service, order-service, notification-service, api-gateway

git add k8s/overlays/prod/kustomization.yaml k8s/services/*/overlays/prod/kustomization.yaml
git commit -m "chore: manual image push for first deploy" && git push
```

### Test one service directly (skip the gateway)

This helps you find out: is the problem in the service, or in the gateway/ingress path?

```bash
kubectl port-forward -n catalog svc/catalog-service 8081:80
# in another terminal:
curl -s http://localhost:8081/health
curl -s http://localhost:8081/books
curl -s http://localhost:8081/metrics | grep 'service="catalog-service"'
```

### Test the real website (UI)

Open the site and click through it:

1. Open `https://bookstore.<domain>`. You should see the book list. No login needed.
2. Register a new account, then log in.
3. Click "Add to Cart", open Cart, change the quantity, go to Checkout, place the order.
4. Open Orders. The new order should show status `pending`.
5. Log out. Then visit `/cart`, `/checkout`, and `/orders` directly. Each should send you to `/login`.

**No browser? Use curl instead:**

```bash
curl -s https://api.bookstore.<domain>/auth/register -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"testpass123"}'
TOKEN=$(curl -s https://api.bookstore.<domain>/auth/login -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"testpass123"}' | jq -r .token)
curl -s https://api.bookstore.<domain>/cart -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" -d '{"book_id":1,"quantity":1}'
curl -s -X POST https://api.bookstore.<domain>/orders/checkout -H "Authorization: Bearer $TOKEN"
curl -s https://api.bookstore.<domain>/orders -H "Authorization: Bearer $TOKEN"
```

**Fun fact:** the **first account you register becomes the admin**. So register your own account first, before sharing the site with anyone!

---

## Ongoing deploys

After the first stand-up, you almost never run `kubectl apply` for app changes.

1. Push your change to `main`.
2. CI tests it, builds the image, scans it, and pushes it.
3. A reviewer approves the `deploy` job.
4. CI updates the image tags in Git.
5. ArgoCD deploys within about 3 minutes.

See the GitOps section in [`ARCHITECTURE.md`](ARCHITECTURE.md).

---

## Monitoring access

```bash
cd terraform
terraform output grafana_url        # Grafana, default user "admin"
terraform output prometheus_url     # Prometheus, user "admin"
terraform output alertmanager_url   # Alertmanager, user "admin"
aws secretsmanager get-secret-value --secret-id /bookstore/grafana-admin --query SecretString --output text
aws secretsmanager get-secret-value --secret-id /bookstore/monitoring-basic-auth --query SecretString --output text
```

> **On Windows,** run `python3 scripts/monitoring_credentials.py` instead. Git Bash changes any argument starting with `/` (like `/bookstore/grafana-admin`) into a Windows path. Then `aws.exe` fails with a confusing "Invalid name" error. The script prints every URL and password in one table and avoids the problem.

**Handy Makefile commands:**
- `make monitoring-status` — shows Docker Compose status on the monitoring box.
- `make monitoring-logs` — follows the setup logs.
- Both fetch an SSH key from Terraform state automatically (saved as `.monitoring-ssh-key.pem`, gitignored).

**Want to see an alert fire?** Run `python3 scripts/simulate_load.py`. It creates CPU and traffic load. The demo alerts (`HighPodCPUUsage`, `HighRequestRate`) fire in about 2 minutes, and the email arrives soon after. Use `--duration 240` for a comfortable margin. It cleans up after itself.

---

## Tearing it down

```bash
cd terraform
terraform destroy
```

`make destroy` does the same with `-auto-approve`. Use the plain command if you want a confirmation prompt.

**What is built in to make destroy safe** (because this stack is created and deleted often):
- The ALB and Ingress are released **before** the VPC is torn down.
- A helper script removes hidden EKS network leftovers.
- Secrets Manager entries use `recovery_window_in_days = 0`, so they delete right away and the names are free again.
- ArgoCD apps carry a finalizer, so ArgoCD deletes everything it manages **before** the app itself goes away. Terraform handles the order for you.

**Good news about DNS:**
- The public Route 53 zone is only **looked up** by Terraform (a `data` source). It is never created or destroyed by it.
- So `terraform destroy` leaves the zone alone. Your **four NS values stay the same**.
- You do **not** need to touch your registrar again after a destroy and re-apply. That is why Step 3 is "once per domain, ever".
- Only the alias records Terraform created (pointing at the ALB) are removed.
- You can print the nameservers any time with `terraform output route53_public_name_servers`.

**Easy memory trick:** Terraform borrows the zone. It never owns it, so it never throws it away.

---

## Troubleshooting cheat sheet

| Symptom | Likely reason | What to do |
|---|---|---|
| `terraform plan` wants to create everything | State backend not configured | Run `init_backend.py` (Step 2) |
| ACM validation hangs | Registrar NS not set or not spread yet | Re-check Step 3 and wait |
| CI fails at AWS login | No `AWS_ROLE_ARN` secret | Step 5 |
| CI fails at SonarCloud | No `SONAR_TOKEN` secret | Step 5 |
| `AlreadyExistsException` | Old entries from lost state | `make import` (Step 6) |
| Pods in `ImagePullBackOff` | No images in ECR yet | Push to `main` or push by hand (Step 8) |
| ExternalSecret not `SecretSynced` | ESO role or store not ready | Step 7 checks |
| ALB targets unhealthy | Health check path wrong | Gateway ingress needs `healthcheck-path: /health` |
| `Invalid name` on Windows | Git Bash path mangling | Use `monitoring_credentials.py` |
| Destroy fails with `DependencyViolation` | Leftover EKS network resources | The cleanup helper normally prevents it; re-run destroy |

---

## Related

- [`ARCHITECTURE.md`](ARCHITECTURE.md) — how everything fits together and why
- [`UML.md`](UML.md) — diagrams of the app: login, checkout, data model
- [`../README.md`](../README.md) — tech stack, repo structure, local development
