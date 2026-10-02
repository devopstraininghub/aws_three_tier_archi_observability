# k8s Folder, Explained — File by File, Service by Service

Friends, this is a **reading guide** for the `k8s/` folder. Open the real YAML file next to this page and follow along.

**Good to know:**
- This is a personal reference note, not the main documentation. The main docs are `docs/ARCHITECTURE.md`, `docs/DEPLOYMENT.md`, and `docs/UML.md`.
- It describes `main` with a single cluster. If the manifests change, this page can go out of date.
- Companion guide: [`terraform-explained.md`](terraform-explained.md).

**How the folder is organized:**
- Everything is under `k8s/`.
- There are **two separate kustomize trees**:
  - `k8s/base/` + `k8s/overlays/` = the **frontend**. Deployed by one ArgoCD `Application`.
  - `k8s/services/<name>/` × 5 = the **backend microservices**. Deployed by one ArgoCD `ApplicationSet`.
- `k8s/argocd/` = the GitOps wiring that connects both trees to the cluster.
- **No Helm** in this folder. Everything is plain kustomize.

**Easy memory trick:** base = the recipe. overlay = the small change for this kitchen (prod). ArgoCD = the cook who follows it.

---

## Part 1 — Frontend tree (`k8s/base/`, `k8s/overlays/`)

### `base/kustomization.yaml`
- The frontend's kustomize base.
- Sets `namespace: bookstore` for everything listed. So the other files need no `namespace:` line (except `namespace.yaml`, which creates it).
- A comment explains why there is **no** `monitoring/` folder:
  - `ServiceMonitor` and `PrometheusRule` need the Prometheus Operator. This cluster does not run it.
  - An unknown resource type fails the **whole** ArgoCD sync, not just that one file.
  - They were deleted on 2026-08-29. Please do not add them back.

### `base/namespace.yaml`
- Creates the `bookstore` namespace.
- Adds Pod Security Standards labels (`enforce`, `audit`, `warn` = `restricted`). This is the strictest level.
- It blocks root pods, privilege escalation, and missing seccomp profiles **at admission**, before the pod ever starts.

### `base/frontend/deployment.yaml`
- The only workload here: 2 replicas, nginx serving the React build.
- `automountServiceAccountToken: false` — this pod never calls the Kubernetes API.
- `runAsUser: 101` — nginx's own unprivileged user. Not a random number.
- `readOnlyRootFilesystem: true` forces three `emptyDir` mounts (`/tmp`, `/var/cache/nginx`, `/var/run`), because nginx must write there.

### `base/frontend/service.yaml`
- `ClusterIP`, port 80 → container port 8080.
- Its only customer is the shared ALB.

### `base/ingress/ingress.yaml`
- One Ingress with `ingressClassName: alb` (AWS Load Balancer Controller, not the retired `ingress-nginx`).
- **The key line:** `alb.ingress.kubernetes.io/group.name: bookstore`.
  - The api-gateway Ingress uses the **same** group name.
  - So the controller builds **one ALB** for both, not two.
- No `certificate-arn` annotation. The controller finds a matching ACM cert by host name (see `terraform/ingress-cert.tf`).
- The host is a placeholder that `scripts/configure.py` replaces. Never hand-edit it.

### `base/network-policy/network-policy.yaml`
- **Two policies:**
  - `default-deny-all` — an empty `podSelector` with both policy types. It blocks everything in the namespace. Every other rule makes a small hole in it.
  - `frontend-policy` — allows ingress only from `170.20.0.0/16` on port 8080.
- It uses `ipBlock`, **not** `namespaceSelector`. The ALB (`target-type: ip`) connects straight to the pod's IP from its own network interfaces in the VPC. It is not a pod in some namespace.
- Egress allows only DNS (53). The browser calls the API directly from the user's device, not from this pod. So no gateway egress rule is needed.

### `base/pdb/pdb.yaml`
- `minAvailable: 1` with 2 replicas.
- This is safe because Kubernetes can still evict one pod during a node drain or upgrade.
- Compare with the backend services in Part 2.

### `base/limitrange.yaml` and `base/quota.yaml`
- **LimitRange:** default values (50m/64Mi request, 250m/128Mi limit) for any future container that forgets to set its own. A backstop, not an active limit.
- **ResourceQuota:** namespace ceiling of 2 CPU / 2Gi requests, 4 CPU / 4Gi limits, 20 pods. Sized for the frontend only.

### `base/storageclass/gp3.yaml`
- A cluster-wide `StorageClass`, the default one, with `reclaimPolicy: Retain`. A deleted PVC does **not** delete the EBS volume.
- Nothing in this repo claims it right now.
- It is on the AppProject's `clusterResourceWhitelist`, because it is cluster-scoped.

