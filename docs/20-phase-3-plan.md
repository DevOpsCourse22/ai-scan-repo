# Phase 3 — platform build-out

**Submission: 14 October 2026.** Written on 7 September — **37 days**, for
a part-time team of three.

Agreed after the lecturer review. Items **1–6** are the lecturer's
instructions; **0** and **0.5** are the team's own additions; the
**incremental review** item came out of the same session; **item 2.5**
(consolidation) came from the follow-up consultation.

## What 37 days means

This plan is larger than the time available if every item is taken to
completion. That is not a reason to panic, but it is a reason to decide
the cut order **now**, while it is a choice, rather than in October, when
it will be an accident.

**Must land — these are what the course is assessed on:**
K3s running for real · CI/CD per repo · the repo split · Jenkins ·
secrets delivered properly

**Cut first, in this order, if time runs short:**

1. **AWS / EKS / ECR** — already scoped as an end-of-project exercise. It
   is the cheapest thing to drop because dropping it changes nothing that
   came before, and the plan already keeps each cloud counterpart behind
   a swappable interface.
2. **Diff-scoped review** — valuable, but it is a product feature, not a
   platform one.
3. **Admin config page** — ours, not the lecturer's, and partly built
   already. A reduced version (provider + scanner toggles only) is a fine
   landing point.

Nothing above the line depends on anything below it. That is deliberate.

This document records the plan and, where it matters, the trade-off — so
a choice we make now is one we can defend later rather than one we
discover the cost of halfway through.

---

## Standing decision — no cloud until the end

Everything runs on **Elad's server** for the rest of the build: backend,
frontend, gateway, scanners, K3s. AWS comes at the very end, as a
deliberate exercise in shipping to a cloud rather than as the environment
we develop in.

**Why this is the right call:** it removes a bill, an IAM learning curve,
and a whole class of "it works locally" problems from the critical path,
at a stage where none of them teach us anything we are not already
learning. It also means the AWS step at the end is a real migration with
a real before-and-after, which is worth more than having been on AWS the
whole time.

**What it costs, and how we cover it:** anything cloud-shaped stays
untested until late. The mitigation is to keep every cloud-specific
choice behind an interface we can swap — image registry, secret store,
ingress — rather than assuming K3s semantics everywhere. Each item below
names its cloud counterpart for that reason.

---

## 0 — Admin configuration page

**Owner: Track 3 (gateway + dashboard).** Partly built already.

A page where an operator, not a developer, sets:

| setting | state today |
|---|---|
| which AI provider / endpoint | **built** — `GET /providers`, `provider` on `POST /reviews` |
| which model within a provider | review-api has `GET /api/v1/models`; not exposed by the gateway |
| LM Studio vs. a hosted API | works today — a provider is just a named endpoint |
| **which scanners run** | **not built** — all three always run |
| review depth defaults, size caps, retention | not built |

**The scanner toggle is the real new work**, and it is smaller than it
looks: ADR-007 already makes each scanner an adapter behind a registry,
so "run Semgrep or don't" is a filter over that registry rather than a
change to the runner. Not everyone wants Semgrep; Trivy alone is a
reasonable posture for a dependency-focused scan.

**One design rule for this page:** a setting that changes what gets
*checked* must be recorded in the report. If an operator turns Semgrep
off, every report produced afterwards has to say so — otherwise "no
findings" silently changes meaning, which is exactly the failure the
`scanners_run` / `scanners_failed` split exists to prevent.

**Access control:** this page changes behaviour for everyone, so it needs
a real gate — not "unlisted URL". Simplest defensible option now: restrict
to a configured list of GitHub user ids, checked server-side.

---

## 0.5 — Kubernetes, via K3s now and EKS later

**K3s on Elad's server** for the whole build; **EKS** near the end.

K3s is a full, conformant Kubernetes — Deployments, Services, PVCs, RBAC,
Secrets all behave the same. What differs is what surrounds it, and those
are the parts to keep swappable:

| | K3s (now) | EKS (later) |
|---|---|---|
| storage class | `local-path` | `gp3` via the EBS CSI driver |
| ingress | Traefik (bundled) | ALB Ingress Controller |
| load balancer | ServiceLB / NodePort | real ELB |
| registry pull | local or GHCR | ECR with IRSA |

**The one that bites:** `local-path` is `ReadWriteOnce` and node-local. Any
manifest that assumes two Pods can share a volume will work by accident on
a single-node K3s and fail on EKS. Write the PVCs as if the cluster had
more than one node from the start.

Manifests are written by hand first (Sahar), then packaged as Helm under
item 2's deployment repos.

---

## 1 — Jenkins, running as an application

Not the Docker-delivery Jenkins — a real installation with credentials for
Docker, AWS, Terraform, Git and ArgoCD.

### Treat the Jenkins host as the highest-value target in the system

