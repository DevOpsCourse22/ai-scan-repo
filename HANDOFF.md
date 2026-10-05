# HANDOFF — code-review-mcp

> **Current project state is in the root `HANDOFF.md`** (and `CLAUDE.md`).
> This file is the older, engine-focused history and is kept for reference.

> Purpose: seed a **new conversation** with the complete context of this
> project so work continues without re-deriving anything. Read this first.
> Last updated: 2026-09-17, after the **console became its own microservice**
> (review-api is API-only; see history item 24 and docs/21). Before that:
> 2026-07-30, after the **three-component split**
> (`mcp-server/` + `backend/` + `frontend/`) and the reproducibility work
> that goes with it: docker compose, setup scripts, `.env`, `${VAR}` config
> interpolation, a Google AI Studio cloud provider, and
> [docs/18-team-setup-guide.md](../docs/18-team-setup-guide.md).

---

## 1. What this project is

**`code-review-mcp`** — a Python **MCP server** that performs AI code review
of GitHub repositories using a **local LLM (Gemma) served by LM Studio**,
packaged as a portable **Docker** container. It is the user's **final
project** and is being adapted into the **AI Scan Repos** platform per
[docs/14-mcp-adaptation-plan.md](../docs/14-mcp-adaptation-plan.md).

- **Location:** `C:\Projects\Final-Project` (Windows 11; PowerShell primary,
  Bash tool available).
- **Now a git repository** (initialized 2026-07-23 per adaptation plan WS0).
  Disaster-recovery checkpoints: commit `DR-0` = the full Semgrep-hybrid
  state, also tagged **`semgrep-hybrid`**; `DR-1` = the rollback. Recover
  the hybrid any time with `git checkout semgrep-hybrid`.
- **The user is not a Python expert** — explain changes clearly. They drive
  everything through natural-language prompts to Gemma in LM Studio's chat,
  not by reading code. They frequently hot-swap LM Studio models.

### The core idea (non-obvious, explain-first)

**Gemma plays two roles simultaneously:**
1. **MCP host** — in LM Studio's chat, Gemma calls the server's tools.
2. **Reviewer** — the server calls back over HTTP to LM Studio's
   OpenAI-compatible endpoint (`http://localhost:1234/v1`); that model (same
   or another, via the `model` arg) reads the code and writes the findings.

The **server contains no AI** — it is orchestration only: clone, chunk,
call LLM, enforce JSON, merge, checkpoint, write reports. Wall time ≈
number of LLM calls × model speed; the server adds a few seconds.

### The Semgrep rollback (this release)

The previous release bundled a Semgrep static pass ("hybrid reviews").
Per the adaptation plan (inversion I1: the **platform** owns deterministic
scanning; scanners become pinned plugins in the analysis plane), Semgrep
was **removed from the MCP** on 2026-07-23. This MCP is now the clean
**LLM-only** vertical slice / local provider path. The full hybrid
implementation lives under the git tag `semgrep-hybrid` and will be
re-homed as a platform scanner plugin (WS1), NOT re-added here.

---

## 2. Architecture & flow

```
LM Studio (Gemma = host) --stdio--> docker run --rm -i code-review-mcp
        ^                                   |
        | HTTP (OpenAI API, Gemma=reviewer) |
        +-----------------------------------+
   1. git clone --depth 1  -> /tmp inside container (deleted after)
   2. repo -> JSON manifest {repo_url, commit, files:[{path,language,content}], skipped}
   3. review (quick | brief | deep)  [+ per-unit checkpoint in brief & deep]
   4. merge -> report
   5. write <name>.json + <name>.md -> /reports volume -> host reports\ folder
```

- Python 3.10+ (`mcp` FastMCP, stdio transport, `openai` SDK client).
- Docker image `code-review-mcp:latest` (`python:3.12-slim` + git, ~412 MB).
- Container→host LM Studio via `host.docker.internal`
  (`--add-host=host.docker.internal:host-gateway` for Linux parity).
- Reports: `/reports` (container) = `-v` mount of
  `C:\Projects\Final-Project\reports` (host).

---

## 2b. Repository layout (three components, one engine)

```
Final-Project/
├─ mcp-server/            component 1 — MCP front end + THE SHARED PIPELINE
│   ├─ pyproject.toml     package `code-review-mcp`  (mcp<2 pin — see §9)
│   ├─ Dockerfile         -> image code-review-mcp:latest (context: this dir)
│   ├─ PROMPTS.md, example_prompt.txt, README.md
│   ├─ src/code_review_mcp/
│   └─ tests/             91 tests + fixtures/dummy_repo + mock_llm_server.py
│                         + e2e_review.py
├─ backend/               component 2 — review-api, API ONLY (no UI)
│   ├─ pyproject.toml     package `review-api`, DEPENDS ON code-review-mcp
│   ├─ Dockerfile         -> image review-api (context: THIS folder, plus
│   │                        --build-context mcp-server=../mcp-server)
│   ├─ build.sh/.ps1      build [--registry R --tag T --push]
│   ├─ config/api.example.yaml   (api.yaml is gitignored)
│   ├─ src/review_api/
│   └─ tests/             + conftest.py (adds mcp-server/tests to path)
├─ frontend/              component 3 — review-console: index.html behind nginx
│   ├─ Dockerfile         -> image review-console (nginx-unprivileged, :8080)
│   ├─ nginx/             default.conf.template + 05-review-api-url.envsh
│   ├─ build.sh/.ps1
│   └─ tests/             page + proxy + image tests
├─ scanner-worker/        the Kubernetes Job image: scanner + engine in one,
│   ├─ Dockerfile         -> image scanner-worker (context: this dir, plus
│   │                        --build-context mcp-server=… and scanner=…)
│   ├─ entrypoint.py      env -> run_review() -> exit 0 / 1 retry / 2 permanent
│   ├─ build.sh/.ps1
│   └─ tests/             entrypoint + image tests (opt-in: built image)
├─ docs/                  all documentation (01–21)
├─ scripts/               setup / run-api / test / run-mock-llm  (.ps1 + .sh)
├─ docker-compose.yml     services `api` (:8000) and `console` (:8090);
│                         MCP is stdio, not a service
├─ .env.example           THE only per-machine file (gitignored as .env)
├─ .gitattributes         LF for *.sh/Dockerfile/yaml, CRLF for *.ps1
└─ reports/               shared output: <id>.json/.md/.partial.json
```

**Install is two editable packages in ONE command** (`code-review-mcp` is not
on PyPI, so installing `backend` alone makes pip look there and fail):

```powershell
pip install -e "./mcp-server[dev]" -e "./backend[dev]"
```

Why three folders and not two: the user asked for the backend and the POC
frontend to be separable, and `pipeline.py` is shared — so `mcp-server/` owns
the engine and `backend/` depends on it, rather than duplicating recovery
logic. The frontend is its own **service** (since 2026-09-17): the final
stage deploys review-api alone, and the console is a debugging add-on that
proxies `/api/` to it. **No Kubernetes manifests ship in the repo on
purpose** — the user is writing them as course work; docs/21 is the guide.