### `base/secrets/external-secret.yaml`
- Defines the **`ClusterSecretStore`** `aws-secretsmanager`. Every service's `ExternalSecret` points to it.
- Annotations: `PreSync` hook, `sync-wave: "-2"`.
- **Why the hook matters:**
  - Without it, this would be a normal Sync resource.
  - ArgoCD always applies Sync resources **after** every PreSync hook.
  - So on a fresh cluster every ExternalSecret would fail with "ClusterSecretStore not found". Every time, not by chance.
- The old backend's `db-secret` was removed with the monolith. The store stayed.

### `overlays/dev/kustomization.yaml`
- Uses `../../base` and lowers the frontend replicas from 2 to 1.
- **Not used by ArgoCD.** It is only for local `kustomize build` or `kubectl apply -k`.

### `overlays/prod/kustomization.yaml` + `hpa-frontend.yaml`
- This is what ArgoCD actually deploys.
- Adds an HPA: 1 to 3 replicas, 70% CPU target.
- The `images:` block rewrites `bookstore-frontend:latest` to the real ECR image and tag.
- CI runs `kustomize edit set image` here on every deploy. That is where the "chore: bump image tags" commits come from.

**Easy memory trick:** the base says what the app is. The prod overlay says which image and how many copies.

---

## Part 2 — Backend microservices (`k8s/services/<name>/`)

**The five services:** `catalog-service`, `user-service`, `order-service`, `notification-service`, `api-gateway`.

- Almost the same layout each time.
- Each has its **own namespace**: `catalog`, `user`, `order`, `notification`, `gateway`. This gives real isolation, not just a label.
- `catalog-service` is used as the example below. Differences are listed at the end.

### `base/kustomization.yaml`
- Sets `namespace: <service-namespace>`.
- The order of the resource list does not control apply order. ArgoCD's sync waves and hooks do.

### `base/namespace.yaml`
- Same `restricted` Pod Security pattern as the frontend.
- One `Namespace` entry in the AppProject whitelist covers all of them, because the whitelist matches by kind.

### `base/configmap.yaml`
- Non-secret settings only: `DB_PORT`, `DB_NAME`, `APP_PORT`.
- For `api-gateway` it also holds the other services' internal DNS names and `FRONTEND_URL` (its CORS allow-list, stamped by `configure.py`).

### `base/external-secret.yaml`
- A per-service `ExternalSecret` such as `catalog-db-secret`.
- Reads `/bookstore/catalog-db-credentials` (`DB_USERNAME`, `DB_PASSWORD`, `DB_HOST`).
- `PreSync`, `sync-wave: "-1"`. This is **after** the store (`-2`) and **before** the schema-init Job (`0`).
- **Real incident:** with the wrong wave, the Job failed with `CreateContainerConfigError: secret not found` and retried for 15 minutes.
- `refreshInterval: 1h` — ESO checks Secrets Manager hourly and updates the Kubernetes Secret itself. No redeploy needed.

### `base/admin-db-secret.yaml`
- Copies the shared RDS **admin** login (`/bookstore/db-credentials`) into this service's namespace.
- The schema-init Job needs it to create the database and user.
- No new IAM permission is needed. The store's role already covers `/bookstore/*`.
- Same `PreSync` / `-1` wave.

### `base/schema-init-job.yaml`
- An ArgoCD **PreSync hook** Job.
- **Why a hook?** Jobs cannot be changed after they are created. A normal Job would make every later sync fail. Hooks are exempt.
- `hook-delete-policy: BeforeHookCreation,HookSucceeded`
  - `HookSucceeded` alone never cleans up a **failed** Job. Every sync would then wait on the same broken Job forever.
  - `BeforeHookCreation` removes the old one first, whatever happened.
- The SQL is **idempotent**: `CREATE ... IF NOT EXISTS`, `INSERT ... ON DUPLICATE KEY UPDATE`, and a guarded seed.

