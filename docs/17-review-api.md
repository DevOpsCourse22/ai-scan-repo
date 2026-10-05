# 17. Review API — Flask Backend & Demo Console

**Project:** AI Scan Repos
**Document Version:** 1.0
**Status:** Current
**Component:** `review_api` — HTTP control plane over the review pipeline

> Related: [18-team-setup-guide.md](18-team-setup-guide.md) (setting this up
> on a new machine) · [15-mcp-server-documentation.md](15-mcp-server-documentation.md)
> (the MCP front end and the pipeline itself) ·
> [14-mcp-adaptation-plan.md](14-mcp-adaptation-plan.md) (this is WS3, the
> control-plane alignment step) ·
> [07-api-specification.md](07-api-specification.md) (the platform API this
> mirrors in miniature).

**Where it lives:** `backend/` in the repository (image `review-api`); the
pipeline it calls is in `mcp-server/`. It is **API only** — the test console
is a separate microservice, `frontend/` (image `review-console`), which
reverse-proxies `/api/` to it.

---

## 1. What it is and how it works

A Flask service that exposes AI code review over HTTP. It accepts a review
request, consolidates everything into one server-owned prompt, runs the
review in the background against either an **on-prem** or a **cloud** LLM,
and lets a client follow progress, restart a failed run, and fetch the
report.

```
client (demo UI / CI / curl)
   |  REST /api/v1
   v
Flask app  ── config.py ── providers: on-prem (LM Studio/Ollama/vLLM)
   |                                  cloud   (OpenAI/Groq/Azure)
   |  prompts.py  consolidate request -> ONE system prompt
   |  jobs.py     job store + worker thread + per-review credential
   v
code_review_mcp.pipeline.run_review()      <-- shared with the MCP server
   clone -> manifest -> review (quick|brief|deep) -> checkpoint -> report
   v
reports/<review_id>.json + .md   (+ <review_id>.partial.json while running)
```

**It shares the pipeline with the MCP server rather than reimplementing it.**
`pipeline.run_review()` is the single owner of clone, findings intake,
checkpointing, resume, and report writing; the Flask API and the MCP tools
are two thin front ends over it. That means crash recovery behaves
identically no matter which door a review came through — and a review
started by one can be resumed by the other, because both read the same
`reports/` directory.

**Relation to the platform.** This is the local/dev control plane. It
deliberately stops short of the platform's durable PostgreSQL state machine,
OIDC identity, and per-repository authorization
[04-low-level-design, 05-security-authorization-design]; see §8.

---

## 2. Quick start

```powershell
# Windows — does everything below and runs the tests to prove it
powershell -ExecutionPolicy Bypass -File scripts\setup.ps1
powershell -ExecutionPolicy Bypass -File scripts\run-api.ps1
```

By hand, or on Linux/macOS:

```bash
# Install BOTH components together: code-review-mcp is not on PyPI, so the
# backend's dependency on it can only be satisfied from this checkout.
pip install -e './mcp-server[dev]' -e './backend[dev]'
cp backend/config/api.example.yaml backend/config/api.yaml
cp .env.example .env
python -m review_api.app --config backend/config/api.yaml
```

Or in a container, from the repository root:

```bash
docker compose up --build
```

Then open **http://127.0.0.1:8090/** for the test console (compose runs it
next to the API), or call the API directly — its root returns a small JSON
descriptor, not a page:

```bash
curl -s -X POST http://127.0.0.1:8000/api/v1/reviews -H "Content-Type: application/json" -d "{\"review_id\":\"DEMO-01\",\"repo_url\":\"https://github.com/owner/repo\",\"mode\":\"brief\",\"persona\":\"security\"}"
```

---