---

## 3. Source files (`mcp-server/src/code_review_mcp/`)

| File | Responsibility |
|---|---|
| `config.py` | All settings (env-overridable), extension allowlist, skip-dirs, ext→language map. |
| `findings.py` | Deterministic-findings intake (the platform hand-off): loads/normalizes an external scanner-findings JSON (`load_findings_file`, raises `FindingsError`), matches to the manifest (`match_to_manifest`, unmatched reported not dropped). NO scanner here. |
| `repo_ingest.py` (auth) | PRIVATE repos: `_auth_context` (x-access-token username in URL + `GIT_ASKPASS` env), `_write_askpass` (helper holds no secret), `_redact`, `_GIT_NO_HELPER` (disables GCM), `_access_hint`, `check_access` (ls-remote). Per-review `token=` overrides the env token (HTTP API). |
| **`pipeline.py`** | **THE shared review runner** — `run_review()` owns findings intake, clone, file filter, checkpoint discovery/resume, progress+ETA, review, fold-in, report+md writing. Callbacks `on_status`/`on_progress`/`should_cancel`. Also `_resolve_output`, `_match_files`, `ReviewCancelled`. Both front ends call it, so crash recovery exists ONCE. |

**Second front end — `backend/src/review_api/` (Flask HTTP control plane):**

| File | Responsibility |
|---|---|
| `config.py` | The config file (YAML/JSON) + `REVIEW_API_*` env overrides; **`${VAR}`/`${VAR:-default}` interpolation** (unset + no default = `ConfigError` naming the key); `_coerce_scalars` restores declared int/bool types that interpolation flattened to strings; **relative `output_dir` resolves against the CONFIG FILE's dir**, not the cwd; `_RETIRED_KEYS` (`frontend_dir` is dropped with a warning, not rejected); `_getenv()` ignores Kubernetes service-link values (`tcp://…`) on BOTH the `REVIEW_API_*` override path and `${VAR}` expansion; `load_dotenv()` (called from `main()` ONLY — never at import, or tests would absorb a local .env); `Provider` registry (cloud + on-prem); API keys come from `api_key_env` (never stored in the file); validates at startup. |
| `prompts.py` | Prompt consolidation: reviewer identity + review type + focus + enrichment duty + standing safety/output rules; `PROMPT_TEMPLATE_VERSION`. |
| `jobs.py` | `JobStore`/`Job`: lifecycle (`requested→cloning→reviewing→completed/failed/cancelled`), progress+ETA, restart (resume), cancel, one-at-a-time run lock, per-provider settings swap. **Tokens live in `_tokens` keyed by job id and are deleted in `finally` — never on the Job record.** |
| `app.py` | REST v1 routes, `X-API-Key` gate, CORS, RFC 9457 problems, `main()` (loads `.env`, then the config). **No UI:** `static_folder=None`; `/` is a JSON descriptor; `/healthz` (liveness, checks nothing external) and `/readyz` (reports dir AND temp dir writable) are exempt from the key. |

*(The console lives in `frontend/` as the separate `review-console` service.)*

**Back in `code_review_mcp/` — the rest of the pipeline:**

| File | Responsibility |
|---|---|
| `repo_ingest.py` | Clone + JSON manifest. Windows git hardening `_run_git`/`_kill_tree` (stdin=NUL, file-redirected output, prompts off, tree-kill on timeout). |
| `surface.py` | Quick mode's regex static scanner (secrets, eval/exec, pickle, shell=True, verify=False, weak RNG, bare except, asserts, TODO) + heuristic score. No LLM. |
| `llm.py` | Streaming LM Studio client; JSON enforcement (deep + brief schemas); invalid/empty-JSON retries incl. token-budget doubling; `resolve_model()`; fatal (`LLMUnavailableError`) vs per-file errors. |
| `review.py` | Orchestration: `review_manifest` (3 modes), chunking, `pack_digest_batches`, `_quick_report`/`_brief_report`, merge, `render_markdown`, `checkpoint_cb` (deep) + `brief_state` (brief), **serial mode runs INLINE (no thread pool)**. |
| `server.py` | FastMCP server: all tools, job registry, `files` filter, output resolution, progress payload, checkpoint/resume logic (deep files + brief batches), `_find_checkpoints`. |
| `cli.py` | Standalone CLI (no MCP host) for testing. |

Root: `.mcp.json` (venv + docker entries), `docker-compose.yml`,
`.env.example`, `.dockerignore` (for the root-context backend build),
`.gitattributes`, `README.md`, `HANDOFF.md`, `scripts/`.
Docs: `docs/` is the platform documentation tree (01–13 + HLD chapters +
ADRs). Implementation-specific: **`docs/14-mcp-adaptation-plan.md`** (the
split plan), **`docs/15-mcp-server-documentation.md`** (full MCP reference),
**`docs/16-gemma-user-guide.md`** (Gemma 4 26B user guide),
**`docs/17-review-api.md`** (Flask API), **`docs/18-team-setup-guide.md`**
(getting it running on any machine — the doc to send a teammate).
Tests (**128 total, all passing**): `mcp-server/tests/` = `test_pipeline.py`,
`test_dummy_repo.py`, `test_findings_intake.py`, `test_private_repo.py` (91),
plus `fixtures/dummy_repo/` (planted-issue fixture),
`mock_llm_server.py` (stdlib fake LM Studio: `--delay`, `--stall-after`) and
`e2e_review.py` (real-pipeline e2e driver for the kill-test drill;
`--findings-file`). `backend/tests/` = `test_review_api.py` (37) +
`conftest.py`, which puts `mcp-server/tests` on `sys.path` so the API tests
reuse the dummy-repo fixture instead of keeping a second copy.

---

## 4. The MCP tools

- **`start_review(repo_url, system_prompt, output_path="review-report.json", branch="", extensions=None, mode="brief", model="", review_id="", files=None, findings_file="")`**
  → `job_id` instantly; one job at a time. `review_id="X"` names outputs
  `X.json`+`X.md`. **Self-healing:** same review_id+mode again = RESUME from
  checkpoint (commit-verified; stale commit → fresh run).
  `findings_file="scan.json"` (resolved against the reports dir) feeds an
  EXTERNAL scanner's findings into the review: deep injects them into
  prompts (verify/enrich/dismiss + untrusted-data note), brief packs
  `static_flags`, quick merges into issues; report attaches them verbatim
  as `static_findings` (+`confirmed_by_llm` ±2 lines) + top-level
  `deterministic` meta (incl. `unmatched`); missing/invalid file = job
  ERROR (honest degradation), zero findings = fine.
- **`get_review_status(job_id, wait_seconds=40)`** — long-polls server-side;
  compact `{status, done, total, remaining, percent, elapsed_seconds,
  eta_seconds, progress, do}`. `done` counts completed units only; ETA stable.
  `do` = anti-hallucination contract. Unknown job → lists
  `resumable_checkpoints` + hint.
