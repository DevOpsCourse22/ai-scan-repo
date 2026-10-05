# 15. MCP Server Documentation — `code-review-mcp`

**Project:** AI Scan Repos (experience-layer / local provider path)
**Document Version:** 2.0 — post-rollback (LLM-only, no Semgrep)
**Status:** Current
**Component:** `code-review-mcp` — the standalone MCP server in this repository

> This document fully discloses the MCP server: what it is, how it works,
> every tool it exposes, every setting, the report format, and the
> disaster-recovery (checkpoint/resume) design. For a hands-on walkthrough
> with the Gemma 4 26B model, read the
> [Gemma 4 26B user guide](16-gemma-user-guide.md) next.

---

## 1. What this component is

`code-review-mcp` is a Python **MCP (Model Context Protocol) server** that
performs AI code review of GitHub repositories using a **local LLM served by
LM Studio**, packaged as a portable Docker container.

MCP is an open protocol that lets an LLM application (the **host** — LM
Studio, Claude Code, Claude Desktop) launch external programs (**servers**)
and call the **tools** they expose. This server exposes code-review tools;
the host's model decides when to call them and presents the results.

### Version note — the Semgrep rollback

An earlier iteration of this server bundled a Semgrep static-analysis pass
("hybrid reviews"). Per the
[MCP adaptation plan](14-mcp-adaptation-plan.md), deterministic scanning
(Semgrep, Gitleaks, Trivy) is the **platform's** responsibility — the
scanners become pinned scanner plugins in the analysis plane, not features
of this MCP. This version is therefore rolled back to **LLM-only** reviews.
The complete hybrid implementation is preserved in git under the tag
`semgrep-hybrid`.

### The core idea: one model, two roles

When driven from LM Studio, the same local model plays **two roles at
once**:

1. **MCP host** — in LM Studio's chat, the model (e.g. Gemma) calls this
   server's tools.
2. **Reviewer** — the server calls back over HTTP to LM Studio's
   OpenAI-compatible endpoint (`http://localhost:1234/v1`); the model that
   answers (the same one, or another via the `model` argument) reads the
   code and writes the findings.

**The server contains no AI.** It is orchestration only: clone, chunk,
prompt, enforce JSON, merge, checkpoint, write reports. Wall time ≈ number
of LLM calls × model speed; the server itself adds a few seconds.

---

## 2. Architecture

```
LM Studio (model = host) --stdio--> docker run --rm -i code-review-mcp
        ^                                   |
        | HTTP (OpenAI API, model=reviewer) |
        +-----------------------------------+
   1. git clone --depth 1  -> temp dir inside the container (deleted after)
   2. repo -> JSON manifest {repo_url, commit, files:[{path,language,content}], skipped}
   3. review (quick | brief | deep)  [+ per-unit checkpoint in brief & deep]
   4. merge -> report
   5. write <name>.json + <name>.md -> /reports volume -> host reports\ folder
```

- Python 3.10+ (`mcp` FastMCP over stdio transport, `openai` SDK client).
- Docker image `code-review-mcp:latest` (`python:3.12-slim` + git, ~410 MB).
- Container→host LM Studio via `host.docker.internal`
  (`--add-host=host.docker.internal:host-gateway` for Linux parity).