## 3. Endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/v1/health` | Liveness. `?probe=1[&provider=x]` also asks the LLM endpoint for its model list — proves reachability of cloud *or* on-prem. |
| GET | `/api/v1/config` | The effective configuration, secrets stripped. |
| GET | `/api/v1/providers` | Configured LLM endpoints with `location: on_prem\|cloud` and whether each key is set. |
| GET | `/api/v1/models` | Models a provider — or a `?endpoint=…` — actually serves. Full list; this is what fills the model dropdown. |
| GET | `/api/v1/personas` | Named reviewer personas. |
| GET | `/api/v1/repo-access` | `?repo_url=…[&branch=…]` — can we clone it? Accepts `X-Repo-Token`. Seconds, no clone. |
| POST | `/api/v1/reviews` | **Create** a review (§4). `dry_run: true` returns the prompt without calling the model. |
| GET | `/api/v1/reviews` | List reviews (`?status=`, `?limit=`). |
| GET | `/api/v1/reviews/{id}` | **Progress** (§6). Accepts a `review_id` or a `job_id`. |
| GET | `/api/v1/reviews/{id}/prompt` | The consolidated prompt that was used. |
| POST | `/api/v1/reviews/{id}/restart` | **Restart** a failed/cancelled review — resumes from the checkpoint (§7). Same as `/resume` in kubernetes mode or when the review is no longer in memory. |
| POST | `/api/v1/reviews/{id}/pause` | **Pause** a running review (§7a). Kubernetes mode stops at once; local mode after the unit in flight. Survives a restart of this service. |
| POST | `/api/v1/reviews/{id}/resume` | **Resume** a paused, failed or interrupted review from its checkpoint, pinned to the checkpoint's commit (§7a). Accepts `X-Repo-Token`. |
| DELETE | `/api/v1/reviews/{id}` | **Delete a review completely** (§7b): report, checkpoint, scan results, pause record, progress files, and the in-memory entry. `409` while it runs. |
| GET | `/api/v1/llm/presets` | The AI services Settings offers (§8a), plus the server's default provider. |
| POST | `/api/v1/llm/models` | Test an AI service choice and list its models (§8a). POST because the body carries an API key. |
| POST | `/api/v1/reviews/{id}/cancel` | Stop after the unit in flight, keeping the checkpoint complete. |
| GET | `/api/v1/reviews/{id}/report` | Report: `?view=summary` (default) `\|full\|markdown`, or `?file=path`. |
| GET | `/api/v1/reports` | **All** reports on disk, newest first, with repo/commit/mode/counts and links, plus **resumable checkpoints**. `?limit=` |
| GET | `/api/v1/reports/{name}` | **One** report by name. Same `?view=` / `?file=` as the review-scoped route. |
| DELETE | `/api/v1/reports/{name}` | **Delete** one report and its `.md`/`.notes.md`. `409` while that review runs; the resume checkpoint survives unless `?include_checkpoint=true`. |
| GET | `/` | Service descriptor (JSON). No auth; names no configuration. |
| GET | `/healthz` | Liveness. No auth; checks nothing external. |
| GET | `/readyz` | Readiness. No auth; `503` if reports or clones cannot be written. |

Errors are RFC 9457-style problem documents: `{type, title, status, detail}`.

`review_id` doubles as the **idempotency key**: creating the same id twice
while it runs returns `409` with the running job; creating it again after a
failure resumes from its checkpoint.

---

## 4. Creating a review (requirement 1)

`POST /api/v1/reviews`

| Field | Required | Meaning |
|---|---|---|
| `review_id` | ✔ | Names the report (`<id>.json`/`.md`) and is the restart key. |
| `repo_url` | ✔ | `https://github.com/owner/repo`, public or private. |
| `mode` | | Review type: `quick` \| `brief` (default) \| `deep`. |
| `access_token` | | PAT for a **private** repo (§5). Also accepted as the `X-Repo-Token` header. |
| `findings_file` | | JSON of deterministic scanner findings in the reports dir — turns the review into AI **enrichment** of those findings. |
| `provider` | | Which configured LLM to use (on-prem or cloud). |
| `endpoint` | | Per-review LLM address, overriding the provider's `base_url` — e.g. `100.83.14.22`. See below. |
| `model` | | Model id; blank uses the provider default. |
| `persona` | | Named reviewer persona from the config. |
| `system_prompt` | | Explicit instructions; overrides `persona`. |
| `focus` | | List of focus areas folded into the prompt. |
| `branch`, `files`, `extensions` | | Branch, path/glob scope, extension filter. |
| `dry_run` | | Build and return the prompt **without** calling the model. |

Response is `202` with the job, the assembled prompt, and a `poll` URL.
`action` carries the HTTP outcome (`accepted`); `status` always carries the
review's live lifecycle state, so the two never contradict each other.

