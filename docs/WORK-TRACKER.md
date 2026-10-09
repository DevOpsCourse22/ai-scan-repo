# Work tracker — what is done, what is not, and what the lecturer asked for

> Last updated: 2026-10-09

## Current umbrella summary

This project now functions as an umbrella repository for the split architecture:

- [ai-scanner-gateway](../ai-scanner-gateway) — GitHub-facing auth and orchestration
- [ai-scanner-backend](../ai-scanner-backend) — review engine and scanning lifecycle
- [ai-scanner-frontend](../ai-scanner-frontend) — user-facing review UI
- [README.md](../README.md) — project overview, architecture context, and documentation hub

The sections below still contain the older planning and execution notes from the earlier monorepo phase. They remain valuable historical context, but the current umbrella view above is the short version of the present repo layout.

## Status and open items (checked 2026-10-09)

Checked by Sahar + Claude against the lecturer's brief and our own docs,
from the repositories (K3S-SRV was not reachable, so nothing below was
checked on the live cluster).

**Done since 2026-10-06:** Jenkins with a pipeline per code repo, a local
registry, GitHub webhooks through smee.io, one Argo CD Application per
deployment repo (closes the old O6), image tags written by CI (closes O8),
the GitHub credential for Jenkins (half of O7), Prometheus and Grafana
installed on the cluster, and the new `ai-scanner-installer` repo (a web
wizard for minikube / a server / EKS, still a draft). Test results on
2026-10-06: backend 244, mcp-server 195, scanner 29, scanner-worker 73,
security-tests 54, gateway 98 - all passing. gitleaks over all repos: no
real secret, only fake keys planted in tests.

### Gaps against the brief

| # | Gap | Who decides / owns | Status |
|---|---|---|---|
| B1 | **Jenkins runs inside Kubernetes**, installed with a Helm chart; the brief says "applicative, NOT in a Docker container". Ask the lecturer before changing anything: either he accepts it (argument: CI agents as short-lived Pods, nothing to maintain on the host), or Jenkins moves to a service on the host | lecturer, then Elad | ❌ |
| B2 | **No Helm in the deployment repos** (still kustomize). The brief names Helm | lecturer (with B1), then Sahar | ❌ |
| B3 | **Prometheus and Grafana are not in git,** and there are no dashboards; the brief puts Grafana in the deployment repos. Terraform sits in the installer repo, not in the deployment repos | Elad | ❌ |
| B4 | **Jenkins has no ArgoCD or Terraform credential,** both named in the brief. The design does not need one (Jenkins only commits, Argo CD deploys), but it should be stated, or an `argocd app wait` stage added that uses a read-only token | Elad + Sahar | ❌ |
| B5 | **Hand-off to the lecturer (item 7):** read access to the repos, a way onto the tailnet, and a short guide to the API for his browser add-on | Sahar | ❌ |

### Security

| # | Item | Severity | Owner | Status |
|---|---|---|---|---|
| S1 | **The pipelines scan the image, not the code.** Add a stage that runs our own scanner (Semgrep, Gitleaks, Trivy fs) on the repository as a gate, with a `.gitleaksignore` for the fake keys planted in tests (`mcp-server/tests/fixtures/dummy_repo`, the scanner-worker, review-api and gateway tests). For a security-scanning project this is the most visible gap | Medium | Sahar | ❌ |
| S2 | **No NetworkPolicy,** although `09-kubernetes-design.md` describes one | Medium | Sahar | ❌ |
| S3 | (was O1) **The site's AI key also serves anonymous reviews.** quick and brief need no login, and every review runs on the service an admin chose in Settings. No cost on LM Studio, but once a paid cloud key is saved, or the site is public on EKS, anyone who reaches it can spend that key. Fix: require login whenever a key is saved | Medium | Elad | ❌ |
| S4 | (was O2) **`POST /reviews/{id}/restart` does not forward the AI service from Settings.** The UI only uses `resume`. Simplest: remove `/restart` | Low | Elad | ❌ |
| S5 | (was O3) **Only `Elad-Eytan` is in `gateway-admins`,** by username. Add Sahar and Aviel in the `id:<number>` form | Low | Elad | ❌ |
| S6 | (was O4) **`main` cannot be protected** on private repos in a free org (GitHub: "Upgrade to GitHub Pro or make this repository public"). Make the repos public, find a GitHub Education upgrade, or keep PRs by agreement | - | team | ❌ |