- Reports: `/reports` (container) = a `-v` mount of the host `reports\`
  folder.

### Source layout (`mcp-server/src/code_review_mcp/`)

This component is one of three in the repository — see
[README](../Code-Review-MCP/README.md) and [18-team-setup-guide](18-team-setup-guide.md) for
the layout. `pipeline.py` is the review engine; the
[Flask backend](17-review-api.md) calls the same function, so the behaviour
documented here applies to both front ends.

| File | Responsibility |
|---|---|
| `config.py` | All settings (env-overridable), extension allowlist, skip-dirs, extension→language map, `github_token`. |
| `pipeline.py` | **The shared review runner** — `run_review()` owns findings intake, clone, file filtering, checkpoint discovery and resume, progress and ETA, review, and report/markdown writing. Both front ends call it, so crash recovery exists once. |
| `repo_ingest.py` | Clone + JSON manifest. Windows git hardening (`_run_git`/`_kill_tree`: stdin from NUL, file-redirected output, prompts off, tree-kill on timeout). Private-repo auth: `_auth_context`/`_write_askpass`/`_redact`, `check_access`. |
| `findings.py` | Deterministic-findings intake (§4.11); runs no scanner. |
| `surface.py` | Quick mode's regex static scanner (secrets, eval/exec, pickle, shell=True, verify=False, weak RNG, bare except, asserts, TODO) + heuristic score. No LLM. |
| `llm.py` | Streaming LM Studio client; JSON enforcement (deep + brief schemas); invalid/empty-JSON retries incl. token-budget doubling; `resolve_model()`; fatal (`LLMUnavailableError`) vs per-file errors. |
| `review.py` | Orchestration: `review_manifest` (3 modes), chunking, `pack_digest_batches`, quick/brief report builders, merge, `render_markdown`, checkpoint callbacks. Serial mode runs inline (no thread pool). |
| `server.py` | The FastMCP server: all tools, job registry, `files` filter, output resolution, progress payload, checkpoint/resume logic, checkpoint discovery. |
| `cli.py` | Standalone CLI (no MCP host) for testing. |

---

## 3. Review modes

| Mode | LLM calls | Coverage | Typical time (123-file repo) | Use for |
|---|---|---|---|---|
| **quick** | 1 | all files (regex surface scan + AI overview) | ~30–60 s | instant first impression |
| **brief** (default) | batches + 1 (~10) | ALL files, head of each, batched | ~5 min (12b) / ~14 min (26b) | full-repo triage |
| **deep** | 1 per file | full detail, checkpointed | hours whole-repo; ~1–5 min/file on 26b | targeted review with `files=` |

- **quick** — every file goes through the regex surface scanner
  (milliseconds), then ONE small LLM call over the findings table produces
  the overall verdict.
- **brief** — every non-trivial file's head (`REVIEW_BRIEF_FILE_CAP` chars)
  is packed into digest batches under `REVIEW_BRIEF_PACK_CHARS`; each batch
  is one LLM call; a final merge call produces the overall verdict. No file
  is ever silently dropped.
- **deep** — one detailed LLM call per file (large files are chunked and
  merged). Scope it with `files=["*/views.py", "settings.py"]` — full
  paths, bare filenames, and globs all match.

**Recommended workflow:** quick or brief triage over everything → deep on
the flagged files.

---

## 4. Complete tool reference

### 4.1 `start_review` — the entry point

```
start_review(repo_url, system_prompt, output_path="review-report.json",
             branch="", extensions=None, mode="brief", model="",
             review_id="", files=None) -> {status, job_id, mode, next_step}
```

Starts a review **in the background** and returns a `job_id` instantly
(never blocks, never times out). One job runs at a time; starting a second
returns `already_running` with the active job's id.

| Argument | Meaning |
|---|---|
| `repo_url` | GitHub repository URL (`https://github.com/owner/repo`). |
| `system_prompt` | Reviewer instructions (persona, focus areas). JSON output rules are appended automatically. |
| `output_path` | Report filename; relative paths land in the output dir (`/reports` in Docker). Ignored when `review_id` is set. |
| `branch` | Branch to review (default branch when empty). |
| `extensions` | Extension filter, e.g. `[".py", ".ts"]`. Defaults to a broad built-in set covering application code **and** infrastructure-as-code (YAML, JSON, Terraform, HCL) plus extensionless `Dockerfile`/`Makefile`/`Jenkinsfile`. Passing this is taken literally — an explicit `[".py"]` does *not* also pick up a Dockerfile. Whatever is left out is counted in the report's `excluded` block, never dropped in silence. |
| *(automatic)* | Minified bundles and generated files (notebook exports, `@generated` headers) are skipped at ingest and named in the report's `skipped` block with the reason. The test is the file's CONTENT — line-length distribution and generated-file markers — not its name, so it also catches bundles nobody blacklisted. `REVIEW_SKIP_GENERATED=0` reviews them anyway. |
| `mode` | `"quick"`, `"brief"` (default), or `"deep"`. |
| `model` | LM Studio model id to review with (e.g. `google/gemma-4-26b-a4b-qat`). Empty = whatever is loaded. **Pin this** when several models are loaded. |
| `review_id` | Names the outputs: `"X"` → `X.json` + `X.md`. Also the key for checkpoint resume. |
| `files` | Review ONLY matching files: full repo paths, bare filenames, or globs. Combine with `mode="deep"`. |
| `findings_file` | Path to a JSON file of **deterministic scanner findings produced outside the MCP** (the platform hand-off — see §4.11). The findings are fed to the model for verification/enrichment and attached to the report verbatim. |
| *(automatic)* | When the Track 3 scanner package `repo_scanner` is installed, the deterministic tier runs over the clone automatically and its findings feed the review — `findings_file` becomes an override rather than a requirement. The scan happens inside the ingest window, so the repository is cloned once (ADR-001). `REVIEW_RUN_SCANNER=0` disables it; an absent package or a failing scan is recorded, never fatal. |