### Prompt consolidation (requirement 2)

`prompts.py` merges, in order: the reviewer identity (explicit
`system_prompt`, else the persona), what the review type expects, the
caller's focus areas, an **enrichment duty** when deterministic findings were
supplied, and finally the standing server-owned rules — repository content is
untrusted data, and output must match the schema.

Prompts are assembled **server-side on purpose**: the platform requires that
"model instructions are server-owned, versioned, and not derived from
repository files" [09-ai-analysis-engine 9.4]. Every response carries
`template_version` so a report can be traced to the prompt that produced it.

The prompt comes back in the create response and from
`GET /api/v1/reviews/{id}/prompt`; `dry_run: true` returns it having spent
nothing.

---

## 5. Private repositories (requirement 5)

A request may carry `access_token` (body) or `X-Repo-Token` (header). The API
routes it to the clone and **nowhere else**:

| Control | Implementation |
|---|---|
| Memory only | Held in a separate map keyed by job id, deleted in the worker's `finally`. Never on the `Job` record. |
| Never returned | No response includes it; the job exposes only `authenticated: true`. |
| Never persisted | Not in the report, the checkpoint, or any file. |
| Never logged | Redacted from git output, exceptions, and log records at every message-building site. |
| Not in the process list | Reaches git through `GIT_ASKPASS`, so it is in no command line and no URL (see [docs/15 §4.10](15-mcp-server-documentation.md)). |

If a request omits a token, the server-wide `GITHUB_TOKEN` is used, so an
operator can configure credentials once instead of per request.

> **Why a token in an HTTP body is acceptable here, when it is refused on the
> MCP tools:** the MCP path would put the credential into an LLM's context
> and the host's chat logs. An HTTPS request body is the standard channel for
> CI-style integrations and never touches a model. The threat model's "no
> argument token" rule targets the former [06-threat-model].
>
> Practically: use short-lived, read-only, minimally-scoped tokens; run the
> API behind TLS if it is not on localhost; and set `api_key` (§9).

---

## 6. Progress (requirement 3)

`GET /api/v1/reviews/{id}` returns the state machine plus a printable line:

```json
{
  "review_id": "DEMO-01", "status": "reviewing",
  "done": 4, "total": 9, "remaining": 5, "percent": 44,
  "elapsed_seconds": 220, "eta_seconds": 275, "restored_units": 1,
  "attempts": 2, "commit": "5a164c3e",
  "progress": "44% — 4/9 batches done, 5 left, ~4m 35s left (220s elapsed) (+1 restored)",
  "restartable": false
}
```

Lifecycle: `requested → cloning → reviewing → completed | failed |
cancelled`. `done` counts **completed** units only (batches in quick/brief,
files in deep) and the ETA is recomputed only when a unit finishes, so it
stays steady instead of ballooning mid-unit.

If the API restarted and lost its in-memory registry, the status is rebuilt
from the reports volume, marked `"source": "disk"`: `completed` (the report
exists), `paused` (`<id>.paused.json`), a running status with
`"detached": true` (a Job this process no longer tracks is still writing
`progress.json` - not resumable until it stops), or `interrupted` (a
checkpoint, plan or error file survived; resumable). Only a review with
none of those files is a `404`, which still lists **resumable checkpoints**.

---

## 7. Restart after a failed scan (requirement 4)

Because the pipeline checkpoints after **every completed unit**, a failure
never costs more than the unit in flight.

```
POST /api/v1/reviews/{id}/restart        # or POST /reviews with the same review_id
```

The pipeline finds `<review_id>.partial.json`, verifies it belongs to the
same repo **and commit**, reuses the completed units, and reviews only what
is missing. Watch `restored_units` and `attempts`; the finished report carries
`resumed: true` with `reused_batches`/`reused_reviews`. A changed commit
safely starts fresh instead of mixing stale results. On success the checkpoint
is deleted.

Cancellation is honoured **between** units precisely so a cancelled review is
always restartable.

## 7a. Pause and resume (added 2026-10-01)

```
POST /api/v1/reviews/{id}/pause      # 202 {"action": "paused"|"pausing"}; 200 if already paused
POST /api/v1/reviews/{id}/resume     # 202 {"action": "resumed", "restored_units": N, "pinned_commit": "…"}
```