### Documentation

| # | Item | Owner | Status |
|---|---|---|---|
| D1 | **This tracker:** the requirement table and this section updated 2026-10-09 | Sahar | 🟡 this PR |
| D2 | **`README.md` lists three repositories;** add the three deployment repos and the installer | Sahar | ❌ |
| D3 | **The design docs describe things that were not built**, and read as if they were: OPA/Kyverno admission policy, External Secrets, SBOM and image signing, object storage for reports, "GitHub Actions is the preferred source pipeline" (`03-high-level-design/14-technology-stack.md`), NetworkPolicy. Add one "built vs. planned" page, and a status line at the top of each design doc pointing to it | Sahar | ❌ |
| D4 | **Duplicate copies:** `docs/Zipped/`, `docs/Zipped.zip` and the `.docx` files repeat the Markdown and will drift. Keep one source | Sahar + Aviel | ❌ |
| D5 | **No document of the delivery flow as built:** commit → Jenkins → registry → deployment repo → Argo CD → cluster. `ai-scanner-backend-deployment/CI-PIPELINES.md` explains the setup; the umbrella repo should link to it with one diagram | Sahar | ❌ |

---
Living document. Update it as things land; do not let it drift into a
snapshot. Supersedes `where-we-are-2026-09-21.md`, which is kept only as a
record of that day.

**Deadline: officially 2026-10-22** (moved from 10-14 when the lecturer
cancelled two lessons). **The team's target is earlier: hand-off by
2026-10-12**, because Sahar cannot submit that late in October and wants
to finish as early as possible. **Not confirmed with the lecturer** (Sahar
could not attend the 2026-10-05 lesson), so as of 2026-10-06 the date in
force is 2026-10-22. The days between the target and the official deadline
are a buffer for the risky phases (mainly the cloud), not time to plan
into. The lecturer asked for the hand-off two days before submission: he
is joining the project as a learner, to build a browser add-on on top of
the service. Last updated: 2026-09-30.

---

## The lecturer's requirements (the actual brief)

Everything below is measured against this list. **The lecturer's own list
starts at 1.** Items 0 and 0.5 are ours — added by the team, not required —
and are kept in the same table so there is one list to work from. They can
come later; 0 in particular is real feature work, not configuration.

**Standing decision: no cloud until the end of the project.** Everything
runs on Elad's server (backend, frontend, the lot). AWS comes last, as a
deliberate exercise in "how industry actually does it" — it saves money,
avoids cloud headaches during development, and still ends with a version
deployed to the cloud.