Works for **private** repositories too — same URL form, authentication via
the server's `GITHUB_TOKEN` (§4.10). No tool accepts a token argument.

**Self-healing:** calling `start_review` again with the same `review_id`
and mode after a crash **resumes from the checkpoint** (§6).

### 4.2 `get_review_status` — long-polling progress

```
get_review_status(job_id, wait_seconds=40) -> compact status JSON
```

Waits up to `wait_seconds` (0–45) **server-side** before answering, so one
call covers a long stretch of the review. Running jobs return:

```json
{"status": "reviewing", "done": 5, "total": 9, "remaining": 4,
 "percent": 56, "elapsed_seconds": 70, "eta_seconds": 53,
 "progress": "56% — 5/9 batches done, 4 left, ~0m 53s left (70s elapsed)",
 "do": "Report this progress line, then either CALL get_review_status again or TELL the user you stopped polling..."}
```

- `done` counts **completed** units only; the ETA is stable (recomputed
  only when a unit finishes).
- The `do` field is an anti-hallucination contract for small models: either
  keep calling, or say you stopped — never pretend to monitor.
- An unknown `job_id` (e.g. after a server restart) returns
  `resumable_checkpoints` and a resume hint instead of an error.

### 4.3 `get_report` — summaries and full reports

```
get_report(report, file="", full=False)
```

`report` is a job_id **or** a report filename. Default returns a compact
summary (stats + per-file scores/issue counts); `file="src/app.py"` returns
one file's complete review; `full=true` returns the raw JSON (guarded
against oversized responses). Works even with no volume mounted — results
are never lost while the session lives.

### 4.4 `get_report_entry` — context-safe iteration

```
get_report_entry(report, index=0) -> {index, of, remaining, review, next}
```

Reads ONE file's review per call, so a small-context host can walk a large
report without exhausting its window.

### 4.5 `append_report_notes` / `read_report_notes` — the scratchpad

```
append_report_notes(report, text)      # appends to <name>.notes.md
read_report_notes(report, offset=0)    # reads it back, 6000 chars per page
```

The iterate→render→summarize workflow: read an entry, write your rendering
to the notes, next entry… then read the notes back and produce the final
summary. Measured cost with Gemma driving: ~2.5 KB of context for a whole
report.

### 4.6 `list_reports`

Lists all report files in the output directory plus any
`resumable_checkpoints` (interrupted reviews that can be resumed).

### 4.7 `check_llm`

Verifies the LM Studio endpoint is reachable and the model returns parseable
JSON. While a review job is active it answers `busy` instantly instead of
queueing a probe behind the running generation.

### 4.8 `ingest_repository_only`

Clone → JSON manifest only, **no LLM**. Useful for testing stage 1 and for
inspecting what a review would see.

### 4.9 `review_repository` (hidden by default)

The whole pipeline as ONE blocking call. **Hidden unless
`REVIEW_ENABLE_BLOCKING=1`**: hosts with short tool timeouts (LM Studio:
60 s) kill it mid-review, and small models reliably pick it by name over
the job flow. Long-timeout hosts (Claude Code) may enable it.

### 4.9b `check_repo_access` — can the server clone this repo?

```
check_repo_access(repo_url, branch="") -> {status, authenticated, ...}
```

A `git ls-remote` (no clone, no LLM, ~1 s). Run it before a long review of a
private repository so a missing, expired, or unauthorized token costs
seconds instead of failing an hour into the run. `branch` optionally
confirms a branch exists. Reports whether a token is configured — **never
its value**.

### 4.10 Private repositories

Private repos are reviewed exactly like public ones: pass the normal
`https://github.com/owner/repo` URL. Authentication comes from the
**server's own environment**.

**Setup** — a GitHub personal access token in `GITHUB_TOKEN` (or `GH_TOKEN`):