- **Pause** frees the engine for other reviews. Kubernetes mode deletes the
  Job (the checkpoint already holds every finished unit, so at most the unit
  in flight is redone); local mode stops after the unit in flight
  (`pausing: true` until then). Either way `<id>.paused.json` is written to
  the reports volume with repo, branch, mode, commit, model and provider -
  never a token - so the review still reads as `paused` after a restart.
- **Resume** needs no job in memory. It takes repo, commit and mode from the
  checkpoint first (they are what the checkpoint is matched against), then
  the pause marker, then the request body - the gateway sends what it stored,
  for a review paused before its first checkpoint. The new run is **pinned
  to that commit** (`COMMIT`, contract v3), so a branch that moved on while
  the review was paused cannot make it start over. A fresh `X-Repo-Token`
  comes with the request.
- `409` when the review is running, finished, or still being written by a
  detached Job; `409 {"action": "busy"}` when another review holds the
  engine.

**Verified live** (2026-07-26): a brief review of the 123-file test repo was
started through the API, the **API process was killed** at batch 1/9; on
restart the lost review appeared under `resumable_checkpoints`, resubmitting
the same `review_id` reported `(+1 restored)` and reviewed only the remaining
batches; a later cancel stopped at 3/9 with `restartable: true` and the
checkpoint intact; `restart` then resumed with `(+3 restored)`.

### 7b. Resume skips the scanners, and delete (added 2026-10-01)

The deterministic scan (Semgrep, Gitleaks, Trivy) writes
`<id>.findings.json` stamped with `scan_context` (`repo_url`, `commit`,
`scanned_at`). On resume the engine still clones (the AI needs the code), but
when the stamp matches the pinned commit and repo it reuses those findings
instead of scanning again: the status goes `cloning` → `scanning`
(`scan_reused: true`, a few milliseconds) → `reviewing`. A missing or
mismatched stamp scans normally.

`DELETE /api/v1/reviews/{id}` removes every file the review left on the
reports volume and forgets it. It is refused (`409`) while the review is
active or while an untracked Job is still updating its progress (younger
than the stale limit): pause first. The gateway's `DELETE /reviews/{id}`
stops a running review first, then calls this.

---

## 8. Cloud and on-prem LLMs

A **provider** is a named LLM endpoint in the config. One request field
(`provider`) chooses where the review runs, so the same API serves a local
model or a hosted one with no code change.

`kind: openai_compatible` covers LM Studio, Ollama, vLLM, **Google AI
Studio**, OpenAI, Groq, Together, and OpenAI-compatible Azure deployments —
one client, many backends. `location: on_prem | cloud` is advisory but
deliberate: it makes visible, in `/api/v1/providers` and in the UI, whether a
review would send code off the premises.

**Keys never live in the config file.** A provider names the *environment
variable* holding its key (`api_key_env: GOOGLE_API_KEY`), resolved at call
time, so the config is safe to commit and `/api/v1/config` can report
`api_key_configured` without exposing anything.

### 8a. The site-wide AI service (Settings, added 2026-10-01)

A site admin picks one AI service in the website's Settings: `local`
(LM Studio, Ollama or vLLM on a private or Tailscale address) or a cloud
preset (`openai`, `anthropic`, `gemini`, `groq`, `openrouter`, `mistral`),
plus a model and an API key. The gateway stores it (key encrypted) and sends
it with every create and resume as an `llm` object
`{"preset", "endpoint", "model"}`, with the key in the `X-LLM-Key` header.

- Cloud addresses are fixed in `review_api/llm_presets.py`; the setting is a
  preset **name**, so nobody can point review-api at an arbitrary host by
  editing a setting. `local` goes through the private-network check.
- The key is never written to disk by review-api. Local mode keeps it in
  memory for the run; Kubernetes mode puts it in the per-attempt Secret as
  `llm_api_key`, which the worker reads as `LLM_API_KEY`
  (`docs/job-container-contract.md` v4).
- Without an `llm` object (no admin choice yet) the configured default
  provider is used, exactly as before.

### Choosing a model

`GET /api/v1/models[?provider=…][&endpoint=…]` returns what that endpoint
actually serves, and the console renders it as a dropdown. Guessing a model id
is the most common way to lose a review to a 404 fifteen minutes in, so the
picker only offers ids the endpoint reported.