This is a DevSecOps project, so it is worth saying plainly: a box holding
AWS credentials **and** the Docker socket **and** Git write access **and**
ArgoCD's token is more valuable to an attacker than anything it deploys.
Compromising it grants everything downstream at once, and it is reached
through a web UI that runs other people's code by design.

Concretely:

- **Docker socket access is root on the host.** A job that can run
  `docker run -v /:/host` owns the machine. Prefer a rootless builder
  (Kaniko, Buildah) over mounting `/var/run/docker.sock`.
- **Scope the AWS credentials to what a pipeline actually does** — push an
  image, apply a plan — not `AdministratorAccess` because it was quicker.
- **Do not let Jenkins hold long-lived cloud keys** once we are on AWS:
  OIDC federation gives a job a short-lived role instead.
- **Pipelines defined in the repo** (`Jenkinsfile`), reviewed like code,
  rather than configured in the UI where changes leave no diff.

None of this slows the build down much, and all of it is directly on the
course's subject.

---

## 2 — Split the monorepo

**Six repos, confirmed:** a code repo per component and a deployment repo
per component.

| repo | contents |
|---|---|
| `frontend` | React dashboard |
| `backend` | review-api, mcp-server, **Dockerfiles, Jenkinsfile** |
| `gateway` | FastAPI gateway |
| `frontend-deployment` | Helm, K8s, Terraform, Grafana |
| `backend-deployment` | Helm, K8s, Terraform, Grafana |
| `gateway-deployment` | Helm, K8s, Terraform, Grafana |

### This is the standard pattern, and it is standard for good reasons

Separating deployment config from source is what GitOps *requires*, not a
stylistic preference: ArgoCD watches a deployment repo and reconciles the
cluster to it, so a deploy becomes a commit — with a diff, an author and
an audit trail — and a rollback becomes `git revert`. Keeping config in
the code repo also creates a loop where CI commits a new image tag into
the repo it just built, retriggering itself.

A code repo per service is equally standard: independent release cadence,
per-repo CI that only runs what changed, access scoped per component, and
a build that does not rebuild three services because one changed.

**What the pattern costs — and what industry does about it.** Almost every
change in this project so far has crossed components; this week alone, the
provider selector touched the gateway, its tests, the API client and the
form as one reviewable change. After the split that becomes two PRs in two
repos that must merge in order, with a contract between them that nothing
checks unless we make it.

Teams that run this successfully close that gap deliberately. We should do
the same three things, at the same time as the split — not afterwards:

1. **Freeze the gateway ↔ review-api contract first**, in
   `07-api-specification.md`, and version it. Once the repos are split,
   an unversioned contract breaks silently across repo boundaries.
2. **Move contract tests with the consumer.** The gateway's tests that
   mock review-api are the only thing that will notice a drift.
3. **Split when the cross-component work is quiet**, not mid-feature.

**Suggested order:** deployment repos first (pure addition, breaks
nothing), then the code repos once item 3's pipelines are proven on one of
them.

### How six repositories are actually managed day to day

Yes — six independent repositories, each with its own `.git`, its own
history, its own clone. There is no parent repository tying them
together, and that is the point: each one can be released, reviewed and
permissioned on its own.

**In practice, clone them side by side:**

```
FinalProject/
├── frontend/              ├── frontend-deployment/
├── backend/               ├── backend-deployment/
├── gateway/               └── gateway-deployment/
```

**For running the whole system locally**, keep one `docker-compose.yml`
that builds from the sibling directories. It can live in one of the
deployment repos or in a small seventh "workspace" repo — either works;
what matters is that starting the stack stays one command after the
split, or people will stop running the whole thing locally and integration
problems will move to CI.

**Do not use git submodules.** They look like the answer to this exact
problem and are a well-known source of detached HEADs, stale pointers and
"it works on my machine" for teams this size. Plain sibling clones plus a
short script that runs `git pull` across all six is enough.

**What genuinely changes for a developer:**

- A change touching two components is two PRs, in two repos, merged in
  dependency order (the provider one first, then its consumer).
- `git log` no longer tells you the story of a cross-component change —
  the commit messages have to, and issue links become how they are tied
  together.
- Each repo needs its own README that says how to run **that** component
  alone, because that is now how people will encounter it.

---

## 2.5 — Consolidate to three components first

From the follow-up consultation: fold the current directories into the
**three** components the split will produce, before splitting anything.

This is the right order. Consolidating inside one repository is cheap and
fully reversible; discovering afterwards that a directory went to the
wrong repository is neither.

### Where each directory goes, and why

| today | goes to | reason |
|---|---|---|
| `backend/` | **backend** | review-api |
| `mcp-server/` | **backend** | the review engine review-api drives |
| `scanner/` | **backend** | produces the findings review-api consumes |
| `frontend/` | **backend** | see below |
| `frontend-react/` | **frontend** | the dashboard |
| `gateway/` | **gateway** | unchanged |

