# 16. User Guide — Driving `code-review-mcp` with Gemma 4 26B

**Project:** AI Scan Repos
**Document Version:** 1.0
**Audience:** anyone who wants AI code reviews from LM Studio's chat — no
Python knowledge needed. Everything is done by talking to Gemma.

> Companion reference: [15-mcp-server-documentation.md](15-mcp-server-documentation.md)
> (every tool and setting in detail) and [PROMPTS.md](../Code-Review-MCP/mcp-server/PROMPTS.md)
> (the paste-ready prompt library).

---

## 1. What you get

You paste a prompt into LM Studio's chat. Gemma clones the GitHub repo you
named (inside a Docker container), reviews the code, and hands you back a
verdict with findings — while writing two report files to your disk:

- `reports\<ID>.json` — machine-readable full report
- `reports\<ID>.md` — human-readable review document

Gemma plays **both roles**: in the chat it is the *operator* calling the
review tools; behind the scenes the server calls Gemma again over HTTP as
the *reviewer* that actually reads the code.

---

## 2. One-time setup

### 2.1 Prerequisites

1. **LM Studio** (0.3.18 or newer) with the model
   **`google/gemma-4-26b-a4b-qat`** downloaded.
2. **Docker Desktop** running.
3. The image built once (and after every code change):

```powershell
cd C:\Projects\Final-Project
docker build -t code-review-mcp .\mcp-server
```

> Setting this up on a machine that has never run the project? Start with
> [18-team-setup-guide.md](18-team-setup-guide.md) — it covers the whole
> repository, including how to work without a local GPU.

### 2.2 Start the LM Studio server

LM Studio → **Developer** tab → load `google/gemma-4-26b-a4b-qat` → **Start
Server** (port 1234). The green "Reachable at http://localhost:1234" line
must be visible.

### 2.3 Register the MCP server

LM Studio → **Program** tab (right sidebar) → `Install` → `Edit mcp.json`:

```json
{
  "mcpServers": {
    "code-review": {
      "command": "docker",
      "args": [
        "run", "--rm", "-i",
        "--add-host=host.docker.internal:host-gateway",
        "-v", "C:\\Projects\\Final-Project\\reports:/reports",
        "code-review-mcp:latest"
      ],
      "timeout": 300000
    }
  }
}
```

Save, then **toggle the server ON** in the Program tab. You should see the
`code-review` tools appear (start_review, get_review_status, get_report…).

> ⚠️ **After every `docker build`, toggle the MCP server OFF and ON again**
> in LM Studio. Otherwise it keeps the old container running and your
> changes silently don't apply.

---

## 3. Gemma 4 26B — what to expect

| Property | What it means for you |
|---|---|
| Model id | `google/gemma-4-26b-a4b-qat` — **always pin it** with `model "..."` in your prompts, because LM Studio routes an empty model id unpredictably when several models are loaded |
| Reasoning model | Gemma "thinks" in hidden tokens before answering. Long silences in the progress line (`thinking (512 tokens streamed)`) are normal and healthy |
| Quality | The strongest local reviewer in this setup — richest findings, best exploitability judgment |
| Speed (measured, 123-file repo) | quick ≈ 1 minute · brief ≈ 10–15 minutes · deep ≈ 1–5 minutes **per file** |
| Deep-mode quirk | Occasionally a file's review fails JSON validation after all retries → it is recorded under `failures` and the run continues. Rerunning with the same `review_id` re-reviews just the missing files |

**Tip:** for very large deep runs, `google/gemma-4-12b-qat` is faster and
more JSON-reliable; you can run brief with the 26b and deep with the 12b —
each `start_review` takes its own `model` argument.

---

## 4. Your first review (walkthrough)

Paste this into the chat (replace the repo URL if you like):

```
Run a brief review of https://github.com/hadsahar/Secured-e-voting-system
with start_review (review_id "TEST-01", model "google/gemma-4-26b-a4b-qat",
system prompt "Strict security reviewer"). Poll get_review_status with
wait_seconds 40 until done, printing the progress line each cycle and
following the "do" field exactly, then give me the verdict, the top
findings, and where the files were saved.
```

What you will see:

1. Gemma calls `start_review` → answers instantly with a `job_id`.
2. Gemma calls `get_review_status` in a loop. Each call waits up to 40 s
   server-side and returns one compact line, e.g.
   `44% — 4/9 batches done, 5 left, ~6m left (320s elapsed)`.
3. After the last batch, status flips to `done` and Gemma reads the report
   with `get_report` and summarizes it.
4. On your disk: `reports\TEST-01.json` and `reports\TEST-01.md`.

**Honesty note:** LM Studio limits how many tool calls Gemma may chain in
one turn. If Gemma stops polling, the review **keeps running server-side**.
Just say: *"call get_review_status for that job again"* — or come back
later and ask for `list_reports`.