- *Classic PAT*: needs the `repo` scope.
- *Fine-grained PAT*: needs **Contents: Read** on the repositories you scan.
- Organization repos with SAML SSO additionally require the token to be
  **authorized for that organization**.

Docker — pass the host variable through (already in [.mcp.json](../Code-Review-MCP/.mcp.json)):

```powershell
$env:GITHUB_TOKEN = "ghp_..."      # or set it in LM Studio's mcp.json "env"
docker run --rm -i --add-host=host.docker.internal:host-gateway `
  -v C:\Projects\Final-Project\reports:/reports `
  -e GITHUB_TOKEN code-review-mcp
```

A bare `-e GITHUB_TOKEN` (no `=value`) forwards the host's variable and
keeps the secret out of the command line. For the venv entry, set the
variable in the environment or the MCP host's `env` block.

**Why there is no `token` argument on any tool.** A tool argument would
travel through the MCP protocol: into the chat transcript, the model's
context, and the host's logs. The threat model's row *"clone token appears
in logs or process list"* prescribes exactly the opposite — *URL redaction,
no argument token, structured errors*. So the token is server-side
configuration only. **Never ask the user to paste a token into chat.**

**How the credential is handled** (`repo_ingest.py`):

| Control | Implementation |
|---|---|
| Not in the process list | The token is **never** a git argument. Git obtains it through `GIT_ASKPASS`, which reads it from the child process environment. |
| Not in the URL | The URL carries only the non-secret username `x-access-token@`; the password comes from askpass. Credentials embedded in a URL by the caller are **rejected** with guidance. |
| Not on disk | The askpass helper script contains no secret — it dereferences an environment variable. |
| Not in logs or errors | `_redact()` scrubs the token from git output, exceptions, and log records; applied at both the runner and every message-building site. |
| Not in reports | The manifest and report store the clean URL plus an `authenticated: true/false` provenance flag — never the value. |
| No ambient credentials | `-c credential.helper=` disables inherited helpers (Git Credential Manager etc.), so authentication is explicit, identical in the container and on any machine, and cannot open a GUI prompt that would hang the server. |
| TLS enforced | An `http://` URL is upgraded to `https://` before any credential is sent. |

**Honest diagnosis.** GitHub answers *"repository not found"* — not 403 — for
a private repo requested anonymously, which reads like a typo. Clone and
access failures therefore append a hint stating whether a token was
configured at all, and either the private-repo explanation or the
scope/expiry/SSO checklist.

**Verified** (2026-07-23): public clone unaffected; a real private repo
anonymously → correct private-repo hint; an invalid token → GitHub's
`Invalid username or token` in 0.7 s with the token absent from all output
and no hang (proving askpass is really driving the credential exchange);
token passthrough into the container confirmed.

### 4.11 Deterministic-findings intake (`findings_file`) — the platform hand-off

Per the [AI Analysis Engine design](03-high-level-design/09-ai-analysis-engine.md),
the platform is **deterministic-first**: scanners produce findings, and the
AI's job is *enrichment* — explanation, verification, prioritization,
remediation — not coverage. The MCP supports this direction today: it runs
**no scanner itself**, but `start_review(findings_file=...)` accepts a JSON
file of findings produced by the project's separate scanner component and
feeds them through the review.

