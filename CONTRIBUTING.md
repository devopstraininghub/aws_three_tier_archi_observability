# Contributing

Friends, thank you for wanting to help! This is a **learning project**. It is a real, working example of a three-tier AWS architecture, cut into microservices, with Terraform, EKS, ArgoCD, and a DevSecOps pipeline.

We do not run a big open-source process here. But issues and pull requests are **welcome**, and beginners are especially welcome. Every expert was a beginner once. 🙂

---

## 1. Before you open a PR

Please do these small things first. They save everyone time.

- **Read [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md).**
  - It explains how the system really works today.
  - It also records past problems and why things are built this way.
  - A change that looks like an "obvious improvement" may already have a reason it was not done.
- **Not sure where to start?** Read [`docs/LEARNING_PATH.md`](docs/LEARNING_PATH.md).
- **Infrastructure change?** Go inside `terraform/` and run:
  - `terraform fmt`
  - `terraform validate`
- **App change?** Each service has its own tests:
  - `cd services/<service-name>`
  - `npm test`
  - Local setup is in the [README](README.md#5-local-development).
- **Keep the PR small.** One change per PR.
  - Do not mix in unrelated formatting or refactoring.
  - Small PRs are easy to review and quick to merge.

**Easy memory trick:** Read first → change one thing → test it → then open the PR.

---

## 2. A simple PR checklist

Copy this into your PR description.

- [ ] I read `docs/ARCHITECTURE.md`.
- [ ] I changed only one thing.
- [ ] Tests pass (`npm test`) for the services I touched.
- [ ] `terraform fmt` and `terraform validate` pass (if I touched Terraform).
- [ ] I did not commit any secret, `.env`, `config.env`, or `terraform.tfvars`.
- [ ] I updated the docs if behaviour changed.

**Real-time example:** You fix a bug in `order-service`. You change `services/order-service/app.js`, add a test in `__tests__/orders.test.js`, run `npm test`, and open a small PR. That is a perfect contribution!

---

## 3. Reporting bugs

Open a GitHub issue. Please include:

- **What you expected** and **what actually happened**.
- Your **Terraform / Kubernetes version** and **AWS region**.
- **Logs**, for example:
  - `terraform apply` output
  - `kubectl describe` or `kubectl logs`
  - ArgoCD sync errors

A clear bug report is half the fix.

---

## 4. Security issues

Please **do not** open a public issue for a security problem. Follow [`SECURITY.md`](SECURITY.md) instead.

---

## 5. Be kind

- Ask questions freely. There are no silly questions here.
- Explain your reasoning in the PR. It helps reviewers and helps you learn.
- Be patient with feedback. It is about the code, not about you.

You can do it. Happy contributing! 💪
