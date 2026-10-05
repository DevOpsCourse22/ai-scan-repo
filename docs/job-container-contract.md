# Scanner+AI Job Container Contract

**Status:** Agreed — v2, 2026-09-26 (Sahar and Elad). v1 was the draft
agreed between Sahar and Claude; v2 adds the six changes from Elad's review
(see `docs/WORK-TRACKER.md`, "Contract changes"). **v3 proposed 2026-10-01**
(branch `elad/ui-redesign-pause-resume`, for Pause/Resume): an optional
`COMMIT` input, and phase/counts/commit in `.progress.json`. Both are
additions only: a v2 backend ignores the new fields, and a v3 backend
works with a v2 image except that resume cannot be pinned.
**v4 proposed 2026-10-01** (same branch, for the AI model setting): an
optional `LLM_API_KEY` input, from the per-attempt Secret, and
`LMSTUDIO_BASE_URL` may now be a cloud service's OpenAI-compatible address
(chosen by name in Settings, fixed in review-api's `llm_presets.py`). A v3
image ignores the key, so a cloud service cannot authenticate with it.
**Purpose:** the shared interface the `backend` orchestration code and the
Kubernetes Job manifest are both built against, so neither side has to
guess at the other's shape. See `SCOPE-DECISIONS.md` (2026-09-19 entry) for
the decisions this contract implements: combined scanner+mcp-server image,
polling-based reconciliation with a 5-attempt retry cap, PVC scoped to
reports and checkpoints only.

This document defines one container image's behavior: what it reads, what
it writes, and what its exit code means. It does not define the Kubernetes
manifest itself (Job spec, PVC, Secret, NetworkPolicy).

**Implemented by:** `Code-Review-MCP/scanner-worker/` (the image) and
`Code-Review-MCP/backend/src/review_api/k8s_jobs.py` (the side that creates
and polls the Jobs).

## Inputs (environment)

| Name | Required | Description |
|---|---|---|
| `REPO_URL` | yes | Repository to review, `https://github.com/<owner>/<repo>` |
| `REVIEW_ID` | yes | Unique review identifier and the key of every output file. **Must stay identical across a failed attempt and its retry** — the retry finds the previous checkpoint by this value, so a new ID on retry would silently discard resumable progress. A plain file name: letters, digits, `_ . -`, max 128, no leading dot. It is NOT a Kubernetes name — the backend derives the Job name from it (lowercased, shortened, hashed) |
| `MODE` | yes | `quick` \| `brief` \| `deep` |
| `LMSTUDIO_BASE_URL` | yes | The AI endpoint. In the cluster: `http://lmstudio:1234/v1` (the Tailscale egress Service) |
| `SYSTEM_PROMPT` | yes | Reviewer system prompt |
| `BRANCH` | no | Defaults to the repository's default branch |
| `LMSTUDIO_MODEL` | no | The model to use, as LM Studio names it. **(v2)** Without it the user's model choice is lost and LM Studio answers with whatever is loaded — or loads a second model next to the first on the same GPU |
| `COMMIT` | no | **(v3)** A full 40-character commit SHA. The clone is moved to this exact commit instead of the branch tip. review-api sets it when it **resumes** a paused or interrupted review, with the commit from the checkpoint: a checkpoint is only reused when the commit matches, so without the pin a branch that moved on while the review was paused would restart it from file 1. Anything that is not a full SHA is a permanent failure (exit `2`); a commit that can no longer be fetched (force-pushed away) fails the clone |
| `GITHUB_TOKEN` | no | Only from a Kubernetes Secret (`secretKeyRef`) — never a literal in the Job spec, a command-line argument, or a log line. See "The token" below |
| `GITHUB_TOKEN_FILE` | no | Alternative to `GITHUB_TOKEN`: path to a Secret mounted as a file |
| `LLM_API_KEY` | no | **(v4)** The API key of the AI service chosen in Settings, only from the per-attempt Secret (key `llm_api_key`). Removed from the environment before the engine loads and scrubbed from every log line, like the token. Absent: the engine's server default (`LMSTUDIO_API_KEY`) applies |

## Outputs (on the reports PVC, mounted at `/reports`)

| Path | Written | Contents |
|---|---|---|
| `<REVIEW_ID>.json` | at success | Final report (machine-readable). **Its existence is how the backend recognises success** |
| `<REVIEW_ID>.md` | at success | Final report (human-readable) |
| `<REVIEW_ID>.findings.json` | after the scan | Deterministic scanner output (Semgrep+Gitleaks+Trivy), the input to AI enrichment |
| `<REVIEW_ID>.plan.json` | before the review | Every file the review set out to cover: paths, languages, sizes, line counts. No content. Kept after success |
| `<REVIEW_ID>.partial.json` | per unit (brief, deep) | Resume checkpoint. Same path on every attempt, so a retry (new Pod, same review) resumes. Deleted at success |
| `<REVIEW_ID>.error.json` | at failure | **(v2)** Why this attempt failed — see below |
| `<REVIEW_ID>.progress.json` | per file | **(v2)** Live per-file timing — see below |

All JSON files are written **atomically** (temp file + rename): a reader
never sees a half-written file.

`.error.json`, `.progress.json`, `.partial.json` and `.plan.json` are
sidecars, never reports: the API's report listing and the UI must not show
them, and deleting a report deletes its sidecars.

**Not on the PVC:** the cloned repository. It lives in a short-lived
`emptyDir`, scoped to one Pod and gone when the Pod is removed — per the
ephemeral-processing principle (ADR-001).

### `<REVIEW_ID>.error.json` (v2)

```json
{"permanent": true, "error": "permanent: IngestError: git clone failed: ... Remote branch nope not found ..."}
```

- Written on **every failure the worker detects itself**, with the same one
  line it writes to `/dev/termination-log`. `permanent` is `true` exactly
  when the exit code is `2`.
- **Deleted by the worker when an attempt starts**, and again at success, so
  a file on the PVC always belongs to the latest attempt. The backend also
  deletes it before launching an attempt.
- How the backend reads a failed Job:
  - `permanent: true` → stop now; the review is `failed` with this error.
  - `permanent: false` → retry (new Job, same `REVIEW_ID`), up to 5 attempts.
  - **no file** → the worker could not write one (OOM kill, node lost,
    deadline) → treat as retryable.
- Why a file and not the Pod's termination message: reading Pod status
  needs `list` on pods (a Job's Pod has a generated name), which RBAC cannot
  narrow to our Jobs' Pods — and every Pod carries its Job's env
  (`REPO_URL`, `SYSTEM_PROMPT`). The file keeps review-api's RBAC at `get` on
  `jobs/status`. The termination log is still written, for
  `kubectl describe pod`.

### `<REVIEW_ID>.progress.json` (v2)

```json
{
  "current_file": "src/app.py",
  "current_started_at": 1758830000.12,
  "durations": [["src/a.py", 41.2], ["src/b.py", 188.0]],
  "phase": "reviewing",
  "done": 12,
  "total": 70,
  "reused": 72,
  "commit": "5e4e7d5…(40 hex)",
  "eta_seconds": 5400,
  "updated_at": 1758830012.0
}
```

- **(v3)** `phase` is the pipeline's last status (`starting`, `cloning`,
  `scanning`, `reviewing`). `done`/`total` are the pipeline's unit counts
  (files in deep mode, batches in brief), `null` until the first progress
  callback. After a resume they count the **remaining** work only, and
  `reused` holds what the checkpoint gave back — the same convention as
  the in-process status. `commit` is the commit being reviewed, `null`
  before the clone. In kubernetes mode this file is the only source
  review-api has for any of these, so the UI's progress bar and the commit
  a later resume pins both come from here.

- Updated when a file **starts** and when it **finishes**, and in between
  **at most once a minute** while the review shows any sign of life (a
  status change, streamed tokens) — in every mode. Written fresh (empty
  `durations`, no `current_file`) when an attempt starts; deleted at
  success, together with the checkpoint.
- **`updated_at` is the stuck signal** (decided 2026-09-27): the backend
  treats an attempt as stuck when `updated_at` is older than
  `REVIEW_K8S_STALE_SECONDS` (default 15 minutes), counted from the later
  of the attempt's launch and the last write. It deletes that Job (the
  worker's SIGTERM path keeps the checkpoint) and retries it like any other
  failed attempt. `activeDeadlineSeconds` stays only as a backstop
  (`REVIEW_K8S_JOB_DEADLINE_SECONDS`, default 12 hours): wall-clock time
  says nothing about stuck, and a 30-minute limit x 5 attempts killed any
  review longer than ~2.5 hours while it was still progressing.
- **The rule that keeps this honest: no phase may be silent for longer than
  the stale limit unless it is bounded by its own timeout.** The clone
  (300 s, git killed) and the scanners (900 s *each*, run one after another
  — up to 45 minutes together) report no progress of their own, so the
  worker keeps `updated_at` moving through the `cloning` and `scanning`
  phases with a ticker, once a minute. Without it, a large repository whose
  scan legitimately ran past 15 minutes was killed as stuck on every
  attempt and could never reach the AI stage. The ticker stops the moment
  the review leaves those phases: in the AI phase only real signs of life
  (streamed tokens, file starts and finishes) may refresh the file, so a
  model that stops answering still goes stale and is caught. Adding a new
  silent step means either giving it a timeout and adding its phase to
  `BOUNDED_SILENT_PHASES` in `scanner-worker/entrypoint.py`, or making it
  report progress.
- `current_file` is the file most recently started (`""` before the first);
  it may already have finished — readers check it against the checkpoint,
  exactly as the in-process view does. `current_started_at` is `null` when
  `current_file` is `""`.
- Timestamps are **wall-clock epoch seconds**: a monotonic clock does not
  carry between Pods. (One node on K3s; NTP-synced nodes on EKS.)
- `durations`: files finished **in this attempt** only, each timed from its
  own start. A retry starts the list fresh, so files restored from the
  checkpoint never enter the estimate.
- Paths and seconds only — no content, no findings.
- Per-file events exist in **deep** mode only. In brief (batches) and quick
  (one call) modes the file carries no per-file timing, the same limit the
  in-process view has — but its `updated_at` still moves.

## Internal sequence (one process, in order)

1. Remove the previous attempt's `.error.json`; write a fresh
   `.progress.json`.
2. Clone the repository (using the token if supplied) into `/workspace` —
   never the PVC.
3. Run the deterministic scanner tier over the clone; write
   `<REVIEW_ID>.findings.json`.
4. Run AI enrichment with those findings, checkpointing as it goes; write
   `<REVIEW_ID>.json` and `<REVIEW_ID>.md`.
5. Exit.

## Exit codes

| Code | Meaning |
|---|---|
| `0` | The review pipeline ran to completion. This includes runs that found real issues — findings are a successful outcome, not a failure |
| `1` | A technical failure prevented completion and **a retry may succeed** (LLM unreachable, network or DNS error, git timeout, SIGTERM, unhandled exception) — distinct from "the code being reviewed has problems" |
| `2` | **(v2)** A permanent failure **no retry can fix**: missing or invalid input, not a GitHub URL, repository not found or no access (bad or missing token), branch not found, no reviewable files, reports/workspace volume not writable |

On any non-zero exit the reason is written as one line to
`/dev/termination-log` and to `<REVIEW_ID>.error.json`. Unclear failures
default to `1`: the backend's 5-attempt cap bounds the cost of a wrong guess
in that direction.

## The container's environment (v2)

The Job runs with a **read-only root filesystem** as uid 10001, so
everything that writes is pointed at a mount:

| Path | Volume | Holds |
|---|---|---|
| `/workspace` | `emptyDir` (2Gi) | `TMPDIR` — the clone lands here; `HOME`; Semgrep's `~/.semgrep`; Trivy's DB (`TRIVY_CACHE_DIR`) |
| `/tmp` | `emptyDir` | a backstop for anything ignoring `TMPDIR` |
| `/reports` | the PVC | the outputs above, nothing else |

- `TMPDIR=/workspace`: the engine clones through `tempfile`, so without it
  the clone would land in `/tmp` while `/workspace` sat unused.
- git's `safe.directory` is set in the **system** config
  (`/etc/gitconfig`): with `HOME` on an emptyDir a `--global` setting does
  not exist at run time, and without it Gitleaks finds nothing, with no
  error.
- **The scanners need the internet on every run.** Semgrep downloads its
  rules and Trivy its vulnerability database; nothing is cached between
  Pods. If they cannot, they are recorded under `scanners_failed` in
  `.findings.json` and the review continues AI-only — which the success
  definition (point 5) forbids. **Before a NetworkPolicy exists, bake the
  rules and the DB into the image**; until then the Job needs egress to
  GitHub, the LLM endpoint, `mirror.gcr.io`/`ghcr.io` (Trivy) and
  `semgrep.dev`.

## The token (v2)

The worker removes `GITHUB_TOKEN` (and `GH_TOKEN`) from its environment
**before** the review engine is loaded, so no child process — git, Semgrep,
Trivy, Gitleaks — inherits it. It is handed to the engine as an argument
and reaches git only through `GIT_ASKPASS`, so it appears in no command
line, URL or file. Every log line and error message is scrubbed of it.

v1 said the token is "dropped from memory immediately after clone
completes". That cannot be honoured literally: the engine holds its
argument until the review returns. What the contract guarantees is the
above — no inheritance, no logging, no persistence. Delivering it as a
mounted file (`GITHUB_TOKEN_FILE`) keeps it out of the Pod's env as well.

The AI service key (v4) follows the same rules: taken out of the
environment first, handed to the engine's client only, scrubbed from
every log line, the termination log and `.error.json`.

## What this contract does not decide

- The Job manifest itself (resource limits, security context, RBAC,
  network policy, PVC size/class) — `k8s/` and `k8s_jobs.build_job_manifest`.
- How review-api turns `.progress.json` into the progress view — the
  reading side of `GET /api/v1/reviews/<id>/files`.