```json
{ "reachable": true, "model_count": 4, "default_model": "google/gemma-4-26b-a4b-qat",
  "models": ["google/gemma-4-12b-qat", "google/gemma-4-26b-a4b-qat", "…"],
  "base_url": "http://100.83.14.22:1234/v1" }
```

The list is refreshed when you connect, when the provider changes, after
testing an endpoint, and from the **↻** button. It is sorted, so the order
does not depend on how the upstream server happened to enumerate them.

Two deliberate choices:

- **The full list, unlike the health probe.** `/api/v1/health?probe=1` trims
  to 25 because it is a liveness payload; a picker must show everything, and a
  cloud provider can offer dozens.
- **An unreachable endpoint is `200`, not an error.** It returns
  `reachable: false`, an empty `models`, the provider's `default_model`, and a
  hint. Being down is upstream state, not a malformed request — and the picker
  needs to degrade to *"other — type an id"* rather than render a failure and
  strand the user. The dropdown always keeps that option plus *(provider
  default)*, so no endpoint state can leave a model unselectable.

### Naming an endpoint per request

A configured provider covers the endpoints a team uses regularly. For the
one-off — *"run this against the machine at 100.83.14.22"* — a request may
carry its own address, and the demo console has a field and a **Test** button
for exactly that. It needs no config change and no restart.

```jsonc
POST /api/v1/reviews
{ "review_id": "R-1", "repo_url": "…", "endpoint": "100.83.14.22" }
```

Anything a person would plausibly type is accepted and normalised, so the
caller does not have to know the URL shape:

| Typed | Used |
|---|---|
| `100.83.14.22` | `http://100.83.14.22:1234/v1` |
| `100.83.14.22:1234` | `http://100.83.14.22:1234/v1` |
| `http://100.83.14.22` | `http://100.83.14.22:1234/v1` |
| `gpu-box.tailnet.ts.net:1235` | `http://gpu-box.tailnet.ts.net:1235/v1` |
| `https://gpu.internal/openai/v1` | unchanged — an explicit path is respected |

Port `1234` and the `/v1` suffix are filled in because that is what LM Studio
serves. The resolved address comes back as `endpoint` on the job, appears in
the status payload, and is shown in the console next to the provider.

Test it before spending a review on it:

```
GET /api/v1/health?probe=1&endpoint=100.83.14.22
```

That returns the model list the endpoint actually offers — which the console
then uses to populate the model box, so nobody has to guess an id.

#### Why the address is restricted — and to what

**Accepting a URL from a request body is an SSRF surface.** Whatever address
arrives decides where the server sends repository source code, and this API is
unauthenticated and CORS-open by default: a page in a developer's browser can
reach `localhost:8000`. Unrestricted, the field would be both an exfiltration
channel and a pivot into anything the host can reach.

So a supplied endpoint must resolve into one of
`custom_endpoint_networks`, which defaults to loopback, RFC 1918 private
ranges, IPv6 ULA, and **`100.64.0.0/10` — the Tailscale range**. That is
exactly the intended use ("a model on a machine my team owns"), and it costs
nothing, because reaching a hosted model is what the configured cloud
*providers* are for.

**Link-local is deliberately excluded.** `169.254.169.254` is the cloud
instance metadata endpoint; steering a server at it from a request body is the
textbook SSRF credential theft.

Rejection happens at request time, not mid-review, and the message names the
address, what it resolved to, and the allowed ranges. Two further controls:
`allow_custom_endpoint: false` turns the feature off entirely, and credentials
embedded in the URL are refused outright.

> **Honest limit:** the hostname is resolved once for the check and again by
> the HTTP client, so a hostile DNS server could answer differently each time.
> Use literal IPs where that matters.

### Sharing one GPU across the team — `lmstudio_tailscale`

Between "everyone needs a GPU" and "send the code to a cloud vendor" sits a
third option: **one machine runs the model, the rest reach it over
Tailscale.** Tailscale is a private mesh network, so the endpoint is reachable
by tailnet members and by nobody else — no port forwarding, nothing published.
The provider stays `location: on_prem`, correctly: the code never leaves
machines the team owns.

The base URL is assembled from a single per-workstation value:

```yaml
base_url: http://${TAILSCALE_LMSTUDIO_HOST:-tailscale-host-not-set}:${TAILSCALE_LMSTUDIO_PORT:-1234}/v1
```

so a teammate changes one line in `.env` and nothing in the committed config.
Both sides of the setup — including the two steps people always miss, LM
Studio's *Serve on Local Network* toggle and a Tailscale-scoped firewall rule
— are in [18-team-setup-guide §4c](18-team-setup-guide.md).

**`setup_hint`.** This provider needs a value only a human can supply, and
without it the only symptom would be a connection timeout against a
placeholder host. Providers may therefore declare a `setup_hint`, which is
returned by `/api/v1/providers`, rendered in the demo console, and used in
place of the generic advice when a probe fails. The unset case deliberately
degrades to a wrong-looking URL plus that hint rather than a `ConfigError`,
because every provider in the file is parsed on every load — a fallback-less
`${VAR}` here would stop the API for everyone using a local model.

> LM Studio has **no authentication**. Tailnet membership is the access
> control, which is appropriate for a project team and not for a shared or
> public tailnet.

### The cloud path for contributors without a GPU

`google_ai_studio` exists specifically so the project can be run and
demonstrated by someone who does not have the on-prem workstation. Google AI
Studio serves **Gemma** over an OpenAI-compatible endpoint, so it is the same
model family as the local default, reached through the same client, selected
by one request field:

```json
{"review_id": "CLOUD-01", "repo_url": "…", "provider": "google_ai_studio"}
```