- **`get_report(report, file="", full=False)`** — summary / one file / full
  JSON, by job_id or filename. Results retrievable through MCP even with no
  volume.
- **`get_report_entry(report, index=0)`** — one file per call (context-safe).
- **`append_report_notes(report, text)` / `read_report_notes(report, offset=0)`**
  — persistent `<name>.notes.md` scratchpad for iterate→render→summarize.
- **`list_reports()`** — reports + `resumable_checkpoints`.
- **`check_repo_access(repo_url, branch="")`** — `git ls-remote` (~1s, no
  clone/LLM): can the server reach this repo? Reports token PRESENCE only.
  Run before long private-repo reviews.
- **`check_llm()`** — endpoint probe; returns `busy` instantly while a job runs.
- **`ingest_repository_only(...)`** — clone→manifest only, no LLM.
- **`review_repository(...)`** — blocking; **HIDDEN unless
  `REVIEW_ENABLE_BLOCKING=1`** (small models pick it by name → -32001).

---

## 5. The three modes

| Mode | LLM calls | Coverage | 123-file repo | Use |
|---|---|---|---|---|
| **quick** | 1 | all files (regex surface scan + AI overview) | ~30–60s | instant impression |
| **brief** (default) | batches+1 (~10) | all files, head of each, batched | ~5 min (12b) / ~10–15 min (26b) | full-repo triage |
| **deep** | 1/file | full detail, checkpointed | hours (whole repo); ~1–5 min/file on 26b | targeted with `files=` |

`files=["*/views.py", "exact/path.py", "admin.py"]` — full path OR basename,
globs OK. Workflow: quick/brief triage → deep on flagged files.

---

## 6. Report shape (JSON)

```
{
  "repo_url", "branch", "commit", "model", "generated_at", "mode",
  "resumed": true, "reused_batches"/"reused_reviews": N,   # only after a resume
  "stats": {files_reviewed, files_failed, files_skipped, files_excluded,
            issues_total, issues_by_severity{critical..info}, average_score,
            static_findings: N},                            # if findings_file
  "deterministic": {enabled, source, file, findings_total,
                    findings_by_severity, unmatched?},      # if findings_file
  "overall": {summary, score, key_issues[], batches, ...}, # brief/quick
  "reviews": [ { file, summary, issues[], positives[], score,
                 static_findings: [{source, rule, severity, line, cwe?,
                   description, confirmed_by_llm}] } ],     # if findings_file
  "failures": [{file, error}], "skipped": [{path, reason}],
  "excluded": {count, by_type{".png": 10, ...}, examples[]}  # not a reviewed type
}
```
`skipped` is a reviewable file the walk gave up on (too large, not UTF-8);
`excluded` is a file whose type is not reviewed at all. Both are reported so
a report can never imply coverage it did not have.
`<name>.md` is the human-readable render (verdict, "## Key issues",
per-file sections).

---

## 7. Config defaults (env vars, `-e` on docker run)

`LMSTUDIO_BASE_URL` (image default `http://host.docker.internal:1234/v1`),
`LMSTUDIO_MODEL` (empty=first loaded chat model via /v1/models),
`LMSTUDIO_API_KEY`=lm-studio, `REVIEW_OUTPUT_DIR` (image: /reports),
`REVIEW_MAX_CHUNK_CHARS`=48000, `REVIEW_MAX_FILE_BYTES`=512000,
`REVIEW_MAX_FILES`=500, `REVIEW_TEMPERATURE`=0.2, `REVIEW_MAX_TOKENS`=4096,
`REVIEW_BRIEF_MAX_TOKENS`=3000, `REVIEW_BRIEF_PACK_CHARS`=12000,
`REVIEW_BRIEF_FILE_CAP`=1200, `REVIEW_TRIVIAL_FILE_CHARS`=50,
`REVIEW_SKIP_GENERATED`=1 (0 reviews minified/generated files anyway),
`REVIEW_RUN_SCANNER`=1 (0 skips the deterministic scanner tier),
`REVIEW_JSON_RETRIES`=2, `REVIEW_STALL_TIMEOUT`=300 (scaled × parallel),
`REVIEW_HEARTBEAT_SECONDS`=3, **`REVIEW_PARALLEL`=1 (serial default)**,
`REVIEW_ENABLE_BLOCKING` unset, **`GITHUB_TOKEN`/`GH_TOKEN`** (empty =
anonymous; needed for private repos — env only, never an argument).

---

## 8. Chronological history

1. **Initial build** — FastMCP server: clone → JSON manifest → per-file LLM
   review → merged report; strict pure-JSON responses; CLI for testing.
2. **First real-Gemma test** — Gemma is a reasoning model: hidden
   `reasoning_content` eats token budgets; report records actual model used.
3. **`-32001` #1: git deadlock** — via py-spy: Git-for-Windows hangs inside
   MCP servers (console-less, pipe stdio); subprocess timeout killed only the
   wrapper, orphan git.exe held pipes. Fix: `_run_git` (stdin=DEVNULL,
   file-redirected output, prompts off, `taskkill /T`); async tools.
4. **Docker + liveness goal** — containerized; streaming + stall watchdog
   (no total timeouts, error only on stall/unreachable); stdlib mock LLM.
5. **`-32001` #2: LM Studio's 60s tool cap** — job API (`start_review` +
   `get_review_status`). Proved with real Gemma that a visible blocking
   `review_repository` gets picked by name 3/3 → hid it behind
   `REVIEW_ENABLE_BLOCKING`; `check_llm` made busy-aware.
6. **Report persistence** — `get_report`/`list_reports` (results via MCP);
   Windows-path sanitization in the Linux container.
7. **Context-safe consumption** — long-poll status; `get_report_entry` +
   notes tools (Gemma-driven workflow measured at ~2.5 KB context total).
8. **Brief evolution** — one-shot packed call (29s) → user wanted ALL files →
   batched full coverage (`pack_digest_batches`, every non-trivial file's
   head in exactly one batch + merge call). 123 files → 9 batches.
9. **Dual output + review_id** — `<id>.json` + `<id>.md` (server-rendered
   markdown with severity icons), sanitized ids.
10. **Quick mode + files filter** — static surface scan of all files + one
    LLM overview; `files=` targeting (verified: deep on 1 file, 2 criticals).
11. **Parallelism** — `REVIEW_PARALLEL` thread pool, 2.39× measured vs
    threaded mock; user's serial LM Studio returned EMPTY responses under
    4 concurrent → default rolled back to 1; empty-response budget-doubling.
12. **Full real-hardware verification** — quick 25s / brief 317s /
    deep(files) 162s on gemma-4-12b-qat, all valid JSON+MD.
13. **Progress reporting** — done/remaining/percent/elapsed/eta + printable
    `progress` line each poll; done = completed-only; stable ETA; verified
    live on e4b (SLOW: ~28 min full brief vs 12b ~5 min).
