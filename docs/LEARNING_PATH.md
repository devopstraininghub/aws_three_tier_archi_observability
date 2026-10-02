# Learning Path

Friends, this project has a lot of tools in one place. If you try to read everything at once, it feels heavy. So here is a **simple step-by-step path**. Take one step at a time. Do not rush.

No need to mug up anything. At each step, you read a little, run a little, and understand the logic. Simple only.

**How to use this page:**
- Each step has a **goal**, **what to read**, **what to try**, and a **check yourself** question.
- Finish a step before you start the next one.
- One step per day is a very good speed.
- It is fine to skip the AWS steps (6 to 9) if you do not have an AWS account yet. Steps 1 to 5 work on your laptop.

**Easy memory trick:** Run it → Pack it → Place it → Create it → Deliver it → Watch it.

---

## Step 1 — Run one service on your laptop

**Goal:** see a microservice working with your own eyes.

**Read:**
- [`README.md`](../README.md), sections 1 and 5.
- `services/catalog-service/app.js`. It is short and easy.

**Try:**
```bash
cd services/catalog-service
npm install
npm test
```
- Tests need **no database**. They use a fake one.
- Read `__tests__/books.test.js` and match each test to a route in `app.js`.

**Check yourself:**
- What does `GET /health` return, and why does every service have it?
- What is `/metrics` for?

**Real-time example:** `/health` is like a doctor tapping your knee. Kubernetes uses it to know if the app is alive.

---

## Step 2 — Understand login (JWT)

**Goal:** understand how a user proves who they are.

**Read:**
- [`docs/UML.md`](UML.md), sections 2, 4, 4b, and 4c.
- `services/user-service/app.js`.
- `services/api-gateway/app.js` (the `verifyJwt` function).

**Try:**
- Run `npm test` in `services/user-service` and `services/api-gateway`.
- Find where the access token gets a 15-minute life.

**Check yourself:**
- Why are there two tokens (access and refresh)?
- Why does a wrong email take the same time as a wrong password?
- What is the difference between `401` and `403`?

**Easy memory trick:** Access token = short-life wristband. Refresh token = one-time ticket stub.

---

## Step 3 — Follow one order from click to database

**Goal:** see how services work together.

**Read:**
- [`docs/UML.md`](UML.md), sections 1, 3, and 5.
- `services/order-service/app.js` — the `/orders/checkout` function.
- `client/src/pages/Checkout.jsx`.

**Try:**
- In `order-service`, find the `beginTransaction`, `commit`, and `rollback` lines.
- Ask: what would go wrong if there were no transaction?

**Check yourself:**
- Why is the notification "fire and forget"?
- Why does the `orders` table not store the price? Is that good or bad?

**Real-time example:** Think of a cashier. The bill is saved first. The SMS receipt comes after. A failed SMS never cancels your purchase.

---

## Step 4 — Pack it with Docker

**Goal:** understand what an image is.

**Read:**
- `services/order-service/Dockerfile` (two stages: `deps` then `runtime`).
- `client/Dockerfile` (build with Node, serve with nginx).

**Try:**
```bash
cd services/order-service
docker build -t order-service:test .
docker run --rm -p 3000:3000 order-service:test
```
- It will not connect to a database. That is fine. Look at the logs and read the error.

**Check yourself:**
- Why do we use two stages?
- Why does the container run as a non-root user?
- Why is `npm` deleted from the final image?

**Easy memory trick:** Build stage = the kitchen. Final stage = the plate you serve. Do not serve the kitchen!

---

## Step 5 — Place it in Kubernetes (read-only)

**Goal:** read manifests without being scared.

**Read:**
- [`k8s-explained.md`](../k8s-explained.md), Part 2.
- `k8s/services/order-service/base/` — read the files in this order:
  1. `namespace.yaml`
  2. `configmap.yaml`
  3. `external-secret.yaml`
  4. `schema-init-job.yaml`
  5. `deployment.yaml`
  6. `service.yaml`
  7. `network-policy.yaml`
  8. `hpa.yaml` and `pdb.yaml`

**Try (no cluster needed):**
```bash
kustomize build k8s/services/order-service/overlays/prod
```
- It prints the final YAML that ArgoCD would apply.

**Check yourself:**
- Which file gives the pod its database password?
- Why is `default-deny-all` there?
- Why does the schema-init Job run **before** the Deployment?

---

## Step 6 — Create it with Terraform (read first, apply later)