Set `GOOGLE_API_KEY` in `.env` (key from <https://aistudio.google.com/apikey>)
and confirm with `GET /api/v1/health?probe=1&provider=google_ai_studio`, which
returns the model list that key can actually see — use one of those ids for
`GOOGLE_MODEL`, rather than trusting a name written in a config comment.

Setup details are in [18-team-setup-guide §4](18-team-setup-guide.md).

Known limit: providers with a non-OpenAI wire format (Anthropic's native API,
AWS Bedrock) need a new adapter kind; the platform's AI Provider Adapter
covers that [09-ai-analysis-engine 9.5].

---

## 9. Configuration (requirement 6)

One file — `backend/config/api.yaml` (or `.json`; YAML needs PyYAML) — found
via `--config`, `REVIEW_API_CONFIG`, or the `config/` convention. Running with
no config file works: the defaults describe a local LM Studio.

| Key | Purpose |
|---|---|
| `host`, `port`, `debug` | Where the API listens. Defaults to **127.0.0.1**. |
| `output_dir` | Reports/checkpoints/findings dir — share it with the MCP server. |
| `api_key` | When set, every `/api/v1/*` call must send `X-API-Key`. |
| `cors_origins` | Browser origins allowed to call the API. |
| `github_token_env` | Env var used for private clones when a request has no token. |
| `default_provider`, `providers` | The LLM registry (§8). Each provider may declare `setup_hint` — what a human must do to make that endpoint work. |
| `default_mode` | `quick` \| `brief` \| `deep`. |
| `default_persona`, `personas` | Named reviewer personas. |
| `review_defaults` | Pipeline knobs mirroring the `REVIEW_*` env vars. |

Environment overrides (for Docker/CI without editing the file):
`REVIEW_API_HOST`, `REVIEW_API_PORT`, `REVIEW_API_DEBUG`,
`REVIEW_API_OUTPUT_DIR`,
`REVIEW_API_DEFAULT_PROVIDER`, `REVIEW_API_DEFAULT_MODE`,
`REVIEW_API_DEFAULT_PERSONA`, `REVIEW_API_KEY`, `REVIEW_API_CORS_ORIGINS`,
`REVIEW_API_CONFIG`.

`frontend_dir`, from before the console became its own service, is ignored
with a warning rather than rejected, so older copies of the config still
load. Any variable whose value looks like `tcp://…` is ignored too: that is a
Kubernetes service-link variable (a Service named `review-api` injects
`REVIEW_API_PORT=tcp://<ip>:8000`), not a setting — and it crash-looped the
first in-cluster deployment until this was handled.

A typo fails at **startup**, not mid-review: unknown keys, providers without
`base_url`, and an unresolvable default provider all raise `ConfigError`.

### One config file, every workstation

The config is committed and identical for everyone; only `.env` differs.
Two mechanisms make that work:

- **`${VAR}` / `${VAR:-fallback}` interpolation.** Any string value is
  expanded from the environment as the file is read, so
  `base_url: ${LMSTUDIO_BASE_URL:-http://localhost:1234/v1}` is the laptop
  default *and* the container override (`host.docker.internal`) *and* the CI
  value, without a second file. Expansion returns strings, so declared types
  (`port` is an `int`) are restored afterwards.
- **Relative paths resolve against the config file's directory**, not the
  process working directory, so `output_dir: ../../reports` is the
  repository's `reports/` folder whether the API was launched from the root
  or from `backend/`.

A `${VAR}` with **no** fallback that is unset raises `ConfigError` naming the
variable and the exact config key, on the same principle as the rest of this
section: fail at startup, not mid-review.

`main()` also loads the repository's `.env` before parsing the config, so a
direct `python -m review_api.app` picks up the same file Docker Compose reads.
It is loaded only from the CLI entry point, and never overrides a variable
that is already set in the real environment.

### Security posture — read this before exposing it

- **No user authentication.** `api_key` is a single shared secret, not
  identity. OIDC and per-repository authorization are platform work.
- **Defaults to localhost.** Binding to `0.0.0.0` exposes an endpoint that
  clones repositories and accepts tokens — put TLS and the API key in front
  of it first.
- **One review at a time** by design (a single local model would only queue
  against itself); a second request gets `409 busy`.
- **In-memory job registry.** An API restart loses job records — but not
  checkpoints, so work is recoverable (§7).

---

## 10. The test console (a separate service)

The console is **not part of this API**. It is its own microservice —
[`frontend/`](../Code-Review-MCP/frontend), image `review-console` — and this API serves no
HTML or static files at all. The split was made for the final stage: the
product deploys the API alone, and the console is a debugging add-on that
can be applied, used and removed without touching it.

It is an unprivileged nginx that serves one page and reverse-proxies `/api/`
to `REVIEW_API_URL` (default `http://review-api:8000`). The proxy is the
point: inside Kubernetes a browser cannot resolve a ClusterIP Service name,
but it can reach the console through `kubectl port-forward`, and the console
can reach the API — so the console tests the API, and the API's model
connection, *from inside the cluster*. Every call stays same-origin, so no
CORS is involved; the page reads `/console.json` and names the upstream it
is testing in its header.

The page itself exercises every endpoint: provider/persona discovery,
repo-access check, prompt-only dry run, review start, live polling, restart,
cancel, report retrieval, and **§4 Saved reports** — list, open and delete
reports, with deletion confirmed and a separate warning when the resume
checkpoint would go too. Rows are built with `createElement`/`textContent`,
never `innerHTML`, and the table scrolls inside its own box.

It still works without a proxy: opened from `file://`, its *API base* box
defaults to `http://localhost:8000` and can point at any API, with an
`X-API-Key` box beside it. The access-token field is a password input,
cleared when a review starts and never stored.

Build, run, security and editing notes: [`frontend/README.md`](../Code-Review-MCP/frontend/README.md).
Deploying both to Kubernetes: [docs/21](21-kubernetes-manifests-guide.md).

---

## 11. Testing

```powershell
scripts\test.ps1        # both suites: 128 tests
# or just this component:
.venv\Scripts\python.exe -m pytest backend/tests -q
```

`backend/tests/test_review_api.py` covers config loading, `${VAR}` expansion
and env overrides, config-relative path resolution, frontend discovery and
its absence, provider routing for cloud vs on-prem, prompt consolidation and
`dry_run`, the review lifecycle and progress fields, findings-file
validation, one-review-at-a-time, checkpoint discovery after a restart, the
failure → restart → resume cycle against the *real* batching pipeline, API
key enforcement, a guard that the shipped example config stays valid and
keeps offering a keyless-file cloud path, and — most importantly — that a
supplied `access_token` reaches the clone and appears in **no** response, job
record, report, or log.

The suite drives the real pipeline with only the clone and the LLM call
faked, reusing the MCP component's planted-issue fixture repository rather
than keeping a second copy (see `backend/tests/conftest.py`).