14. **Docs** — full EN+HE documentation; first HANDOFF.
15. **Self-healing (deep)** — after user's 2h deep review crashed losing all
    progress AND Gemma hallucinated "I will continue to monitor" while not
    calling tools (LM Studio consecutive-tool-call cap): per-FILE checkpoints
    (`<id>.partial.json`, atomic), resume via same review_id (commit-verified,
    `resumed`/`reused_reviews`), non-daemon job thread (container finishes
    after host disconnect), **serial mode inline** (Python forbids new threads
    during interpreter shutdown), checkpoint discovery in unknown-job status +
    list_reports, anti-hallucination `do` contract. Verified: docker kill at
    2/7 → resume reused 2; instant disconnect → container finished 7/7 alone.
16. **PROMPTS.md** — paste-ready prompt library: 5 scan levels × 6 reviewer
    personas + utility/recovery prompts + combined workflows.
17. **Semgrep hybrid** — every review started with a Semgrep pass; findings
    fed prompts and reports (`static_findings`, `confirmed_by_llm`); image
    baked 5 offline rule packs. Live-verified on gemma-4-26b-a4b-qat.
    (Removed in 19 — preserved under tag `semgrep-hybrid`.)
18. **Brief checkpoint/resume** — WSL-VM OOM kill (exit 137) during a brief
    review exposed that brief had no checkpointing. Fix: checkpoint after
    every completed BATCH (`batch_results` in `<id>.partial.json`, atomic);
    resume repacks batches deterministically from the same commit and reuses
    stored results unless the batch count changed. KILL-TEST PROVEN.
19. **Semgrep rollback + platform adaptation kickoff (2026-07-23)** —
    per `docs/14-mcp-adaptation-plan.md` (deterministic scanning belongs to
    the platform, not the MCP): `git init` with DR checkpoints (`DR-0` =
    hybrid baseline, **tag `semgrep-hybrid`**; `DR-1` = rollback), deleted
    `semgrep_scan.py` and ALL hybrid wiring (config/llm/review/server/
    Dockerfile — image 816→412 MB), **checkpoint/resume kept fully intact**.
    Tests: removed 6 semgrep tests, added `tests/fixtures/dummy_repo`
    (planted-issue fixture) + `tests/test_dummy_repo.py` (incl. full
    crash→checkpoint→resume DR cycles for deep AND brief) → **63 passing**.
    Added `tests/e2e_review.py` driver; live kill-test re-proven on the
    rolled-back code (brief killed at 1/9 → resume `+1 restored`, 8 batches
    reviewed, `reused_batches:1`). Real-hardware verification on
    gemma-4-26b-a4b-qat: quick 58s/123 files; deep on settings.py 54s
    finding the ground truth (SECRET_KEY critical L22, DEBUG high L25).
    New docs: 14 (adaptation plan, copied in), 15 (full MCP reference),
    16 (Gemma 4 26B user guide); README/PROMPTS swept clean of semgrep.
20. **Deterministic-findings intake (2026-07-23, same session)** — the
    inverse hand-off the plan aims at: scanner findings flow INTO the MCP
    (the scanner itself stays out). New `findings.py`
    (load/normalize/match, FindingsError, unmatched-reported-not-dropped);
    `start_review(findings_file=)`; deep prompt injection with enrichment
    wording + "code is untrusted data" line (AI-engine doc 9.4); brief
    `static_flags`; quick merge; verbatim `static_findings` attachment with
    `confirmed_by_llm` (identity-preserving, doc 9.6); `deterministic`
    report meta; `stats.static_findings` recomputed from reviews (resume-
    safe). 73 tests. LIVE-VERIFIED on gemma-4-26b-a4b-qat
    (FINDINGS-DEEP-01, 34s): findings file with 2 real findings + 1
    planted fake SQLi → both real ones confirmed_by_llm, the fake
    dismissed WITH reasoning ("standard path manipulation using os.path")
    and left unconfirmed, +2 model-only findings. This is the MCP-scale
    precursor of Context Builder → AI Provider Adapter (WS2); the platform
    still adds policy gates, redaction at normalization, finding-scoped
    context windows, and the safety validator.
21. **Private-repo cloning (2026-07-23, same session)** — user needs to scan
    private repos. Token from **`GITHUB_TOKEN`/`GH_TOKEN` env ONLY**; the
    threat model row "clone token appears in logs or process list"
    (docs/06 line 80) prescribes *URL redaction, no argument token,
    structured errors*, so **no tool takes a token argument** (a test
    enforces this against regression — an argument would land in the chat
    transcript + LM Studio logs). Mechanism: non-secret `x-access-token@`
    username in the URL + `GIT_ASKPASS` helper that dereferences an env var
    (so the token is in NO argv, NO URL, NO file, NO log — `_redact` at the
    runner AND at every message-building site), `http://`→`https://` upgrade
    before sending a credential, creds-in-URL and SSH URLs rejected with
    guidance, manifest/report store only `authenticated: true` provenance.
    New `check_repo_access` tool (ls-remote, ~1s) + `_access_hint` that
    explains GitHub's misleading "not found" for anonymous private access
    and, when a token IS set, the scope/expiry/SSO checklist.
    **Found and fixed while testing:** (a) redaction existed only in
    `_run_git`, so any other message-builder could leak — now redacted at
    each site too; (b) `git@github.com:...` was misdiagnosed as
    creds-in-URL (SSH check now first); (c) **Git Credential Manager was
    being consulted before askpass** on Windows, emitting the misleading
    "Cannot prompt because user interactivity has been disabled" and
    risking the console-less GUI-prompt hang class → `-c credential.helper=`
    now disables inherited helpers (also makes auth explicit/identical in
    the container; 1.2s→0.7s). 91 tests. LIVE-VERIFIED: public clone
    unaffected; real private repo anonymously → correct hint; INVALID token
    → GitHub's "Invalid username or token" in 0.7s, token absent from all
    output, no hang (proves askpass really drives the exchange); token
    passthrough into the container confirmed both ways.
22. **Flask review API + demo console (2026-07-26)** — user asked for an HTTP
    BE for the AI (adaptation-plan WS3). FIRST extracted
    `code_review_mcp/pipeline.py::run_review()` from `server.py` so the
    checkpoint/resume logic is NOT duplicated; `server.py` is now a thin
    adapter mapping callbacks to its job registry (all 91 MCP tests still
    green — but note the seam moved: tests now monkeypatch
    `pipeline.ingest_repository` / `pipeline.review_manifest`, not
    `server.*`). Then `src/review_api/`: config file with cloud+on-prem
    provider registry (keys via `api_key_env` only), prompt consolidation
    with `dry_run`, job store with progress/ETA/restart/cancel, REST v1
    routes, and a single-page demo console (API base field = local or
    remote). **120 tests.** LIVE-VERIFIED against LM Studio: provider probe
    (7 models), dry-run prompt, real deep review with findings enrichment
    (26B, 84s, 3 findings attached, 2 confirmed + 1 false positive
    dismissed), and the full DR drill — **API process killed at batch 1/9**
    → restart listed the checkpoint → same review_id resumed `(+1
    restored)` → cancel stopped at 3/9 `restartable:true` → restart resumed
    `(+3 restored)`.
    Design decisions worth keeping: (a) an `access_token` in an HTTPS body
    IS acceptable (never reaches a model, standard for CI) while an MCP tool
    argument is NOT — tokens are memory-only, deleted in `finally`, never in
    a response/report/log; (b) `action` = HTTP outcome vs `status` = review
    lifecycle, because `{"status": "accepted", **job.public()}` raced the
    worker thread and produced nondeterministic responses; (c) one review at
    a time (`409 busy`) since a single local model would only queue.