**Goal:** understand infrastructure as code.

**Read:**
- [`terraform-explained.md`](../terraform-explained.md), Parts 1 and 2.
- [`docs/ARCHITECTURE.md`](ARCHITECTURE.md), sections 3 to 8.
- `terraform/modules/network/main.tf` — the easiest module.

**Try:**
```bash
cd terraform
terraform init -backend=false
terraform validate
```
- This checks the code without touching AWS. No cost, no risk.

**Check yourself:**
- Why do `rds` and `eks` build at the same time?
- Why is the public Route 53 zone only "looked up"?
- Why does `DB_HOST` use `.address` and not `.endpoint`?

> ⚠️ Do **not** run `terraform apply` yet. Real money starts when RDS and EKS exist.

---

## Step 7 — Deliver it with CI/CD and GitOps

**Goal:** understand how code reaches the cluster without a person typing `kubectl`.

**Read:**
- [`README.md`](../README.md), section 7.
- `.github/workflows/ci-cd.yml` — read the stage comments only. They explain each job.
- [`docs/ARCHITECTURE.md`](ARCHITECTURE.md), section 9.

**Try:**
- Open the repo's git log. Find the commits named `chore: bump image tags to ...`.
- Open `k8s/services/order-service/overlays/prod/kustomization.yaml` and find the `newTag` value. It is the same tag.

**Check yourself:**
- Why does CI never run `kubectl`?
- What does ArgoCD `selfHeal` do?
- Which Trivy step is set to report only, and how would you make it block?

**Easy memory trick:** CI changes the tag in Git. ArgoCD changes the cluster to match Git.

---

## Step 8 — Watch it (observability)

**Goal:** understand how problems get noticed.

**Read:**
- [`docs/ARCHITECTURE.md`](ARCHITECTURE.md), section 6.
- [`terraform-explained.md`](../terraform-explained.md), the `monitoring-ec2` part.

**Try (on a running system):**
```bash
python3 scripts/simulate_load.py --duration 240
```
- Watch which alerts fire, and which email arrives.

**Check yourself:**
- Why does monitoring run on its own EC2 and not inside the cluster?
- Which 9 alerts exist? Name three.

---

## Step 9 — Build the whole thing yourself

**Goal:** do the real deployment, with care.

**Read:**
- [`docs/DEPLOYMENT.md`](DEPLOYMENT.md), all of it, **once** before you start.

**Try:**
- Follow Steps 1 to 8 in order.
- Read the plan before every apply.
- Run `terraform destroy` when you finish, so you do not pay for idle resources.

**Check yourself:**
- What are the two failures you expect before the GitHub secrets exist?
- Why do pods show `ImagePullBackOff` on the first day, and is that a problem?

---

## Bonus ideas (good first improvements)

When you feel confident, try one of these. Each is small and safe to learn from.

- Change the Trivy container scan from **report-only** to a real **hard fail**.
- Make the "first registered user becomes admin" check **atomic**.
- Save the **price** on each order at checkout time.
- Make `notification-service` send a real email, not only a log row.
- Add a Loki row to `scripts/monitoring_credentials.py`.

**Real-time example:** Pick the price change. You edit the `orders` table in the schema-init Job, change the checkout insert, and add a test. That touches SQL, Node, Kubernetes, and CI — the whole pipeline.

---

## Quick recap table

| Step | Topic | One-line meaning |
|---|---|---|
| 1 | Local service | Run it and test it on your laptop |
| 2 | JWT | Short-life token plus a one-time refresh token |
| 3 | Checkout | A transaction for orders, fire-and-forget for notifications |
| 4 | Docker | Package the app into an image |
| 5 | Kubernetes | Describe how the app runs, in YAML |
| 6 | Terraform | Create AWS from code |
| 7 | CI/CD + GitOps | Test, build, tag, and let ArgoCD deploy |
| 8 | Observability | Metrics, logs, and alert emails |
| 9 | Full deploy | Build, test, then destroy |

**Final words:** You do not need to know everything on day one. Even experienced engineers learn this one tool at a time. Keep going. You can do it! 💪

---

## Related docs

- [`../README.md`](../README.md) — overview
- [`ARCHITECTURE.md`](ARCHITECTURE.md) — how it all fits together
- [`DEPLOYMENT.md`](DEPLOYMENT.md) — build it from zero
- [`UML.md`](UML.md) — diagrams of the app