---

## 5. The scan levels (cheat sheet)

The full paste-ready library with reviewer personas lives in
[PROMPTS.md](../Code-Review-MCP/mcp-server/PROMPTS.md). The short version:

| Level | Mode | Prompt skeleton |
|---|---|---|
| 0 Preflight | — | "Call check_llm and tell me if the endpoint is reachable and whether a job is running." |
| 1 Quick | `quick` | "Run a quick review of {REPO} … mode quick" — surface scan + one AI verdict, ~1 min |
| 2 Brief | `brief` | the walkthrough above — full-repo triage |
| 3 Deep targeted | `deep` + `files` | "…mode deep, files: [\"*/views.py\", \"settings.py\"]…" — the intended deep workflow |
| 4 Deep full | `deep` | every file, hours; checkpointed per file; only for final audits |

**The golden workflow:** Level 1 or 2 over everything → Level 3 on what got
flagged.

### Consuming a big report without blowing the context

For deep reports, tell Gemma to iterate instead of dumping:

```
Call get_report_entry for "TEST-01" starting at index 0. For each file,
append a 3-line digest to the notes with append_report_notes, then move to
the next index until remaining is 0. Finally call read_report_notes and
write me the final summary: verdict, criticals first with exploit impact,
fixes in priority order.
```

---

## 5a. Reviewing a PRIVATE repository

Private repos work exactly like public ones in chat — you pass the same
`https://github.com/owner/repo` URL. The difference is one-time setup.

### Set the token once

1. Create a GitHub personal access token:
   - **Classic PAT** → tick the `repo` scope, or
   - **Fine-grained PAT** → grant **Contents: Read** on the repos you scan.
   - Org repo with SAML SSO? Click **Authorize** for that organization on
     the token page afterwards.
2. Give it to the **server** (not to Gemma). Easiest with LM Studio — add an
   `env` block to your mcp.json entry and keep `-e GITHUB_TOKEN` in the args:

```json
{
  "mcpServers": {
    "code-review": {
      "command": "docker",
      "args": [
        "run", "--rm", "-i",
        "--add-host=host.docker.internal:host-gateway",
        "-v", "C:\\Projects\\Final-Project\\reports:/reports",
        "-e", "GITHUB_TOKEN",
        "code-review-mcp:latest"
      ],
      "env": { "GITHUB_TOKEN": "ghp_your_token_here" },
      "timeout": 300000
    }
  }
}
```

3. Toggle the MCP server off/on so it picks up the new env.

> 🔒 **Never type your token into the chat.** No tool accepts a token
> argument, on purpose: anything you put in chat goes into the model's
> context and LM Studio's logs. If Gemma ever asks you for a token,
> something is wrong — the server reads it from its own environment.
>
> Also note mcp.json stores the token in plain text on your disk. That is
> fine for a local course project; treat the file as a secret, and prefer a
> short-lived, read-only, minimally-scoped token.

### Check access before a long run

```
Call check_repo_access for https://github.com/myorg/my-private-repo
and tell me whether the server can clone it.
```

You get back `status: ok` with `authenticated: true` (the token works), or
`no_access` with a precise reason — no token configured, wrong scope,
expired, or SSO not authorized. This takes ~1 second; do it before kicking
off a review that would otherwise fail 40 minutes in.

### Then review as usual

```
Run a brief review of https://github.com/myorg/my-private-repo using
start_review with mode "brief", review_id "PRIV-01",
model "google/gemma-4-26b-a4b-qat", system prompt "Strict security
reviewer". Poll get_review_status until done, then give me the verdict.
```

The report records `authenticated: true` as provenance — proof the clone
used a credential — but never the token itself.

---

## 5b. Feeding deterministic scan results (the platform hand-off)

When the project's separate scanner component (Semgrep etc.) has produced
findings, you can hand them to the review so Gemma **verifies and enriches
them** instead of starting blind — this is the deterministic-first pipeline
from the platform design, at MCP scale.

1. Drop the findings JSON into the shared `reports\` folder, e.g.
   `reports\EVOTE.findings.json`:

```json
{"source": "semgrep",
 "findings": [
   {"file": "DjangoWebApp/src/secure_voting_system/settings.py",
    "rule": "django-secret-key-hardcoded", "severity": "high",
    "line": 22, "description": "Django SECRET_KEY hardcoded."}]}
