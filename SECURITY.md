# Security Policy

Friends, this is a **learning and demo project**. It is not a live service holding real user data. But good security habits matter even in a demo, so we take every report seriously.

---

## 1. How to report a vulnerability

Please **do not** open a public GitHub issue for a security problem. Anyone could read it before it is fixed.

**Do this instead:**
1. Open the **Security** tab of this repository.
2. Click **Report a vulnerability**.
3. This uses GitHub's private reporting, so only the maintainers see it.

**Please include, if you can:**
- What the problem is, and what damage it could do.
- Steps to reproduce it.
- Which part is affected: Terraform/AWS, a microservice, the CI/CD pipeline, Kubernetes manifests, and so on.

**Real-time example:** You find that a service accepts a token without checking it. Report privately with the file name and the steps. Do not post it publicly.

---

## 2. What is already covered

Please check this list first. Your finding may already be handled (see the [README security controls](README.md#9-security-controls)).

- **Gitleaks** — scans for secrets in every commit and the full Git history.
- **SonarCloud** — code quality, security hotspots, and a quality gate.
- **Trivy (containers)** — scans every image. Results go to the GitHub Security tab. Today these findings are **reported, not enforced**, so the build does not fail on them.
- **Trivy (Terraform)** — scans infrastructure code and **does** fail the pipeline on CRITICAL or HIGH findings.
- **GitHub OIDC** — CI gets temporary AWS access. No static AWS keys anywhere.
- **External Secrets Operator + AWS Secrets Manager** — no secrets in Git or in Kubernetes manifests.
- **Hardened pods** — non-root containers, read-only filesystems, and Kubernetes NetworkPolicy between services.

**Known open ideas** (good, safe things to discuss):
- Make the Trivy container scan a real hard fail.
- Make the "first user becomes admin" check atomic.

**Easy memory trick:** Check the list → then report privately. Never post a real vulnerability in public.

---

## 3. Scope

**In scope:**
- The code in this repository.
- The Terraform and Kubernetes configuration in this repository.

**Out of scope:**
- Any live deployment that someone else runs from a fork or a copy of this project.
- That deployment's security belongs to the person who runs it.

---

## 4. A small request

If you run this project yourself, please:
- Never commit `config.env`, `.env`, or `terraform.tfvars`.
- Narrow `monitoring_admin_cidr` and `public_access_cidrs` from `0.0.0.0/0` before using it for anything beyond a demo.
- Turn on `deletion_protection` and snapshots on RDS before using this as a template for anything that must stay up.

Thank you for helping keep things safe! 🙏