**Two sharp edges, both from real bugs:**
- The shell heredoc is unquoted (`<<SQL`) so `$CATALOG_DB_PASSWORD` expands. But an unquoted heredoc also treats backticks as shell commands. The column name `` `desc` `` (a reserved word) was run as a shell command and vanished from the SQL. The fix: escape every backtick as `` \` ``.
- The seed insert runs only if `SELECT COUNT(*) FROM catalog_db.books = 0`. The Job runs on **every** sync. If an admin deletes a default book on purpose, it must stay deleted.

**Label gotcha:** the Job's pod carries `app: catalog-service`. Without that exact label, the namespace's `default-deny-all` policy blocks its way to RDS.

### `base/deployment.yaml`
- `replicas: 1`.
- `prometheus.io/scrape`, `port`, and `path` annotations. This is how Prometheus on the monitoring EC2 finds the pod (through the API server's pod proxy, since pod IPs are not reachable from outside).
- `runAsUser: 1001` — the usual non-root user for Node images. Different from the frontend's `101`.
- **Secrets and config are kept apart:**
  - `DB_HOST`, `DB_USERNAME`, `DB_PASSWORD` come from the ExternalSecret's Secret.
  - `DB_PORT`, `DB_NAME`, `APP_PORT` come from the ConfigMap.
- **Probes:**
  - Readiness: `/health` on 3000, wait 10s. It only controls traffic.
  - Liveness: `/health` on 3000, wait 30s. A longer wait, because a failed liveness check **restarts** the pod.

### `base/service.yaml`
- `ClusterIP`, port 80 → container port 3000.
- Internal only. No service here is `LoadBalancer` or `NodePort`. The only way in from outside is the ALB through the gateway.

### `base/hpa.yaml`
- `catalog-service`: 1 to 5 replicas, scales on CPU (70%) **and** memory (80%).
- Every backend service's HPA has both a CPU and a memory trigger.
- Older notes said only catalog had a memory trigger. That is no longer true, so check the file before you trust any note.

### `base/network-policy.yaml`
- Same pattern: `default-deny-all` plus one scoped allow policy.
- **Pod-to-pod uses `namespaceSelector`** (matching `kubernetes.io/metadata.name`). This traffic really comes from another pod.
- **RDS uses `ipBlock` `170.20.0.0/16`.** RDS is not a pod, so it cannot be selected.
- **Port gotcha (in `api-gateway`'s policy):**
  - Egress must use **3000** (the pod's real port), not **80** (the Service port).
  - NetworkPolicy matches the real destination **after** kube-proxy's DNAT.
  - An older version used 80. It would have silently blocked all gateway → service traffic once enforcement was on.

### `base/pdb.yaml`
- `maxUnavailable: 1`, **not** `minAvailable: 1`.
- With one replica, `minAvailable: 1` means "never evict the only pod". A node drain or upgrade would then **hang forever**.
- `maxUnavailable: 1` is still a real PDB, but it allows normal maintenance.
- This is the **opposite** of the frontend's PDB, on purpose.

### `base/limitrange.yaml`
- Same values as the frontend (50m/64Mi request, 250m/128Mi limit).
- Copied per namespace, because namespaces do not share these.

### `overlays/prod/kustomization.yaml`
- Same `images:` pattern as the frontend. CI runs `kustomize edit set image bookstore-<service>=...:<SHA>` on every deploy.

### What is different per service
- **`user-service`** — adds a `JWT_SECRET` env var. It is the only one that **signs** tokens.
- **`order-service`** — adds `NOTIFICATION_SERVICE_URL`. Its NetworkPolicy also allows egress to the `notification` namespace.
- **`notification-service`** — ingress comes only from the `order` namespace, not from `gateway`. Only `order-service` calls it.
- **`api-gateway`**
  - No `schema-init-job.yaml` and no `admin-db-secret.yaml`, because it has no database.
  - It has its own `ingress.yaml`.
  - Its ExternalSecret pulls `JWT_SECRET` instead of DB credentials.
  - `replicas: 2`, because every external request goes through it.
  - Bigger resources: 100m/128Mi request, 500m/256Mi limit.
  - It has a `pdb.yaml` with `minAvailable: 1`. That is safe because it runs 2 replicas.

### `api-gateway/base/ingress.yaml`
- The other half of the shared-ALB pair.
- Extra annotation: `alb.ingress.kubernetes.io/healthcheck-path: /health`.
- **Why:** the ALB's default health check path is `/`. The gateway has no `/` route (only `/health`). Every target would show 404, stay unhealthy, and get no traffic, even though `kubectl get pods` says `Running` and `Ready`.

**Real-time example:** Pods green, website down. Check the ALB target group health before blaming the app.

---

## Part 3 — ArgoCD wiring (`k8s/argocd/`)

### `appproject.yaml`
- Limits what the apps can do:
  - one source repo
  - one cluster
  - 6 namespaces (`bookstore` + the 5 service ones)
  - 3 cluster-wide kinds: `Namespace`, `StorageClass`, `ClusterSecretStore`
- If you leave out `clusterResourceWhitelist`, **all** cluster-scoped resources are denied.
- Terraform applies it first (`kubectl_manifest.argocd_appproject`). It was once manual-only and was missed on a full rebuild.

### `application.yaml`
- The frontend's ArgoCD Application.
- `source.path: k8s/overlays/prod`, `kustomize: {}` (ArgoCD runs `kustomize build` itself).
- `targetRevision` and `repoURL` are stamped by `configure.py`. A wrong branch gives `unable to resolve '<branch>' to a commit SHA`.
- `automated: { prune: true, selfHeal: true }`:
  - **prune** — delete what you removed from Git.
  - **selfHeal** — undo manual `kubectl` changes.
- `retry` with back-off (5s, doubling, up to 3m, 5 tries) absorbs short hiccups like a secret rotating.

**The one thing to remember about GitOps here:**
- **ArgoCD syncs from the pushed Git branch. Never from your laptop.**
- `configure.py` writes real values locally. That does nothing for the cluster until you **push**.
- This bit a real test run: nothing was pushed, so ArgoCD kept deploying placeholders, and the ESO logs showed `"region":"AWS_REGION_HERE"`.
- Only `argocd.tf`'s own `kubectl_manifest` resources read local disk. Every real workload goes through Git.

### `applicationset-microservices.yaml`
- One ApplicationSet that makes 5 Applications.
- A `list` generator with one `{service, namespace}` pair each.
- Adding a sixth service is **one new list item**.
- It uses `k8s/services/{{service}}/overlays/prod` and `{{namespace}}`.

---

## Part 4 — Patterns that repeat everywhere

- **Sync-wave order:** `ClusterSecretStore` (`-2`) → ExternalSecrets (`-1`) → schema-init Jobs (`0`) → everything else (`0`, normal Sync).
  - A wrong number is not a rare race. On a fresh cluster it is a **guaranteed** failure.
- **NetworkPolicy shape:** every namespace has `default-deny-all` plus one scoped policy per workload.
  - `ipBlock` for the ALB and RDS (not pods).
  - `namespaceSelector` for pod-to-pod.
  - **DNS (port 53) egress is in every policy.** Without it, nothing can resolve a Service name.
- **PDB by replica count:**
  - 2 or more replicas → `minAvailable: 1` is safe.
  - 1 replica → use `maxUnavailable`, or maintenance hangs.
- **Security floor on every container:**
  - `runAsNonRoot: true`
  - `allowPrivilegeEscalation: false`
  - `readOnlyRootFilesystem: true`
  - `capabilities.drop: ["ALL"]`
  - `seccompProfile: RuntimeDefault`
  - `automountServiceAccountToken: false` at pod level
  - Together with the namespace's `restricted` label, it is enforced twice: once at admission and once by the manifest itself.
- **Secrets vs config:** secrets go `ExternalSecret → Secret → secretKeyRef`. Config goes `ConfigMap → configMapKeyRef`. Never mixed.
- **What is not here:**
  - No StatefulSet. RDS is the only stateful part, and Terraform owns it.
  - No PVC uses the `gp3` class yet.
  - No service mesh (Istio, Linkerd). NetworkPolicy and the ALB's TLS are all the traffic control today.

---

## Apply order (real, not folder order)

1. `terraform/argocd.tf` applies the AppProject (reads local disk).
2. `terraform/argocd.tf` applies `application.yaml` and `applicationset-microservices.yaml`.
3. ArgoCD takes over. It checks Git about every 3 minutes, runs `kustomize build` for each path, and reconciles.
4. Inside each sync: `ClusterSecretStore` (`-2`) → ExternalSecrets (`-1`) → schema-init Jobs (`0`, PreSync) → Deployments, Services, Ingress, NetworkPolicy, HPA, PDB (`0`, Sync).
5. `selfHeal: true` keeps repeating this forever and fixes any drift.

---

## Quick recap table

| Concept | One-line meaning | Real-time example |
|---|---|---|
| Kustomize base / overlay | Shared recipe + small per-environment change | prod overlay sets the image tag |
| Sync wave | The order ArgoCD applies things | Secret store (`-2`) before secrets (`-1`) |
| PreSync hook | Runs before the real deploy | schema-init Job creates tables |
| ExternalSecret | Copies a secret from AWS into the cluster | `order-db-secret` |
| NetworkPolicy | Firewall between pods | Only `gateway` may call `catalog` |
| PDB | Limits how many pods can be taken down on purpose | `maxUnavailable: 1` for single-replica services |
| HPA | Auto-scales pods by load | `order-service` 1 to 5 replicas |
| Ingress group | Many Ingresses share one ALB | frontend + gateway on one ALB |
| Self-heal | ArgoCD undoes manual changes | Reverts a hand-edited replica count |

**Easy memory trick:** Secrets first, then the database setup, then the app. Same order every time.