23. **Three-component split + reproducibility (2026-07-30)** — the user needs
    teammates to recreate and test the project on their own workstations,
    including with cloud models, since they do not have 24/7 access to the
    GPU rig. Repository reorganised into `mcp-server/` + `backend/` +
    `frontend/` (user chose three folders over two: the MCP is its own
    deliverable, and `backend` declares a dependency on `code-review-mcp` so
    `pipeline.py` is shared, not duplicated). All moves were `git mv`, so
    history is intact. The console left the Flask package: `app.py` now
    resolves an EXTERNAL `frontend/` dir (explicit setting → repo layout →
    beside the package → cwd) and `/` degrades to an explanatory 404 rather
    than taking the API down — that is what lets the same file be served at
    `/` *and* opened standalone against a remote API base.
    Portability work: `${VAR}`/`${VAR:-default}` interpolation in the config
    (+ `_coerce_scalars`, because interpolation flattens ints to strings —
    caught when `port` came back `'8000'`), config-relative path resolution,
    `.env` + `load_dotenv()` in `main()`, `docker-compose.yml` (one `api`
    service, port published on 127.0.0.1), a root-context `backend/Dockerfile`,
    setup/run/test/mock scripts in PowerShell *and* bash, `.gitattributes`
    (LF for `*.sh`), and a **`google_ai_studio`** provider — Gemma over
    Google's OpenAI-compatible endpoint, so a teammate with no GPU runs the
    same model family by changing one request field. New
    `docs/18-team-setup-guide.md`; README rewritten as orientation, plus a
    README per component; docs 14–17 and the index updated. **128 tests**
    (7 new: interpolation, missing-variable failure, config-relative paths,
    frontend discovery, explicit override, API-without-console, and a guard
    that the shipped example config stays valid and keeps a keyless-file
    cloud path).
    **REAL BUG FOUND BY REBUILDING:** `mcp>=1.2.0` was unpinned and mcp 2.0.0
    has since shipped without `mcp.server.fastmcp` — the fresh image died at
    import while the local venv's older 1.x kept working, i.e. it would have
    broken every teammate's install and nobody's existing one. Pinned
    `mcp<2` with the reason in the file. Live-verified end to end through the
    container: clone → 123 files → report on the host mount.
24. **Console split out as its own microservice (2026-09-17).** User: strip
    the debug FE out of the AI backend for the final stage, keep it as a
    separate thing to test the AI connection from Kubernetes, one folder and
    one independently buildable/pushable image per microservice, a README
    each. Then, mid-task: **remove the Kubernetes manifests and teach me to
    write them** — it is a DevOps course. Done: review-api serves no HTML
    (JSON `/`, `/healthz`, `/readyz`); review-console = `index.html` +
    unprivileged nginx proxying `/api/` to `REVIEW_API_URL`, exposing
    `/console.json` so the page names its upstream; per-folder
    `build.sh/.ps1` with `--registry/--tag/--push`; backend image built from
    `backend/` with a NAMED build context for mcp-server, non-root uid 10001;
    compose runs both (console on host **8090**, because the gateway owns
    8080). Manifests were written, deployed to minikube, used to find two
    real bugs, then DELETED; **docs/21** is a guided exercise instead.
    Bugs found: (a) **Kubernetes service links** — a Service named
    `review-api` injects `REVIEW_API_PORT=tcp://ip:8000`, and the API read it
    as its port → CrashLoopBackOff; (b) with a read-only root and no `/tmp`
    volume every review failed while `/readyz` said ready. Both fixed in the
    app. Verified: all three images built and pushed separately to a local
    registry; both images under read-only root + non-root + no caps; a real
    review through console → API → LM Studio under those constraints; in
    minikube (imperative kubectl, no YAML) the injected variable is present
    and ignored, and the console reports the model reachable from inside the
    cluster. **415 tests** across five suites.