### `frontend/` vs `frontend-react/` — settled by reading the code

They are not two versions of the same thing, and neither is dead.

**`frontend/`** is an 896-line static HTML console, and **review-api
serves it itself**: `app.py:102` passes it to Flask as `static_folder`,
`backend/Dockerfile` bakes it into the image, and `docker-compose.yml`
mounts it read-only over the baked copy so edits apply on restart.

It is not a frontend in the deployment sense — it is **part of the
backend**, the way a Django admin page is. It ships with review-api, is
served by review-api, and would break if separated from it.

**`frontend-react/`** is the Track 3 dashboard. It talks to the *gateway*,
never to review-api directly, is built and served independently, and is
what "the frontend" means in the six-repo split.

**So: `frontend/` moves under `backend/`** (as `backend/console/` or
similar) and `frontend-react/` becomes the `frontend` repo. That also
removes the standing confusion about which one people mean.

### "Two per component" — clarified, and not a separate task

The instruction that each component needs two was about **repositories,
not containers**: each component gets a code repo and a deployment repo.

That is the six-repo structure in item 2 restated — three components
times two — so there is no separate work item here, and nothing to
confirm.

Worth noting only because the phrasing could have been read as a
container requirement, which would have meant a Compose change or a
Kubernetes sidecar. It does not.

One thing the restructure does bring for free: `docker-compose.yml`
currently defines a **single** service (`api`), because it predates both
the gateway and the React dashboard. Three components means it should
define three.

---

## 3 — CI/CD per code repo

Only the three code repos; the deployment repos are ArgoCD's input, not
Jenkins'.

Minimum pipeline, in this order — cheapest and most certain first:

1. lint + type-check (`tsc --noEmit` for the frontend)
2. unit tests — **fail the build on red**
3. `security-tests/` (the prompt-injection suite)
4. Semgrep / Gitleaks / Trivy **on our own code**, with a real exit code
5. build image, tag with the commit sha
6. push to the registry (item 4)
7. update the image tag in the matching deployment repo → ArgoCD deploys

**Two things to get right at the start, because both are hard to add
later:**

- **The scanner tier gates; the AI tier does not.** An LLM is not
  deterministic: the same commit can produce different findings on two
  runs, and a build that fails randomly gets bypassed within a fortnight
  — taking the deterministic checks that *did* work down with it. The AI
  review comments; it never fails a build.
- **`repo-scan` currently returns 0 always** (`scanner/src/repo_scanner/cli.py`).
  A scanner that cannot fail a build is decoration. Needs a
  `--fail-on <severity>` before step 4 means anything.

Run any new gate in **advisory mode first** and measure its false-positive
rate before it blocks. A gate the team learns to route around is worse
than no gate, because it also produces false confidence.

---

## 4 — Where images live

| option | verdict |
|---|---|
| **GHCR** | **recommended now.** Free for our use, same account as the code, private by default, vulnerability scanning included, no cloud bill |
| Docker Hub | rate limits on anonymous pulls will bite the cluster |
| **ECR** | **the end-state**, alongside the AWS migration; needs IRSA and lifecycle policies |
| self-hosted (Harbor) | more to operate than it teaches us here |

**Recommendation: GHCR now, ECR at the end** — the same migration shape as
everything else in this phase, and `imagePullSecrets` is the only thing
that changes in the manifests.

Tag with the **commit sha**, not `latest`: a tag that moves makes a
rollback impossible to describe and a running Pod impossible to identify.

---

## 5 — Environment variables and delivering the stack

Two separate problems that get confused with each other.

### Non-secret configuration
ConfigMaps, per environment, held in the deployment repos. Straightforward.

### Secrets — the part that needs a decision

Plain `Secret` manifests are base64, **not encryption**. They cannot go in
a git repo, which breaks GitOps the moment a secret is involved.

| approach | fit |
|---|---|
| **Sealed Secrets** | **recommended for K3s.** Encrypted with the cluster's public key, so the encrypted file lives in git safely; only the controller can decrypt |
| SOPS + age | good, but needs key distribution we would have to run |
| External Secrets Operator | the right answer on AWS with Secrets Manager; too much apparatus for K3s |
| Vault | out of scope for the time we have |

**Recommendation: Sealed Secrets on K3s, External Secrets + AWS Secrets
Manager after the migration.** Both keep the rule that matters: nothing
readable is ever committed.

This connects to a live constraint — the project already keeps a local
folder out of git for exactly this reason. Sealed Secrets is what lets
that material live in the repo without being readable.

### Delivering the stack
Helm per component, in its deployment repo; ArgoCD reconciles. `helm
template` in CI catches a broken chart before the cluster does.