**File shape** (tolerant; relative paths resolve against the reports dir,
so the scanner component can drop the file into the shared `reports\`
folder):

```json
{
  "source": "semgrep",
  "findings": [
    {"file": "app/views.py", "rule": "sql-injection", "severity": "high",
     "line": 12, "description": "...", "cwe": "CWE-89"}
  ]
}
```

A bare list also works. `severity` accepts the report vocabulary
(critical/high/medium/low/info) plus raw scanner vocab (`ERROR`→high,
`WARNING`→medium). Findings are capped at 20 per file and descriptions at
300 chars to protect prompt budgets.

**What each mode does with the findings:**

- **deep** — each file's findings go into the prompt BEFORE the code with
  enrichment instructions: verify each finding with its exact line, explain
  impact, give concrete remediation — or dismiss it as a false positive
  with reasoning; then find what static analysis can't see. The prompt also
  marks the code as untrusted data (prompt-injection control).
- **brief** — flagged files carry compact `static_flags` lines inside the
  digest batches (budget-accounted), steering the model to the hot spots.
- **quick** — findings merge into each file's surface-scan issues.

**Report contract (identity-preserving):**

- Findings are attached to reviews **verbatim** as `static_findings` — the
  model can never remove, downgrade, or rewrite a deterministic finding.
- Each carries `confirmed_by_llm: true` when the model independently
  reported an issue within ±2 lines — the highest-confidence tier.
- The report gains a top-level `deterministic` meta block (source file,
  totals, by-severity, and `unmatched` for findings whose paths were not in
  the reviewed set — dropped evidence is reported, never silent).
- `stats.static_findings` counts attached findings and is recomputed from
  the reviews, so checkpoint resume preserves it.
- A missing/invalid findings file **fails the job immediately** (honest
  degradation — a review asked to enrich findings must not silently run
  without them).

**Live-verified** (2026-07-23, gemma-4-26b-a4b-qat): a findings file with 2
real findings + 1 deliberately false SQL-injection claim → both real
findings `confirmed_by_llm`, the false positive explicitly dismissed in the
summary ("standard path manipulation using os.path") and left unconfirmed,
plus 2 new model-only findings. Report `FINDINGS-DEEP-01`.

**Relation to the platform:** this is the MCP-scale precursor of the
Context Builder → AI Provider Adapter chain (adaptation plan §5.1–5.2).
The full platform adds what the MCP deliberately does not: policy gates,
redaction at normalization, minimized finding-scoped context windows
(today the model still sees the whole file), and the schema/safety
validator. Findings supplied here are expected to arrive already
normalized and redacted by their producer.

---

## 5. Report format

Every review writes two files side by side:

- `<name>.json` — the machine-readable report:

```json
{
  "repo_url": "...", "branch": "...", "commit": "...", "model": "...",
  "generated_at": "...", "mode": "brief",
  "resumed": true, "reused_batches": 1,
  "stats": {
    "files_reviewed": 123, "files_failed": 0, "files_skipped": 4,
    "files_excluded": 31,
    "issues_total": 7,
    "issues_by_severity": {"critical": 1, "high": 2, "medium": 3, "low": 1, "info": 0},
    "average_score": 7.4
  },
  "overall": {"summary": "...", "score": 6, "key_issues": [...], "batches": 9},
  "deterministic": {"enabled": true, "source": "semgrep",
                    "file": "scan.json", "findings_total": 3,
                    "findings_by_severity": {"high": 2, "medium": 1}},
  "reviews": [
    {"file": "src/app.py", "summary": "...", "score": 7,
     "issues": [{"severity": "high", "line": 42, "title": "...",
                 "description": "...", "suggestion": "..."}],
     "static_findings": [{"source": "semgrep", "rule": "sql-injection",
                          "severity": "high", "line": 40,
                          "description": "...", "confirmed_by_llm": true}],
     "positives": ["..."]}
  ],
  "failures": [{"file": "...", "error": "..."}],
  "skipped": [{"path": "...", "reason": "..."}]
}
```

  `resumed` / `reused_batches` (brief) or `reused_reviews` (deep) appear
  only after a checkpoint resume. `overall` appears in quick/brief modes.
  `deterministic` and per-review `static_findings` appear only when a
  `findings_file` was supplied (§4.10). `model` records the **actual**
  model id LM Studio reported.

- `<name>.md` — the human-readable render: verdict, "Key issues" list with
  severity icons (🔴🟠🡡🔵⚪), per-file sections, coverage notes.

Known cosmetic limits (accepted): deep-mode `positives` are in the JSON but
not rendered in the `.md`; quick mode's *overview* line numbers can be
approximate (per-file surface entries are exact).

---

## 6. Disaster recovery: checkpoints and resume

This is the reliability core of the server, kept fully intact through the
Semgrep rollback and **verified by killing real processes mid-review**.

### How checkpoints work

- **Deep mode** checkpoints after every completed **file**; **brief mode**
  after every completed **batch**. Quick mode never checkpoints (a ~30 s
  run needs none).
- The checkpoint is `<name>.partial.json` next to the report, written
  **atomically** (temp file + rename — never half-written), containing the
  repo URL, commit, mode, and every completed unit's result.
- A crash — LM Studio killed, container OOM-killed (WSL exit 137), host
  reboot — loses at most the unit in flight.

### How resume works

Run the **same `start_review` again with the same `review_id` and mode**:

1. The server finds `<review_id>.partial.json`.
2. It verifies the checkpoint's **commit matches** the repository's current
   commit (a changed repo safely triggers a fresh review).
3. Completed units are reused: the status line shows `(+N restored)`, only
   the remaining units go to the LLM, and the final report carries
   `resumed: true` with `reused_reviews`/`reused_batches`.
4. On success, the final report **replaces** the checkpoint (the
   `.partial.json` is deleted). Checkpoints are therefore only observable
   during a run or after a crash.

### Finding lost work

- `get_review_status` with an unknown job id lists `resumable_checkpoints`.
- `list_reports` shows them too.
- The container thread is **non-daemon**: if the MCP host disconnects
  mid-review, the container finishes the review on its own and writes the
  final report before exiting.

### Verified drills

| Drill | Result |
|---|---|
| Unit tests: crash a deep review after 3/6 units, rerun | checkpoint holds 3 reviews; resume reuses all 3, reviews the rest, deletes the checkpoint (`mcp-server/tests/test_dummy_repo.py`) |
| Unit tests: crash a brief review after batch 1, rerun | `reused_batches: 1`, only remaining batches reviewed |
| Live kill-test (mock LLM, real clone of the 123-file test repo) | process killed at batch 1/9 → rerun showed `(+1 restored)`, reviewed 8 remaining batches, final report `resumed: true, reused_batches: 1`, checkpoint deleted |

---

## 7. Configuration (environment variables)

Set with `-e` on `docker run`, or in the MCP host's server entry.

| Variable | Default | Meaning |
|---|---|---|
| `LMSTUDIO_BASE_URL` | `http://localhost:1234/v1` (`http://host.docker.internal:1234/v1` in the image) | LM Studio endpoint |
| `LMSTUDIO_MODEL` | *(empty)* | Default model id; empty = first loaded chat model |
| `LMSTUDIO_API_KEY` | `lm-studio` | Ignored by LM Studio, required by the client |
| `REVIEW_OUTPUT_DIR` | *(empty; `/reports` in the image)* | Base dir for relative output paths |
| `REVIEW_MAX_CHUNK_CHARS` | `48000` | Per-request file content budget (bigger files are chunked) |
| `REVIEW_MAX_FILE_BYTES` | `512000` | Skip files larger than this |
| `REVIEW_MAX_FILES` | `500` | Cap on files per run |
| `REVIEW_TEMPERATURE` | `0.2` | Sampling temperature |
| `REVIEW_MAX_TOKENS` | `4096` | Max tokens per deep-mode response |
| `REVIEW_BRIEF_MAX_TOKENS` | `3000` | Token cap per brief-mode response |
| `REVIEW_BRIEF_PACK_CHARS` | `12000` | Char budget per brief digest batch |
| `REVIEW_BRIEF_FILE_CAP` | `1200` | Max chars contributed per file to a digest |
| `REVIEW_TRIVIAL_FILE_CHARS` | `50` | Files with fewer non-whitespace chars skip the LLM |
| `REVIEW_JSON_RETRIES` | `2` | Self-correction retries on invalid JSON |
| `REVIEW_STALL_TIMEOUT` | `300` | Seconds with NO tokens before the stream is declared dead (scaled × parallel) |
| `REVIEW_HEARTBEAT_SECONDS` | `3` | Min seconds between streaming heartbeats |
| `REVIEW_PARALLEL` | `1` | Concurrent LLM requests (1–8). Raise ONLY after enabling parallel request processing in LM Studio |
| `REVIEW_ENABLE_BLOCKING` | *(unset)* | `1` exposes the blocking `review_repository` tool |
| `GITHUB_TOKEN` / `GH_TOKEN` | *(empty = anonymous)* | PAT for cloning **private** repositories (§4.10). Env only — never a tool argument, never logged |

### Reliability mechanisms in the LLM client

- **Liveness, not timeouts**: responses are streamed; as long as tokens
  arrive (including hidden reasoning tokens) the model may take as long as
  it needs. Only a genuine stall or connection failure ends a run.
- **JSON enforcement**: fence-stripping and brace-scanning parsers, plus
  self-correction retries.
- **Empty-response budget doubling**: reasoning models (Gemma 4) can spend
  the whole token budget thinking and emit nothing; the retry doubles the
  budget and instructs "think less".
- **Fatal vs per-file errors**: endpoint down / stalled aborts the run
  (`LLMUnavailableError` — every remaining file would fail identically);
  a single file's parse failure is recorded in `failures` and the run
  continues.

---

## 8. Deployment

### Docker (what LM Studio uses)

```powershell
cd C:\Projects\Final-Project
docker build -t code-review-mcp .\mcp-server
```

MCP host entry (see [.mcp.json](../Code-Review-MCP/.mcp.json)):

```json
"code-review-docker": {
  "command": "docker",
  "args": ["run", "--rm", "-i",
           "--add-host=host.docker.internal:host-gateway",
           "-v", "C:\\Projects\\Final-Project\\reports:/reports",
           "code-review-mcp:latest"]
}
```

**After ANY code change:** rebuild the image, then **toggle the MCP server
off/on in LM Studio** — otherwise it keeps the old container (the #1 cause
of "the fix didn't take").

### Bare venv (for Claude Code / development)

```json
"code-review": {
  "command": "C:\\Projects\\Final-Project\\.venv\\Scripts\\python.exe",
  "args": ["-m", "code_review_mcp.server"],
  "env": {"LMSTUDIO_BASE_URL": "http://localhost:1234/v1",
          "REVIEW_ENABLE_BLOCKING": "1"}
}
```

The venv entry includes the Windows git-deadlock hardening (`_run_git`);
the Docker path avoids that problem class entirely.

---

## 9. Testing

| Layer | What | How |
|---|---|---|
| Unit (91 tests) | URL validation, JSON extraction, chunking, merging, all 3 modes, job lifecycle, long-poll, parallel dispatch, files filter, checkpoint/resume (deep + brief), markdown render, report retrieval, notes, private-repo token handling | `.venv\Scripts\python.exe -m pytest mcp-server/tests -q` — no network, no LLM |
| Dummy repo fixture | `mcp-server/tests/fixtures/dummy_repo` plants one instance of every surface-check class + a clean file + a trivial file + a filtered `.txt`; `mcp-server/tests/test_dummy_repo.py` runs the pipeline over it, including full crash→checkpoint→resume cycles for deep AND brief | part of the unit suite; also reused by the backend tests |
| Mock LLM | `mcp-server/tests/mock_llm_server.py` — stdlib OpenAI-compatible fake (`--delay` per chunk, `--stall-after` to simulate a wedge) | `scripts/run-mock-llm.ps1` (or `.sh`) starts it on port 8765 |
| E2E driver | `mcp-server/tests/e2e_review.py` — runs a REAL review (job thread, checkpoints, resume) against any endpoint, printing every progress line | mock endpoint for offline runs, LM Studio for real-hardware runs |
| Whole repository | Both suites together (**128 tests**) | `scripts/test.ps1` / `scripts/test.sh` |
| Kill-test drill | start an e2e review → kill the process mid-run → rerun the same command → must show `(+N restored)` and finish | verified on 2026-07-23 (batch 1/9 restored) |

Real-hardware verification of this version (Gemma 4 26B,
`google/gemma-4-26b-a4b-qat`): quick review of the 123-file e-voting test
repo in 58 s; scoped deep review of `settings.py` in 54 s finding the known
ground truth (hardcoded `SECRET_KEY` — critical, `DEBUG = True` — high).

---

## 10. Known limitations & operational gotchas

- **LM Studio tool-call timeout ≈ 60 s** (`-32001`), and it caps consecutive
  tool calls per turn — the job API + long-polling design exists because of
  this. If polling stops mid-review, the review continues server-side.
- **Small models pick tools by name** — the blocking tool stays hidden.
- **Serial LM Studio + concurrent requests = empty responses** — keep
  `REVIEW_PARALLEL=1` unless parallel request processing is enabled.
- **Windows memory pressure can SIGKILL the container** (WSL-VM OOM, exit
  137). Both deep and brief checkpoint, so recovery is always: same
  `review_id` + same mode → resume.
- **Hot-swapping models mid-review** is the typical OOM trigger; avoid
  loading a big model while a review runs.
- **On the 26b, deep mode can occasionally fail JSON on a file** after all
  retries → recorded as a per-file failure, run continues. The 12b is the
  more reliable deep reviewer; the 26b writes richer findings.
- **git for Windows deadlocks in console-less MCP servers** — handled by
  `_run_git` hardening in the venv path; irrelevant in Docker.