25. **Merged onto upstream (2026-09-17).** The checkout was behind the
    team's latest; the user moved this conversation's work aside and brought
    in upstream. A full tree comparison showed this work was already present
    byte for byte (same commit `5fc02a9`, no unique refs or stashes), so the
    merge came down to four upstream additions: the scanner's `--fail-on` CI
    gate (`cli.py`, `runner.py`'s now-public `SEVERITY_ORDER`,
    `test_cli_gate.py`) and `docs/20-phase-3-plan.md`. Their scanner API is
    unchanged, so the pipeline hook still binds. The K8s guide was
    renumbered **20 → 21** (upstream's plan is older and already shared) and
    given a K3s section matching the plan. **426 tests** (scanner 17 → 27).

26. **scanner-worker image (2026-09-25).** The Job image review-api's
    `kubernetes` execution mode was written against
    (`docs/job-container-contract.md`) now exists: `scanner-worker/`, one
    image with the scanner tier and the engine, built from named contexts
    like `backend/`. `entrypoint.py` is a thin wrapper around `run_review()`:
    it validates REPO_URL / REVIEW_ID / MODE / LMSTUDIO_BASE_URL /
    SYSTEM_PROMPT (+ BRANCH, GITHUB_TOKEN or GITHUB_TOKEN_FILE,
    LMSTUDIO_MODEL), and exits **0** done, **1** retryable (LLM down,
    network, timeout, SIGTERM, unknown), **2** permanent (bad input or URL,
    no access, missing branch, unwritable volume), writing a one-line reason
    to `/dev/termination-log`. The token leaves `os.environ` before the engine
    is imported, so scanners never inherit it, and it is scrubbed from every
    log line. Read-only root: TMPDIR/HOME/caches live on `/workspace`;
    `safe.directory` is in the SYSTEM git config. **Verified in Docker** with
    `--read-only --user 10001 --cap-drop ALL`: a full brief review against the
    mock LLM (all three scanners ran, 218 findings, exit 0), a bad token (exit
    2, token not in the output), a missing branch (2), a GitLab URL (2), an
    unreachable LLM (1), and `docker stop` mid-scan (1, "received SIGTERM").
    **Not yet**: run as a real Job on k3s; review-api does not read the exit
    code yet, so an exit-2 review is still retried 5 times. **407 tests**
    across mcp-server 181, backend 154, scanner 27, scanner-worker 45
    (+6 opt-in image tests).
27. **review-api is Kubernetes-ready (2026-09-25, same session).** The
    image lacked the `kubernetes` python client, so `REVIEW_API_EXECUTION_MODE
    =kubernetes` started, passed /healthz, and would fail every review.
    `backend/Dockerfile` now installs `"./backend[kubernetes]"` (quoted: `[ ]`
    is a shell glob). Running that image outside a cluster exposed a second
    bug: `_launch_attempt` raised out of `create()`, so the API answered a bare
    500 and the review stayed `queued`. It now ends `failed` with the reason.
    **409 tests.**
28. **First Job on k3s, then contract v2 (2026-09-25/26).** Elad wrote the
    first Job + PVC by hand (`K8S-Tailcale-Egress/job-test/`); `scan-test-1`
    completed on k3s in 91 s — clone, all scanners, LM Studio through the
    Tailscale egress, report on the PVC. After Sahar's review of the code,
    the contract went to **v2**: `<id>.error.json` (`{"permanent", "error"}`,
    the same one line as the termination log) is how review-api learns why a
    Job failed — a file on the PVC, because reading Pod status needs `list`
    on pods and every Pod carries its Job's env; `<id>.progress.json` gives
    the running file and this attempt's per-file durations in epoch seconds
    (deep mode), since a Job cannot send in-process progress. `k8s_jobs.py`:
    valid Job names for any API id (slug + hash), label ≤ 63 + full id in an
    annotation, 404 ends a review, `MAX_POLL_ERRORS`, permanent errors stop
    at once, stale error files removed before each attempt, `LMSTUDIO_MODEL`
    passed. Image `scanner-worker:0.2.0`. **478 tests.**
29. **One manifest folder, one namespace (2026-09-30).** The cluster files
    lived in two folders that disagreed on the namespace: `k8s/`
    (`ai-scan-repos`) and `K8S-Tailcale-Egress/` (`code-review`). All are now
    in `k8s/`, one sub-folder per service (`storage/`, `postgres/`,
    `review-api/`, `gateway/`, `tailscale-operator/`, `job-test/`), all in
    `ai-scan-repos`, applied with `kubectl apply -k k8s/`. Also fixed: the
    review-api Deployment's missing `/tmp` volume and `LMSTUDIO_BASE_URL`,
    `:latest` placeholder images (tags now in `k8s/kustomization.yaml`,
    worker `scanner-worker:0.3.0`), the unused `ai-scan-repos-jobs`
    namespace, and a placeholder Postgres password in git (the Secret is now
    created from the command line). Every manifest line is commented;
    `k8s/README.md` is the deploy guide. Gateway stays out of the
    kustomization until the public address is decided. Schema-validated
    (kubeconform); **not yet applied to the cluster**.
    Same day, second pass: the whole stack in one `kubectl apply -k k8s/`.
    New `frontend-react/Dockerfile` (+ `nginx/default.conf`), `k8s/frontend/`
    and `k8s/ingress/` (Traefik, one address split by path, so the UI calls
    the gateway same-origin), and the gateway un-commented with the public
    address `http://100.111.216.122` in its three env vars. Secrets stay
    command-line only (`k8s/README.md` step 5).

---

## 9. Gotchas (do not re-learn)

- **Kubernetes service links collide with `*_PORT` settings.** Every Service
  in a namespace injects `<NAME>_PORT=tcp://<ip>:<port>` into every pod, so a
  Service called `review-api` sets `REVIEW_API_PORT` to a URL. The API now
  ignores `tcp://`/`udp://`/`sctp://` values with a warning
  (`config._getenv`), on both config paths; the cluster-side fix is
  `enableServiceLinks: false`. Anything else reading `*_PORT` from the
  environment has the same trap.
- **review-api is API-only — do not re-add static serving.** The console is
  the `review-console` service. `test_pure_api.py` fails if any GET route
  returns HTML or a `static` endpoint appears.
- **Probes: `/healthz` and `/readyz`, never `/api/v1/health`.** The latter
  needs `X-API-Key` once one is set and calls the model when `probe=1`; as a
  liveness probe it would restart a healthy API on every model outage.
- **nginx `add_header` in a `location` drops every inherited header.** The
  console's security headers are server-level only; a test enforces that no
  location adds any.
- **`REVIEW_API_URL` must have no path** — `proxy_pass` with a URI replaces
  the `/api/` prefix. `05-review-api-url.envsh` refuses one at start.
- **The console resolves its upstream at start** → the API's Service /
  compose service must exist first (`host not found in upstream`).
- **`docs/20-phase-3-plan.md` §2.5 predates the console split.** Written on
  7 September, it says review-api serves `frontend/` and that the console
  "would break if separated", and that compose has one service. Both stopped
  being true on 17 September. Its actual decision — `frontend/` goes into the
  **backend repository** (e.g. `backend/console/`) — still works, because it
  is about repos, not containers: the backend repo can ship review-api and
  review-console as two images. Raise it with the team rather than editing
  their plan.
- **Install the scanner editable** (`pip install -e ./scanner`). A plain
  install copies it into site-packages, so when teammates changed it the
  suite kept importing the old copy and their new tests failed to import.
- **No k8s manifests in the repo, deliberately.** The user writes them.
  Help by explaining, not by committing YAML. docs/21 is the brief.
- **Git Bash rewrites `/paths` in docker/minikube args** (`/etc/nginx` →
  `C:/Program Files/Git/etc/nginx`). Prefix with `MSYS_NO_PATHCONV=1`. It
  also collapses `\\` inside heredocs — write edit scripts to files.
- **`core.autocrlf=true` on this machine, and Python's `write_text()` writes
  CRLF on Windows.** Harmless for `.py` (git normalises), fatal for files a
  Linux container reads directly — hence `.gitattributes` pins `*.envsh`,
  `*.template`, `*.conf` to LF.

- **Git for Windows deadlocks in MCP servers** — never pipes/inherited stdin;
  tree-kill on timeout. Docker path avoids the class entirely.
- **LM Studio MCP tool timeout ≈ 60s** (`-32001`); also caps consecutive
  tool calls per turn → long polling loops WILL be cut off; design for it.
- **Small models pick tools by NAME** — keep blocking tools hidden.
- **Gemma reasoning tokens** count against max_tokens; empty content at
  `finish=length` → double budget and retry (implemented). On the 26b, deep
  mode can still fail JSON on a file after retries → recorded as a per-file
  failure, run continues. 12b is the more reliable deep reviewer.
- **Serial LM Studio + concurrent requests = empty responses** — keep
  `REVIEW_PARALLEL=1` unless parallel request processing is enabled.
- **Python forbids new threads during interpreter shutdown** — anything that
  must survive host disconnect must not create threads after main() returns
  (hence inline serial mode).
- **User hot-swaps models** (e4b/12b/31b/26b-a4b/qwen3.6-27b/deepseek-r1);
  empty model id is rejected when several are loaded → `resolve_model()`
  picks the first chat model; **recommend pinning `model=` in prompts**; e4b
  is slow despite its size; 12b is the fast workhorse.
- **`get_review_status`/`check_llm` are async**; heavy tools offload via
  `anyio.to_thread`.
- **Windows memory pressure can SIGKILL the container** (WSL-VM OOM, exit
  137 in `docker events --since 6h --filter event=die`): the in-memory job
  registry dies with it and a respawned container answers "unknown_job".
  Loading a big model mid-review is the typical trigger. Both deep AND brief
  checkpoint → recovery is always: same review_id + same mode → resume.
- **Checkpoints are only OBSERVABLE during a run or after a crash** — a
  successful run DELETES its `.partial.json` (the final report replaces it).
  Quick mode never checkpoints (30s runs need none). If a killed brief left
  nothing behind, suspect a **stale LM Studio container** predating the
  current rebuild (toggle the server off/on).
- **Server-level `review_manifest` test fakes need `**_` in their signatures**
  (brief_state/checkpoint_cb kwargs).
- Report `.md` does not render deep-mode `positives` (JSON has them); quick
  mode's *overview* line numbers can be hallucinated (per-file entries are
  exact) — both known-cosmetic, unfixed by user's choice.