---

## 6 — Tailscale inside Kubernetes

**Owner: Elad.**

Tailscale currently reaches the model host from a developer machine. Inside
a cluster the shape is different, and the choice matters:

- **sidecar** — one container per Pod that needs the tailnet. Simple,
  isolated, but a key per Pod.
- **subnet router** — one Pod advertises a route on behalf of the cluster.
  Fewer keys, one dependency for everything.
- **Tailscale Operator** — the intended path for Kubernetes; more to
  learn, and closest to how this is really run.

**Things worth deciding explicitly:** whether auth keys are ephemeral
(they should be — a Pod that dies should not leave a stale node), whether
ACLs restrict the cluster to *only* the model host rather than the whole
tailnet, and what happens to a review when the tailnet is unreachable
(right now: a failed review, which is at least honest).

---

## Incremental review — scan the diff, not the repository

Not from the lecturer; came out of the same discussion, and it is what
stops run time growing with the project.

**Today every review is a full review.** The checkpoint in
`pipeline.py:206` is keyed to `repo_url + commit + mode`, so a new commit
discards it entirely and all files are reviewed again. That is correct for
what it is — a *resume* mechanism, not an incremental one.

**Why full re-review is defensible for a periodic audit:** an AI review
judges context, and a file that did not change can still be newly broken
by a change elsewhere — a signature that moved, a permission check removed
two layers up. `git diff` will not show that.

**Why diff-scoped review is nevertheless necessary:** at 3.8 hours for 123
files, per-PR review is impossible, and the cost grows with the repository
while the diff usually does not.

**Proposed: make it the user's choice, with an honest default.**

| mode | scope | for |
|---|---|---|
| **Changed files** | files in the diff vs. the base branch | PR review — the default there |
| **Full** | every file in scope | periodic audit, first run on a repo, pre-release |

Two rules that keep the fast mode from lying:

1. **A diff-scoped report must say so** — and must not present itself as a
   clean bill for the whole repository.
2. **Deterministic scanners keep running over everything.** They are cheap
   and they are the only thing that sees a dependency CVE introduced by a
   file nobody edited.

**Implementation is closer than it sounds:** `files=` is already supported
end to end, and the webhook already receives the commit. What is missing
is computing the changed-file list from the diff and passing it through.

---

## Sequencing — revised for the 14 October deadline

**This replaces an earlier ordering that put the repo split last.** That
was the right call for a project with open-ended time and it is the wrong
one here, for a mechanical reason:

> **CI built before the split is CI thrown away.** One monorepo pipeline
> becomes six repo pipelines. Building it twice is work this schedule
> cannot absorb, and the second build would land in the last fortnight.

The concern that argued for splitting late — cross-component changes
becoming painful — also weighs less now than it did. Almost everything
remaining is platform work that is per-repo anyway; the dense
cross-component application work is behind us.

**So: consolidate and split early, then build everything once, in its
final shape.**

| # | work | owner | why here |
|---|---|---|---|
| 1 | `--fail-on` in `repo-scan` | Sahar | item 3 is decoration without it; small, independent |
| 2 | K3s up, hand-written manifests applied | Sahar | course pillar, and proves the manifests before Helm hides them |
| 3 | Freeze + version the gateway ↔ review-api contract | Sahar | **must precede the split** — after it, drift breaks silently |
| 4 | **Consolidate to three components (2.5)** | — | cheap and reversible now; expensive after the split |
| 5 | **Split into six repos** | TBD Sunday | the pivot — everything after is built in final shape |
| 6 | Deployment repos wired to ArgoCD | — | pure addition once the split exists |
| 7 | Jenkins + one pipeline end to end | — | prove the shape on one repo before three |
| 8 | The other two pipelines | — | mechanical once the first works |
| 9 | Secrets (Sealed Secrets) | — | needed before anything real is deployed |
| 10 | Admin config page | Sahar | independent; parallel throughout |
| 11 | Diff-scoped review | — | first cut if time is short |
| 12 | AWS / EKS / ECR | — | the finale, and the first thing to drop |

**Items 1–3 need nobody else** and are what Sahar can carry before the
Sunday meeting. **Item 5 is the pivot** — it should be assigned first on
Sunday, because 6–9 all wait on it.

---

## Open questions

- **Who owns the Jenkins host** and its credentials?
- **Does a shared contract package earn its place?** Six repos means the
  gateway's view of review-api's API lives in the gateway repo. A small
  versioned package (or just a schema file both repos vendor) is the
  usual answer; worth deciding before the split rather than after the
  first silent break.
- **Does the admin page need per-user roles**, or is one operator list
  enough for the scope of this project?
- **What happens to a review when the model is unreachable** in the K3s
  deployment — fail, queue, or fall back to a second provider? The
  provider selector makes the third answer possible for the first time.