```

2. Tell Gemma (note `findings_file`):

```
Run a deep review of {REPO} with start_review: mode "deep",
review_id "{ID}", model "google/gemma-4-26b-a4b-qat",
files: ["settings.py"], findings_file "EVOTE.findings.json",
system prompt "Strict security reviewer. For each supplied static finding:
verify it against the code, explain the real-world impact, and give a
concrete remediation." Poll get_review_status until done, then summarize:
findings confirmed by BOTH the scanner and you first, dismissed false
positives with the reasoning, then your own new findings.
```

What you get back in the report:

- every scanner finding attached **unchanged** (`static_findings`) — the
  model can never delete or downgrade them;
- `confirmed_by_llm: true` on findings Gemma independently verified — the
  highest-confidence tier;
- false positives dismissed *with reasoning* in the file summary;
- Gemma's own findings that no scanner rule can catch.

Verified live with the 26B: 2 real findings both confirmed, a planted fake
SQL-injection finding correctly dismissed ("standard path manipulation
using os.path"), and 2 new model-only findings added.

The `file` paths in the findings JSON must match the repo paths exactly
(run `ingest_repository_only` or check a previous report if unsure) —
findings for unknown paths are reported under `deterministic.unmatched`
rather than silently dropped.

---

## 6. Disaster recovery — when something crashes

The server checkpoints progress after **every completed file** (deep) or
**batch** (brief) into `reports\<ID>.partial.json`. This has been verified
by killing the process mid-review: nothing beyond the unit in flight is
ever lost.

### Symptoms and the one recovery move

| Symptom | Meaning |
|---|---|
| `get_review_status` says `unknown_job` and lists `resumable_checkpoints` | The container restarted (crash, OOM kill, LM Studio toggle) — the in-memory job list died, the checkpoint did not |
| Gemma claims to monitor but nothing advances | It stopped calling the tool (turn limit). The review is still running server-side |
| A `<ID>.partial.json` file sits in `reports\` | An interrupted review that can be resumed |

**Recovery is always the same:** run the *exact same* `start_review` again —
same `review_id`, same mode:

```
Run start_review again for https://github.com/owner/repo with review_id
"TEST-01", mode "brief", model "google/gemma-4-26b-a4b-qat", same system
prompt. It must RESUME from the checkpoint — the progress line will show
"(+N restored)". Poll get_review_status until done.
```

The resumed report carries `resumed: true` and `reused_batches` /
`reused_reviews`, and the checkpoint file is deleted on success. If the
repository's commit changed since the checkpoint, the server safely starts
fresh instead of mixing stale results.

---

## 7. Troubleshooting

| Problem | Cause → fix |
|---|---|
| `MCP error -32001` | A tool call exceeded LM Studio's per-call timeout. Should not happen with the job flow; add `"timeout": 300000` to the mcp.json entry and make sure the image was rebuilt and the server toggled off/on |
| `Repository not found` on a repo you can see in the browser | It is **private** and the server cloned anonymously. Set `GITHUB_TOKEN` (§5a) and toggle the server off/on. The error message itself tells you which case you are in |
| `Invalid username or token` | Token is wrong, expired, or lacks scope. Classic PAT needs `repo`; fine-grained needs Contents: Read. Verify with `check_repo_access` |
| Private org repo still refused with a valid token | SAML SSO — authorize the token for that organization on GitHub's token page |
| Token seems ignored | The env didn't reach the container: keep `-e GITHUB_TOKEN` in the docker args, set the value in mcp.json `env` (or system-wide), then toggle the server off/on |
| "The fix didn't take" after a rebuild | LM Studio is still running the **old** container — toggle the MCP server off/on in the Program tab |
| `Cannot reach LLM endpoint` | LM Studio server not started, or wrong port. From inside Docker the host is `host.docker.internal`, never `localhost` |
| `Model stalled: no output for 300s` | The model is wedged or was unloaded mid-run — reload it in the Developer tab and resume with the same `review_id` |
| Review died, `unknown_job` | See §6 — resume with the same `review_id` + mode |
| Empty responses / failed JSON on many files at once | `REVIEW_PARALLEL` was raised without enabling LM Studio's parallel request processing — set it back to 1 |
| Container killed with exit 137 (`docker events`) | Windows/WSL memory pressure — usually from loading another big model mid-review. Resume as in §6; avoid model swaps during runs |
| A single file under `failures` in a deep report | The 26b ran out of JSON luck on that file — rerun with the same `review_id` (only missing files are re-reviewed) or use the 12b for that file |

---

## 8. Do & don't (learned the hard way)

- **Do** pin `model "google/gemma-4-26b-a4b-qat"` in every prompt.
- **Do** use `review_id` for every run — it names the reports AND enables
  resume.
- **Do** let quick/brief pick your deep targets; whole-repo deep on the 26b
  takes hours.
- **Don't** load or swap models while a review is running (OOM-kill risk).
- **Don't** raise `REVIEW_PARALLEL` on a serial LM Studio.
- **Don't** worry when the progress line sits on "thinking" — reasoning
  tokens stream invisibly and count as liveness; a real stall errors out on
  its own after 300 s.