| # | Requirement | Status |
|---|---|---|
| 0 | *(ours, not the lecturer's)* **Admin config page** — change AI model, LM Studio / other LLM endpoint or a cloud model, which deterministic scanners to run (not everyone wants Semgrep), and more as we think of it | 🟡 **Mostly done 2026-10-01** on branch `elad/ui-redesign-pause-resume` (not merged): new UI with its own Settings page (per-user theme, animation, default depth and branch), **Pause / Resume** across worker, review-api, gateway and UI, per-GitHub-account history, and an **AI model** section: site admins (ConfigMap `gateway-admins`, edited with kubectl) pick the service (LM Studio or a cloud API), model and API key (stored encrypted) for every review. Still open: choosing which deterministic scanners run |
| 0.5 | *(ours, not the lecturer's)* **K8s infrastructure** via k3s now, EKS towards the end | 🟡 **k3s ✅ deployed and accepted 2026-09-30.** EKS: Terraform drafted in `ai-scanner-installer/terraform/eks` (VPC, EKS, EFS, ECR, IRSA), not yet validated or run - planned last, on purpose |
| 1 | **Jenkins, applicative** (NOT in a Docker container), with credentials for Docker, AWS, Terraform, Git, ArgoCD | 🟡 **Jenkins runs since 2026-10-09** with the three pipelines, but **inside k3s** (Helm chart `jenkins/jenkins`, `platform/jenkins/values.yaml`, agents as Pods), not applicative as the brief says - **ask the lecturer (B1)**. Credentials: Git (`github-deploy`); AWS prepared for ECR; none for ArgoCD or Terraform (B4) |
| 2 | **Split GitHub into repos**: Frontend, Backend, Gateway + Frontend-deployment, Backend-deployment, Gateway-deployment. Deployment repos hold Helm, K8s, Terraform, Grafana. Docker and Jenkins live in the Backend repo. (lesson of 2026-08-31) | ✅ done 2026-10-06 by Aviel and Elad: six private repos in the org `DevOpsCourse22`, plus this docs repo. The code matches the monorepo's last `main` (8200439, after PR #10), and the monorepo is frozen. Deployment repos hold kustomize for now, not Helm |
| 3 | **CI/CD per code repo** (code repos, not deployment repos), after the split | ✅ **2026-10-09:** a `Jenkinsfile` in each code repo: Test → Build (BuildKit) → Trivy (fails on CRITICAL) → Push (main only) → Deploy (Jenkins commits the new tag to the deployment repo, Argo CD syncs it). It already caught CRITICALs in the frontend image. Missing: a security scan of the CODE (S1) |
| 4 | **Decide where images live** | ✅ a local registry on K3S-SRV (`ai-scanner-backend-deployment/platform/registry`), written by the pipelines; ECR for EKS (installer) |
| 5 | **How to deliver env vars and the whole k8s stack** | 🟡 kustomize in each deployment repo, one Argo CD Application per repo (2026-10-06); Secrets created by hand at hidden prompts, never in git. **Not Helm** (B2) |
| 6 | **Elad reviews the Tailscale-in-K8s implementation** | 🟡 checks done (operator runs; the egress reaches only LM Studio's address and port; deleting the Service removes its tailnet device). Left: the write-up, and removing the temporary `tag:k3s-srv` grant in the tailnet policy |
| 7 | *(the lecturer's own)* **Browser add-on** using the service - built by the lecturer himself, as a learner | ❌ nothing prepared yet: the lecturer has no access to the repos or the tailnet, and there is no guide for building on the API (B5) |

### ⚠️ Three conflicts between the brief and what we told the team

These were discovered on 2026-09-24, **after** task messages had already
gone out. Resolve before the weekend or work gets redone.

1. **Jenkins, not GitHub Actions.** The brief says applicative Jenkins with
   credentials for Docker/AWS/Terraform/Git/ArgoCD. Aviel was told to start
   GitHub Actions. Jenkins also fits the "no cloud" decision better — it
   can run on Elad's server and reach the cluster directly, which a
   GitHub-hosted runner cannot (the cluster is behind Tailscale).
2. **Six repos, not four.** The brief separates each service's *code* from
   its *deployment* (Helm/K8s/Terraform/Grafana). Aviel was told four, with
   one combined `k8s` repo.
3. **Helm, not raw YAML.** The brief names Helm in the deployment repos.
   Everything we have written so far is plain manifests, and Elad's side
   uses kustomize. Helm would mean templating what exists.
   **Decided 2026-09-25 (Elad + Sahar): converge on Helm, but only after a
   first Job runs on plain YAML.** Converting YAML that already works is
   mechanical and checkable (`helm template` output against the known-good
   manifests); converting YAML that does not yet work means debugging two
   things at once. The Tailscale operator is already installed with Helm.

Also unaddressed: **Grafana** and **Terraform** appear in the brief and in
no plan of ours yet.

---

## Plan to hand-off (2026-10-01 → target 2026-10-12)

Written 2026-09-30, after the first deployment was accepted. Owners are
recommendations for the team to confirm, not assignments. Dates aim at the
early hand-off; the official deadline (10-22) is the buffer.

**The critical path is items 2 → 1 → 3** (repo split, Jenkins, CI/CD per
repo). None has started, and each depends on the one before. Everything else
is secondary to it.

### Decisions for 2026-10-01, before anyone starts

| Decision | Recommendation |
|---|---|
| Who splits the repos, and by when | Aviel, by 10-03. If there is no progress by the evening of 10-02, someone else takes it |
| Where images live (item 4) | Local registry on K3S-SRV for the Jenkins phase; ECR with EKS |
| Helm | Now, written straight into the new deployment repos, checked with `helm template` against today's working manifests |
| CD tool | ArgoCD watching the deployment repos (already anticipated in the kustomization); Jenkins deploying directly is the fallback |
| Model pinning | Through Elad's settings screen: an admin picks one model |
| Cloud scope | EKS + ECR only. Postgres stays in the cluster (no RDS). One-node group, so the shared `ReadWriteOnce` reports PVC works without EFS |
| The lecturer's access | Add him to the tailnet (share K3S-SRV) or give him the EKS address - decide which, so he can reach the service on hand-off day |

### Phases

| Dates | Phase | Work | Recommended owner |
|---|---|---|---|
| 10-01 → 10-03 | **Unblock** | Split into 6 repos | Aviel |
| | | Jenkins on K3S-SRV as a service, with credentials for Git, the registry and the cluster | Aviel or Sahar |
| | | Local registry on K3S-SRV; k3s configured to pull from it | Sahar |
| | | Settings screen + UI redesign (item 0) | Elad |
| | | `restart` in `KubernetesJobStore` - without it a stopped review cannot resume in kubernetes mode | Elad |
| | | Merge PR #8 (gateway reconnect + session id out of logs), new gateway image | Elad |
| 10-04 → 10-07 | **CI/CD** | Pipeline per code repo: tests → **our own scanner as a gate** (`repo-scan --fail-on`) → image build → Trivy image scan → push to the registry → bump the tag in the deployment repo | Sahar + Aviel |
| | | Helm charts in the three deployment repos | Sahar |
| | | ArgoCD on the deployment repos | Elad |
| | | NetworkPolicy - default deny plus an allowlist; claimed in the security docs, still missing | Sahar |
| 10-05 | **Talk with the lecturer** | Confirm the early hand-off date and his access | Sahar |
| 10-08 | **Monitoring** | Prometheus + Grafana (kube-prometheus-stack via Helm): pod health and review Jobs. Grafana is named in item 2 | Sahar or Elad |
| 10-09 → 10-10 | **Cloud** | Terraform: EKS + ECR; images to ECR; the same charts with EKS values | Elad + Sahar |
| 10-10 night | | Full deep scan for the demo - it takes 5+ hours (16 of 144 files took ~36 min on 2026-09-30) | whoever holds LM Studio |
| 10-11 | **Freeze and hand-off package** | No new features. Docs, talking points, hand-off package for the lecturer (below), rehearsal | all |
| **10-12** | **Hand-off to the lecturer** (target) | | |
| after | Lecturer's add-on | Support him; change only what the integration needs | all |

### Preparing the hand-off for the lecturer's add-on

He does not know how add-ons are built and is joining to learn, so the goal
is a service that is easy to integrate with, not an add-on of ours.

- **Reachability:** an address he can open (tailnet share or EKS - decision
  above), with GitHub login working for his account.
- **An integration guide:** the gateway endpoints an add-on calls
  (`POST /reviews`, `GET /reviews/{id}`, `/files`, `/report`, `/auth/me`,
  `/auth/github/login`), which modes need login (deep does; quick and brief
  do not), and how auth works (session cookie from GitHub OAuth).
- **CORS for an extension:** a browser extension calls from its own origin
  (`chrome-extension://<id>`), so that origin must be allowed in
  `GATEWAY_CORS_ORIGINS` for requests that carry the session cookie.
- **The simplest possible first add-on:** a button on a GitHub repo page that
  opens our UI with the repo filled in. That needs the UI to accept a
  `?repo=<url>` query parameter - a small frontend change, worth making before
  hand-off so his first version works without touching auth at all.

### If time runs short, drop in this order

1. Grafana beyond a basic dashboard.
2. ArgoCD - Jenkins deploys directly.
3. Settings screen extras beyond model pinning.

**Not to drop:** Jenkins, CI/CD, Helm and the cloud version - they are the
brief.

### Carried as documented limitations, not fixed

- Review state is in memory: a review-api restart loses running reviews.
- `noexec` is not enforced on local-path.
- ETA and running file are not shown in kubernetes mode.

---

## Where the code actually is

Branch: `main`. Everything below is merged there, except PR #8 (gateway
reconnect and session ids out of logs), open for review.

### Done

- **K8s execution mode** in the backend (`review_api/k8s_jobs.py`) with
  unit tests. Never run against a real cluster.
- **RBAC** — `k8s/review-api/rbac.yaml`, least-privilege, derived from the
  actual API calls. Elad confirmed in review that it matches the calls in
  the code. ⚠️ not yet validated against a live cluster.
- **Manifests written**: namespace, reports PVC, `reports-noexec`
  StorageClass, gateway Deployment+Service, review-api Deployment+Service,
  Postgres StatefulSet.
- **One manifest folder, one namespace** (2026-09-30): `Code-Review-MCP/k8s/`,
  one sub-folder per service, all in `ai-scan-repos`, applied with
  `kubectl apply -k`. Elad's `K8S-Tailcale-Egress/` (namespace `code-review`)
  was merged in: the `lmstudio` egress Service, the operator values and
  policy, the kustomization and the manual Job tests moved; his old
  review-api Deployment was superseded by `k8s/review-api/deployment.yaml`.
  Every manifest line is commented. `k8s/README.md` is the deploy guide.
  ⚠️ not yet applied to the cluster; the old `code-review` setup still runs
  there until then.
- **Deterministic scanners are in the review-api image** (Trivy + Gitleaks
  copied from pinned upstream images, Semgrep via the scanner package).
  ⚠️ the image has not actually been built since that change.
- **Verified on the real cluster**: Postgres starts under Pod Security
  `restricted`. **`noexec` mount options are NOT enforced** by k3s's
  local-path provisioner.
- **Fixed in the AI engine**: retry token reuse; reasoning-only responses
  reported honestly; reports name the model that actually answered.
- **File-by-file progress view** (plan file + API + gateway proxy + React
  tree).
- **Pause and resume** (branch `elad/ui-redesign-pause-resume`, 2026-10-01,
  not merged): Pause deletes the Job and writes `<id>.paused.json` on the
  reports volume; Resume starts a new Job that reuses the checkpoint,
  pinned to the original commit (job contract v3, `COMMIT`,
  `scanner-worker:0.4.0`). review-api rebuilds a review's status from the
  volume after a restart or reboot, so paused and interrupted reviews stay
  resumable; the gateway caches the last status per review in Postgres
  (new nullable columns, added on startup) so history works days later.
  Also fixes: UI Restart returned 500 in cluster mode (now resumes), and
  `scanning`/`retrying` were missing from review-api's active statuses.
- **Round 2 on the same branch (2026-10-01)**: AI model in Settings
  (site-wide, admins only, encrypted key, contract v4 `LLM_API_KEY`,
  `scanner-worker:0.5.0`); resume reuses the saved scanner results for the
  same commit instead of re-running Semgrep/Gitleaks/Trivy; report opens
  with High/Medium/Low circles and a per-file chart; Delete review with a
  warning dialog (`DELETE /reviews/{id}` end to end).

### First deployment — ✅ achieved 2026-09-30

Deployed with PR #7 and accepted against all six criteria of the definition
of success (`docs/k8s-deployment-tests-2026-09-30.md`). What used to block it:

| Item | Now |
|---|---|
| Secrets (`gateway-secrets`, `postgres-credentials`) | ✅ created at hidden prompts (`k8s/DEPLOY-K3S.md` step B4); none in git |
| Ingress | ✅ applied and working: `http://100.111.216.122/` (tailnet only) |
| Images | ✅ built and imported into k3s, tagged by commit (`1ea5c7d`); worker `scanner-worker:0.3.0` |
| review-api creating Jobs (kubernetes mode) | ✅ Jobs created and completed on the cluster, seen in Kubernetes events |
| scanner-worker image | ✅ exit 0 / 1 / 2, `.error.json`, `.progress.json` (contract v2) |
| One namespace, one manifest set | ✅ everything in `ai-scan-repos`, one `kubectl apply -k` |
| Manifest bugs from Elad's review | ✅ all fixed in PR #7 |
| `mcp-server/Dockerfile` runs as root | Not part of the deployment - the worker image replaced it for the cluster. Still a repo finding (Trivy DS-0002) |

### Not done — hardening and debt

- **NetworkPolicy** — claimed in the security docs, does not exist.
- **Decide what to do about `noexec`** — the StorageClass declares a
  control that was measured not to work on this cluster.
- **`runAsGroup`** missing on gateway and backend (Trivy LOW).
- **Postgres without `readOnlyRootFilesystem`** (Trivy HIGH) — needs
  `emptyDir` for `/tmp` and `/var/run/postgresql`, untested.
- **Review state is in memory** — a review-api restart loses running
  reviews. Port to Postgres.
- **Found 2026-09-30, open:** PR #8 - the gateway failed its first request
  after Postgres restarted (no `pool_pre_ping`), and the traceback wrote a
  live session id into the Pod log (no `hide_parameters`). Fixed with tests,
  awaiting review. Also: `elapsed_seconds` keeps counting after a review
  finishes; `/providers` answers without login; one "Failed to fetch" in
  the UI during a deep scan, cause unknown (the Job kept running).
- **`KubernetesJobStore` has no `restart` / `retry_failed`** — now a
  practical blocker: the deep scan stopped on 2026-09-30 cannot resume from
  its checkpoint through the UI (Resume returns 500; a new scan gets a new
  id and starts from zero). Planned for Elad, 10-01 → 10-03. Those
  endpoints will 500 in kubernetes mode.
- **Model pinning** — a request naming a second model made LM Studio load
  it alongside the first and dropped throughput ~30x. In the cluster every
  Job reaches the same single GPU, so the platform should pin one model
  while one card is behind the endpoint. Team decision, not made.
- **Root `app.py`** — old "Track 3 spike" code, not part of
  `Code-Review-MCP/`. Candidate for deletion.
- **Validate the ETA against a real model** — when LM Studio is up, with
  one model loaded. So far it is validated only on the mock, where every
  file takes about the same time: predicted 313s left at 28/123 files,
  actual 308s. A real model's time per file varies far more (641 reasoning
  tokens were spent on a 6-line file), so expect a noisier estimate that
  needs ~10 measured files to settle. How to check: run a small deep scan,
  note `eta_seconds` and `estimate_based_on` at ~10 files, and compare with
  when it actually finishes. The page shows "estimate from N files" so a
  thin estimate is visible as one. If the error is large even after ~10
  files, the fit (`_fit_file_time` in `review_api/app.py`) needs revisiting
  — not the display.
- **The Job deadline will cut real deep reviews short** (found reviewing
  PR #4, 2026-09-27). `ACTIVE_DEADLINE_SECONDS` in `review_api/k8s_jobs.py`
  defaults to 30 minutes, while a deep review on the real model takes hours
  (minutes per file were measured). A deadline kill is a SIGTERM - retryable,
  and each attempt resumes from the checkpoint - but with `MAX_ATTEMPTS = 5`
  any review needing more than ~2.5 hours fails while still making progress.
  A deep run of Test-Repo-Public would not finish. The deadline exists to
  catch a *stuck* Job, and wall-clock time is a poor signal for that; the
  worker's `<id>.progress.json` already carries a good one in `updated_at`.
  **Decided 2026-09-28 (Elad): stuck = progress file stale.** The worker
  refreshes `updated_at` at least once a minute while anything happens, in
  every mode; review-api deletes and retries an attempt whose progress is
  older than 15 minutes (`REVIEW_K8S_STALE_SECONDS`), and the deadline is a
  12-hour backstop (`REVIEW_K8S_JOB_DEADLINE_SECONDS`). Unit-tested; not
  yet run by review-api on the cluster. Raised on PR #4.
  **Follow-up (2026-09-30, Sahar):** the clone and the scanners report no
  progress, and the scanners alone may run 3 x 900 s, so a large repo was
  killed as stuck in the scanning phase on every attempt. The worker now
  keeps `updated_at` moving through the `cloning` and `scanning` phases only
  (each bounded by its own timeout) - not the AI phase, where a hung model
  must still go stale. The rule is in the job contract.
- ~~**`docs/k8s-manifest-talking-points.md` is now wrong about namespaces**~~
  ✅ rewritten 2026-09-30, when `ai-scan-repos-jobs` was deleted.

---

## Found in Elad's review (2026-09-25)

Elad reviewed the contract, `k8s_jobs.py` and the `k8s/` manifests by
reading the code (the server was off). Every bug below was then checked
against the code and confirmed.

### Decision: how review-api learns *why* a Job failed

**Chosen: an error file on the shared PVC, not the Pod's termination
message.**

The worker already exits 0 / 1 (retryable) / 2 (permanent) and writes a
one-line reason to `/dev/termination-log`. What was missing is how
review-api reads that. Two ways:

- **Read the Pod status through the API** — needs `list` on pods, because a
  Job's Pod has a generated name. RBAC cannot narrow `list` to "only the
  Pods of my Jobs": it is every Pod in the namespace. And every Job Pod
  carries the same env as its Job (`REPO_URL`, `SYSTEM_PROMPT`), so this
  would reopen exactly what granting `get` on `jobs/status` only — and not
  on `jobs` — was designed to close: which repositories everyone is
  scanning.
- **Read `<REVIEW_ID>.error.json` from the PVC** — ✅ chosen. The worker
  writes `{"permanent": true|false, "error": "<the line>"}`; review-api
  reads it from the PVC it already mounts — the same channel it already
  uses to detect success (`<id>.json` exists). **No RBAC change.** No file
  (an OOM kill, a node dying) means "unknown", which is treated as
  retryable — the right default.

The termination-log stays either way: writing it costs nothing, and it is
what `kubectl describe pod` shows a human.

### Who fixes what

**Elad — `k8s_jobs.py` and the worker:** ✅ all done on
`elad/scanner-worker` (2026-09-26) — valid Job names (slug + hash of the
full id), label ≤ 63 with the full id in an annotation, a 404 ends the
review and other API errors give up after `MAX_POLL_ERRORS`, error.json
read (permanent stops at once; stale file removed before each attempt),
`LMSTUDIO_MODEL` passed to the Job, sidecars deleted with their report.
What was asked:
- Job name is built from `review_id` without lowercasing, and the API
  accepts uppercase and `_` in ids — a review named `K3S-01` fails at Job
  creation. The test uses `"abc"`, so it never caught this.
- The `ai-scan.io/review-id` label is capped at 63 characters; the API does
  not cap `review_id` length.
- The poller does `except Exception: continue` inside `while True`: a Job
  deleted by anyone other than `cancel()` returns 404 forever and the
  review is stuck in `reviewing`.
- Worker writes `<id>.error.json` on every failure it detects itself;
  review-api reads it — permanent stops at once with the reason shown,
  missing file is retried. Keep `.error.json` out of the report listing in
  the API and the UI.
- Worker writes **`<id>.progress.json`** (contract change 6 below). Without
  it the progress view is blind in kubernetes mode: the running file and
  both time estimates come from progress messages the engine sends *in
  process*, and a Job pod sends none to review-api. File states still show
  either way, since those come from the plan and checkpoint on the PVC.

**Sahar + Claude — the reading side of `<id>.progress.json`**, in
`GET /api/v1/reviews/<id>/files`: when the job store has no live timing
(kubernetes mode, or after an API restart), read it from the file instead.
Ours because the endpoint and its estimate are. **Unblocked (2026-09-27):**
the worker writes the file and its format is in contract v2; PR #4's
`job-kill.yaml` test watches it update live.

**Sahar — `k8s/` manifests** (starting after `elad/scanner-worker` is merged,
since that branch touches manifests too). ✅ **All four fixed 2026-09-30** in
the manifest reorganization (`backend.yaml` is now `k8s/review-api/deployment.yaml`):
`/tmp` emptyDir, `LMSTUDIO_BASE_URL=http://lmstudio:1234/v1`,
`SCANNER_WORKER_IMAGE=scanner-worker:0.3.0` plus image tags in
`k8s/kustomization.yaml`, and `ai-scan-repos-jobs` deleted:
- `backend.yaml` — `readOnlyRootFilesystem` with no `emptyDir` on `/tmp`.
  `/readyz` checks the temp directory is writable, so the Pod is **never
  Ready**.
- `backend.yaml` — no `LMSTUDIO_BASE_URL`, so Jobs get the image default
  `host.docker.internal`, which does not exist in k3s. Correct value:
  `http://lmstudio:1234/v1` (the Tailscale egress Service).
- `backend.yaml` — `SCANNER_WORKER_IMAGE` defaults to `:latest`. With
  `:latest` Kubernetes always tries to pull, there is no registry, and
  every Job sits in `ErrImagePull`. Use a commit tag.
- `namespace.yaml` — delete `ai-scan-repos-jobs`. A PVC belongs to one
  namespace, and the Jobs must mount the reports PVC, so the Jobs cannot
  live anywhere else. The second namespace is not an open option; it
  contradicts the design.

### Contract changes — ✅ in `docs/job-container-contract.md` v2 (2026-09-26)

1. Exit code **2 = permanent**, no retry. Reason on `/dev/termination-log`
   *and* in `<id>.error.json` (above).
2. Optional **`LMSTUDIO_MODEL`** input — today the user's model choice is
   silently dropped. Ties into the open model-pinning decision.
3. **`TMPDIR=/workspace`** — the engine clones via `tempfile`, i.e. into
   `/tmp` (512Mi in the Job) while the 2Gi `/workspace` sits unused.
4. **Scanners need the internet and a writable home.** Semgrep fetches its
   rules and Trivy its database on every run; with a read-only root
   filesystem they can fail silently and produce an AI-only report — which
   point 5 of the success definition forbids.
   - Short term: point `HOME` and cache dirs at `/workspace`; set git's
     `safe.directory` in the *system* config (otherwise changing `HOME`
     makes Gitleaks find nothing, with no error).
   - Long term: bake the rules and the DB into the image. **Mandatory once
     NetworkPolicy exists.**
   - Measure Trivy's DB against the 512Mi limit.
   - Probably true of the review-api image too, since it gained the scanners.
5. The token: the worker already removes it from `os.environ` before the
   engine runs, so git and the scanners do not inherit it. The contract's
   "deleted from memory after the clone" cannot literally be honoured for an
   env var — reword it, or deliver the token as a file from a mounted Secret.
6. **Progress file** — the worker writes `/reports/<REVIEW_ID>.progress.json`
   on every file start and finish (not on heartbeats), atomically (temp file
   + rename, like the checkpoint), and deletes it with the checkpoint at the
   end:
   ```json
   {
     "current_file": "src/app.py",
     "current_started_at": 1758830000.12,
     "durations": [["src/a.py", 41.2], ["src/b.py", 188.0]],
     "updated_at": 1758830012.0
   }
   ```
   - Timestamps are **wall-clock epoch seconds**. review-api uses a monotonic
     clock in process, but a monotonic clock does not carry between pods.
     (One node on k3s, so one clock; NTP-synced nodes on EKS.)
   - `durations` covers files finished in **this attempt** only, each timed
     from its own start. A retry is a new Pod and starts the list fresh, so
     files restored from the checkpoint never enter the estimate — the same
     rule the in-process estimate follows, and has a test for.
   - Paths and seconds only. No content, no findings.

### For EKS, not urgent

The reports PVC is `ReadWriteOnce`, shared by review-api and every Job.
That works because k3s is one node. On EKS it will need EFS
(`ReadWriteMany`).

---

## Definition of "the deployment succeeded"

Agreed target. Until all six hold, do not call it done.

1. All pods `Running`; review-api passes `readyz`.
2. The UI opens **in a browser through the Ingress**, not via port-forward.
3. GitHub OAuth login works and a review can be started.
4. **A real Job appears in the cluster** — visible in `kubectl get jobs`.
5. The report reaches the PVC and comes back to the UI, **with
   deterministic findings, not AI-only**.
6. Deleting the Postgres pod does not lose data.

Point 4 is the difference between "running on Kubernetes" and "using
Kubernetes".

---

## Before any long scan

1. Exactly **one** model loaded in LM Studio (the 26B).
2. Measure throughput first: `python Code-Review-MCP/scripts/bench-models.py`.
   Healthy is ~110 tok/s. 3.7 tok/s once cost 50 minutes before anyone noticed.
3. Never restart the backend mid-scan — review state is in memory.

## Working notes

- Local checkout: **`C:\dev\AI-Scan-Repos`**. The OneDrive folder is empty
  and abandoned — OneDrive corrupts `.git`.
- Elad's cluster: `ssh elad@100.111.216.122`, `KUBECONFIG=~/k3s.yaml`.
  That kubeconfig is **cluster-admin**; replacing it with a namespace-scoped
  ServiceAccount is still open (Elad approved, "document it").
- LM Studio host: `100.93.244.39:1234`. Frequently offline; check before
  assuming.