- **NEVER add a token/credential argument to a tool** — env only
  (`GITHUB_TOKEN`). An argument goes into the chat transcript, the model's
  context, and the host logs. `mcp-server/tests/test_private_repo.py` fails the build
  if any tool grows one. Never ask the user to paste a token in chat.
- **Git Credential Manager is bypassed on purpose** (`-c credential.helper=`
  in clone/ls-remote): it runs BEFORE `GIT_ASKPASS`, can try to open a GUI
  prompt (the console-less hang class), and emits a misleading "Cannot
  prompt because user interactivity has been disabled". Don't remove it.
- **Do NOT re-add Semgrep (or any scanner) to this MCP** — retired behavior
  per the adaptation plan §13; scanners return as platform plugins (WS1).
  The hybrid code is at tag `semgrep-hybrid` if reference is needed.
  **Consuming scanner OUTPUT is fine and implemented** —
  `findings_file=` is the sanctioned direction (findings in, scanner out);
  don't confuse the two.
- **`mcp<2` in `mcp-server/pyproject.toml` is load-bearing.** mcp 2.0.0 is a
  breaking redesign that removed `mcp.server.fastmcp` in favour of
  `mcp.server.mcpserver`. Found on 2026-07-30 by rebuilding the image: the
  container died at import with `No module named 'mcp.server.fastmcp'` while
  the local venv (holding an older resolved 1.x) kept working — i.e. it would
  have broken every teammate's fresh install and nobody's existing one. Do not
  lift the bound without porting the server.
- **`load_dotenv()` is called from `main()` only, never at import.** Importing
  it into `load_config` would make the test suite depend on whether the
  developer happens to have a `.env`, and a local `LMSTUDIO_BASE_URL` would
  then fail the interpolation tests.
- **`${VAR}` interpolation returns strings.** Anything typed in `ApiConfig`
  (`port: int`, `debug: bool`) must survive it — that is `_coerce_scalars`.
  Adding a new typed config key means it is covered automatically; adding a
  new *type* means extending that function.
- **The backend Dockerfile's build context is the REPOSITORY ROOT**, not
  `backend/`, because the image needs `mcp-server/`, `backend/` and
  `frontend/`. Hence the root `.dockerignore`. `mcp-server/`'s own build
  context is that folder, with its own `.dockerignore`.
- **The API image BAKES `frontend/` and `config/api.yaml`.** This is the same
  trap as the LM Studio container: a plain `docker compose up` silently serves
  the old copies, with no error, and `restart` does not help because the files
  live in image layers. Cost real debugging time on 2026-08-16 (a new provider
  and a new UI field appeared to "not exist"). **Both are now bind-mounted
  read-only in `docker-compose.yml`**, so config and UI changes need only a
  restart — Python changes still need `--build`. Diagnose with
  `docker compose exec api sh -c "grep -c <thing> /app/config/api.yaml"`.
- **`.gitattributes` forces LF on `*.sh`, Dockerfiles and YAML.** Without it,
  Windows CRLF checkouts hand teammates
  `bad interpreter: /usr/bin/env bash^M`. `*.ps1` is pinned to CRLF.
- **Reports are per-machine files, not shared state.** `reports/` belongs to
  whichever machine runs the API, so a teammate running their own API sees an
  empty list. Sharing = one host exposes its API (`REVIEW_API_BIND` +
  `REVIEW_API_HOST` = the **tailnet** address, never `0.0.0.0`, which also
  exposes it on café wifi) and everyone else points the console's API base at
  it. `is_loopback_host()` drives a startup line saying whether the API is
  reachable, plus a banner when it is reachable with no `REVIEW_API_KEY` —
  the API clones repos, takes tokens and DELETEs reports, and the shared key
  is the only thing in front of it. Verified live over the real tailnet
  (100.93.244.39) with the console served from a different origin.
- **NEVER send unnumbered source to the model.** `llm.number_lines()` puts
  `NNN | ` in front of every line and `clamp_issue_lines()` nulls anything
  outside the range shown. Asked for a line number without seeing one, a
  model *estimates*: accurate near the top of a file, tens of lines out by
  the bottom. A teammate's 2026-08-16 report on `kubernetes-exercises` had
  six correct findings at six wrong lines (path traversal reported at 54,
  actually 69). Findings that read as authoritative and point at innocent
  code are worse than no findings. Chunk offsets come from
  `split_content_with_lines()`, so part 2 reports file lines, not chunk
  lines; brief mode numbers its packed heads and budgets on the NUMBERED
  length (the gutter is ~40% on short-line files).
- **The clone does not survive `ingest_repository()`.** It is deleted in a
  `finally`, so anything needing the files on disk (the Track 3 scanner tier)
  must run through the `on_clone=` hook, which fires after checkout and
  before the manifest filter — deliberately before, so a scanner sees the
  WHOLE tree, not just reviewable source. Exceptions propagate and the clone
  is still removed. `pipeline._scanner_hook()` wires `repo_scanner` in and
  no-ops when that package is absent (it is optional — this component still
  runs standalone), when `REVIEW_RUN_SCANNER=0`, or when the scan raises,
  recording the reason each time instead of failing the review.
  **Caveat to tell Track 3: the clone is `--depth 1`, so Gitleaks sees ONE
  commit** — it scans the working tree, not history, which is precisely the
  "looks like success" failure they warned about.
- **`ingest_repository` grew a `token` parameter (2026-09-06) — it was
  missing.** `pipeline.run_review` had always called it with `token=` behind
  a `**{...} if github_token else {}` guard, so public reviews passed and
  every PRIVATE one died with `unexpected keyword argument 'token'` before
  cloning. The guard is what hid it from the tests. Same shape as the
  `check_access` bug: a caller written against a signature that did not
  exist.
- **Minified files defeat a line-based chunker.** `split_content_with_lines`
  splits on line boundaries; a minified bundle is ONE line, so a 480 KB file
  produced a single chunk 10x over the 48 KB budget and went to the model
  whole — the "stuck for hours" report from a teammate's real run. An
  over-long line is now hard-split mid-line (still lossless, line number
  unchanged). Separately `looks_generated()` skips such files at ingest:
  the test is CONTENT (line-length distribution, generated-file headers),
  never a filename blacklist, which goes stale. It is deliberately
  conservative — skipping human code is worse than reviewing a bundle — so
  a file must be >20 KB AND shaped wrong. Verified on real jQuery 3.7.1 and
  Bootstrap 5.3.3: 320,455 chars -> 122.
- **Ingest must report what it did not read.** Files of an unreviewable type
  are counted into `manifest["excluded"]`, never dropped silently, and
  `render_markdown` prints a warning next to the verdict. Before this, a
  Kubernetes repo reviewed 2 of 57 files and the report said
  "files reviewed: 2, skipped: 0" — which reads as full coverage.
  `DEFAULT_EXTENSIONS` therefore includes YAML/JSON/Terraform/HCL and
  `classify_file()` matches extensionless `Dockerfile`/`Makefile`/
  `Jenkinsfile` by name. An **explicit** `extensions` argument suppresses the
  name matching: asking for `.py` must not also return a Dockerfile.
  `.env`/`.netrc`/ssh keys are in `NEVER_DEFAULT_NAMES` — sending one to a
  model would put live secrets in its context for no review value.

---

## 10. Current state

- **THREE microservices over one shared pipeline** (§2b): code-review-mcp
  (stdio), review-api (API only) and review-console (nginx test console),
  each its own folder, image, build script and README. **426 tests** across
  gateway 65 · mcp-server 170 · backend 130 · scanner 27 · frontend 34
  (+ security-tests 54/2 skipped). Previously live-verified and still standing: MCP
  kill-test, real-26b review, findings enrichment, private-repo auth, and the
  full API DR drill (API killed mid-review → resume → cancel → restart).
- **scanner-worker** (2026-09-25, history item 26): the Job image for
  `kubernetes` execution mode. Built and verified in Docker under the Job's
  security posture; **not yet run as a Job on the cluster**. See
  `scanner-worker/README.md`.
- **Git:** checkpoints `DR-0` (hybrid baseline, tag `semgrep-hybrid`) →
  `DR-1` (rollback) → `DR-2` (docs+verification) → `DR-3` (findings intake) →
  `DR-4` (private repos) → `DR-5` (Flask API+FE) → `DR-6` (three-component
  split + reproducibility). **No remote yet** — the user pushes it themselves.
- **Run everything:** `docker compose up --build` (console on :8090, API on
  :8000 — JSON only), or
  `scripts/setup.ps1` then `scripts/run-api.ps1` (`.sh` equivalents on
  Linux/macOS). Config: `.env` is the only per-machine file.
- `.mcp.json`: `code-review` (venv, blocking enabled, for Claude Code) and
  `code-review-docker` (container, what LM Studio uses). **Still valid after
  the split** — it references the image and the venv, neither of which moved.
  It does hardcode `C:\Projects\Final-Project`, so teammates must fix those
  two paths (documented in docs/18 §9).
- After ANY MCP code change: `docker build -t code-review-mcp ./mcp-server`
  then **toggle the MCP server off/on in LM Studio** (else it keeps the old
  container — the #1 cause of "the fix didn't take").
- Next per the adaptation plan: WS1 (scanner plugins incl. re-homed Semgrep,
  `NormalizedFinding`, redaction) — in the platform tree, not here.

---

## 11. How to test / continue

```powershell
cd C:\Projects\Final-Project
scripts\test.ps1                                  # 128 tests, no LLM/network
# equivalently: .venv\Scripts\python.exe -m pytest mcp-server/tests backend/tests -q

docker build -t code-review-mcp .\mcp-server      # MCP image
docker compose build api                          # API image (context = root)

# LLM-free everything: scripts\run-mock-llm.ps1 (port 8765), then either
#   $env:LMSTUDIO_BASE_URL="http://localhost:8765/v1"; scripts\run-api.ps1
# or the raw e2e driver:
#   $env:REVIEW_OUTPUT_DIR="<some dir>"
#   .venv\Scripts\python.exe mcp-server\tests\e2e_review.py --repo <url> `
#       --review-id E2E-01 --mode brief
# Kill-test drill: kill e2e_review.py mid-run, rerun the same command ->
#   progress must show "(+N restored)" and the report resumed:true.
```

**Verified on 2026-07-30 after the split:** both suites green from the repo
root AND from each component folder; both images build; the containerised API
reached a host LLM through `host.docker.internal`, cloned the 123-file test
repo and wrote `COMPOSE-01.json/.md` to the host `reports/` mount; the demo
console is served from `/app/frontend` inside the image; `.env` →
`${VAR}` → provider registry confirmed end to end (`GOOGLE_MODEL` in `.env`
appeared as the provider's `default_model`); CORS verified for a
standalone-console origin; the MCP image exposes all 10 tools over stdio with
the blocking tool still hidden.

**Standard real-hardware prompt (paste to Gemma in LM Studio):**
> Run a brief review of https://github.com/hadsahar/Secured-e-voting-system
> with start_review (review_id "TEST-01", model "google/gemma-4-26b-a4b-qat",
> system prompt "Strict security reviewer"). Poll get_review_status with
> wait_seconds 40 until done, printing the progress line each cycle, then
> give me the verdict, the top findings, and where the files were saved.

**Crash-recovery prompt:** "call get_review_status for job <id>" → if
unknown, it lists resumable checkpoints → "run start_review again with the
same review_id and mode" resumes (works for brief AND deep).

**Test repo:** `https://github.com/hadsahar/Secured-e-voting-system`
(Django e-voting, 123 reviewable files, 4 skipped-too-large, commit
`5a164c3e`). Known real findings: hardcoded SECRET_KEY + DEBUG=True in
`settings.py`, WandB API key in `utils/debug.py`, RSA with primes 3 & 11 in
`SMPC/rsa.cpp`, `rand()` secret-sharing in `SMPC/intermediateServer.h`,
commented-out admin permissions in `candidate/admin.py`.

**Local dummy repo (offline):** `mcp-server/tests/fixtures/dummy_repo` — one planted
instance of every surface-check class; exercised by
`mcp-server/tests/test_dummy_repo.py` including both DR crash→resume cycles,
and reused by `backend/tests` via its `conftest.py`.

---

## 12. Persistent memory (auto-loads for this project dir)

`C:\Users\Elad Eytan Feldman\.claude\projects\C--Projects-Final-Project\memory\`:
- `code-review-mcp-project.md` — project overview + full feature history.
- `git-mcp-windows-deadlock.md` — the git deadlock fix.
- `MEMORY.md` — index.
