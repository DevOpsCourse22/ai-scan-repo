# 18. Team Setup Guide — running this project on your own machine

**Project:** AI Scan Repos
**Document Version:** 1.0
**Status:** Current
**Audience:** anyone other than the person who built it

> Related: [15-mcp-server-documentation.md](15-mcp-server-documentation.md)
> (the MCP server) · [17-review-api.md](17-review-api.md) (the HTTP API) ·
> [16-gemma-user-guide.md](16-gemma-user-guide.md) (driving it from LM Studio)

This guide takes a machine that has never seen the project to a finished code
review. **You do not need a GPU, a local model, or access to anyone else's
workstation** — §4 covers the cloud path and §8 covers running with no model
at all.

---

## 1. What you are installing

Three components in one repository, over one shared review engine:

| Folder | What it is | Do you need it? |
|---|---|---|
| `mcp-server/` | MCP server for LLM hosts (LM Studio, Claude Code), **and the review pipeline both front ends use** | Always installed — it is the engine. Only register it with an MCP host if you want the chat workflow |
| `backend/` | The review API (`review-api`) — API only, no UI | Yes, for the demo and for anything scripted |
| `frontend/` | The test console (`review-console`) — a page behind an nginx proxy | Yes, it is how you see everything working. Compose runs it on port **8090** |

The important structural fact: `backend/` **depends on** `mcp-server/`. The
review engine (`code_review_mcp/pipeline.py`) lives in `mcp-server/` and the
API calls it. That is why both are installed together, and why a review
started through one can be resumed through the other.

---

## 2. Prerequisites

| Need | Version | Check with |
|---|---|---|
| **git** | any recent | `git --version` |
| **Python** | 3.10 or newer | `python --version` / `py -3 --version` |
| **Docker** | any recent | `docker --version` — *only* for the container path |

You need **either** Python **or** Docker, not both. Docker is fewer steps;
Python is easier to poke at.

An LLM is *not* a prerequisite — see §4.

---

## 3. Get the code and install

```bash
git clone <the repository URL>
cd Final-Project
```

### Path A — Docker (fewest steps)

```bash
cp .env.example .env          # Windows: copy .env.example .env
docker compose up --build
```

First build takes a few minutes. Then open <http://localhost:8000>.

### Path B — Python

```powershell
# Windows
powershell -ExecutionPolicy Bypass -File scripts\setup.ps1
```

```bash
# Linux / macOS
bash scripts/setup.sh
```

The setup script creates `.venv`, installs both components, copies
`.env.example` → `.env` and `backend/config/api.example.yaml` →
`backend/config/api.yaml` (it never overwrites files you have edited), and
runs the test suite. **If the tests pass, your install is correct** — stop and
read the output if they do not.

Then start the API:

```powershell
powershell -ExecutionPolicy Bypass -File scripts\run-api.ps1   # Windows
bash scripts/run-api.sh                                        # Linux / macOS
```

<details>
<summary>Doing it by hand instead of with the script</summary>

```bash
python -m venv .venv
.venv/bin/python -m pip install -e './mcp-server[dev]' -e './backend[dev]'
cp .env.example .env
cp backend/config/api.example.yaml backend/config/api.yaml
.venv/bin/python -m review_api.app --config backend/config/api.yaml
```

Install **both** packages in one `pip install`. `code-review-mcp` is not on
PyPI, so installing `backend` on its own makes pip go looking for it there
and fail. Passing both local paths together lets pip resolve it from the
checkout.
</details>

---

## 4. Choose which LLM answers reviews

This is the only decision that differs between machines, and it lives in one
file: **`.env`** in the repository root (gitignored — your keys stay yours).

### 4a. You have no GPU and no local model → Google AI Studio

Google AI Studio serves **Gemma over an OpenAI-compatible endpoint**, so you
get the same model family as the project's on-prem default, in the cloud, with
no code change.

1. Create an API key: <https://aistudio.google.com/apikey>
2. In `.env`:

   ```ini
   REVIEW_API_DEFAULT_PROVIDER=google_ai_studio
   GOOGLE_API_KEY=your-key-here
   ```

3. Restart the API, then ask it which models your key can actually use:

   ```
   http://localhost:8000/api/v1/health?probe=1&provider=google_ai_studio
   ```

   That returns the live model list for **your** key and region. Pick a Gemma
   id from it and put it in `.env` as `GOOGLE_MODEL=…` (or send `model` per
   request). Trust the probe over any model name written in a config comment.

> **This sends repository source code to Google.** That is a deliberate,
> visible choice: every provider is labelled `on_prem` or `cloud` in
> `/api/v1/providers` and in the console. Use public or your own repositories
> while evaluating.

### 4b. You have LM Studio locally

Nothing to change — it is the default. Load a model, then Developer tab →
Start Server (port 1234).

One catch: **from inside a container, the host is not `localhost`.** The
compose file already sets `LMSTUDIO_BASE_URL=http://host.docker.internal:1234/v1`
for you. If you run the API natively, `http://localhost:1234/v1` is correct.

### 4c. Use the GPU machine over Tailscale (shared local model)

The middle path between "buy a GPU" and "send code to the cloud": **one
person runs LM Studio, everyone else uses it over Tailscale.** The code never
leaves machines you own, and nothing is exposed to the public internet —
Tailscale is a private network between your devices, so only members of the
tailnet can reach the endpoint.

There are two sides. Do them in order.

---

#### Side 1 — on the machine that has the GPU (do this once)

**1. Install Tailscale and sign in.** <https://tailscale.com/download> —
everyone on the team must join the **same tailnet**.

**2. Tell LM Studio to accept connections from the network.** This is the step
everybody forgets. By default LM Studio listens on `127.0.0.1` only, so a
Tailscale address will not reach it no matter what the teammate configures.

> LM Studio → **Developer** tab → **Start Server** → turn on
> **Serve on Local Network**

**3. Get your Tailscale address** and send it to the team:

```bash
tailscale ip -4
```

That prints something like `100.83.14.22`. The MagicDNS name shown by
`tailscale status` (e.g. `gpu-box.tailnet-name.ts.net`) works just as well and
is easier to remember.

**4. Allow the port through the firewall — scoped to Tailscale only.** In an
**Administrator** PowerShell:

```powershell
New-NetFirewallRule -DisplayName "LM Studio (Tailscale only)" -Direction Inbound -Protocol TCP -LocalPort 1234 -RemoteAddress 100.64.0.0/10 -Action Allow
```

`100.64.0.0/10` is the Tailscale address range. Scoping the rule this way
matters: step 2 binds LM Studio to *all* interfaces, so without it your model
server is reachable by anyone on whatever Wi-Fi you join, with **no
authentication of any kind**. The rule keeps it reachable only from your
tailnet.

**5. Keep LM Studio running with a model loaded** while teammates work. There
is no queueing across users — the model serves one request at a time, so
concurrent reviews will simply wait for each other.

---

#### Side 2 — on each teammate's machine

**1. Install Tailscale and sign in to the same tailnet.** Verify:

```bash
tailscale status
```

The GPU machine should appear in that list.

**2. Check you can reach LM Studio** before touching any config:

```bash
curl http://100.83.14.22:1234/v1/models
```

A JSON list of models means everything above worked. If this hangs or is
refused, stop here — it is a Side 1 problem (step 2 or step 4), not a problem
with this project.

**3. Either type the address in the console, or set it in `.env`.**

*The quick way — no files, no restart.* Start the API, open
<http://localhost:8000>, and use the **Endpoint address** box under the
provider dropdown:

```
100.83.14.22
```

Press **Test** beside it. You should see `✓ reachable`, and the **Model**
dropdown fills with the models that machine actually serves — pick one, or
leave *(provider default)*. That address is used for the reviews you start,
and is shown next to the provider while one runs.

*The permanent way — two lines in `.env`,* so you never retype it:

```ini
REVIEW_API_DEFAULT_PROVIDER=lmstudio_tailscale
TAILSCALE_LMSTUDIO_HOST=100.83.14.22
```

**4. Start the API** as usual (`scripts/run-api.ps1`, `scripts/run-api.sh`, or
`docker compose up`) and confirm from the API's own point of view:

```
http://localhost:8000/api/v1/health?probe=1&provider=lmstudio_tailscale
```

`reachable: true` with a model list means you are done. Run a review.

---

#### If it does not work

| Symptom | Cause |
|---|---|
| `curl` to the address hangs or is refused | **Serve on Local Network** is off (Side 1 step 2), or the firewall rule is missing (step 4). Almost always this |
| `tailscale status` does not list the GPU machine | Different tailnets, or that machine is offline. Both must be signed into the same account/tailnet |
| Probe says `tailscale-host-not-set` | `TAILSCALE_LMSTUDIO_HOST` is empty in `.env`. The API is telling you the variable never got set |
| A provider or UI change does not appear | Running natively: the API reads its config **at startup** — restart it. Running in Docker: see the box below |
| Typed address rejected: *"outside the networks…"* | Only private and Tailscale (`100.64.0.0/10`) addresses are accepted, to stop the server being aimed anywhere. Use a configured cloud provider for a hosted model |
| Works by IP, not by MagicDNS name | MagicDNS is off in the tailnet admin console. Use the `100.x.y.z` address |
| Reviews are very slow when several people run at once | Expected. One model, one request at a time — they queue |
| Works natively, fails in Docker | The container routes through the host's tailnet, which normally just works. Check the host itself can still `curl` the address |

> **Security note.** Tailscale gives you an encrypted, private link and the
> firewall rule keeps the port off untrusted Wi-Fi — but LM Studio itself has
> **no authentication**. Anyone in your tailnet can use the model. That is
> fine for a project team; treat tailnet membership as the access control, and
> do not add the endpoint to a tailnet you share with people outside the
> project.

### 4d. Something else

`ollama`, `vllm`, `openai`, `groq` and `azure_openai` are configured out of the
box in [`backend/config/api.example.yaml`](../Code-Review-MCP/backend/config/api.example.yaml).
Set `REVIEW_API_DEFAULT_PROVIDER=<name>` and the matching `*_API_KEY` in
`.env`. Anything OpenAI-compatible works; add it as a new entry under
`providers:` with its `base_url` and `api_key_env`.

### Always verify before a long run

```
http://localhost:8000/api/v1/health?probe=1
```

`reachable: true` plus a model list means the endpoint answers *and* your key
works. This takes a second and saves the fifteen minutes you would otherwise
spend watching a review fail.

---

## 5. Run your first review

Open <http://localhost:8000> and work left to right:

1. **Connect** — the header pill turns green and the provider list fills in.
2. **Check repo access** — one second, no clone; confirms the repository is
   reachable.
3. **Build prompt only** — returns the exact prompt the server assembled
   (persona + review type + focus + safety rules) **without spending a
   token**. The cheapest way to see how the system works.
4. **Start review** — with `quick` mode first; it is one LLM call.
5. **Auto-poll** — the progress bar, ETA and per-batch counter update live.
6. **Report** — the summary, or **Markdown** for the readable version.

The same thing over HTTP:

```bash
curl -X POST http://localhost:8000/api/v1/reviews \
  -H "Content-Type: application/json" \
  -d '{"review_id":"MY-FIRST-01",
       "repo_url":"https://github.com/hadsahar/Secured-e-voting-system",
       "mode":"quick","persona":"security"}'

curl http://localhost:8000/api/v1/reviews/MY-FIRST-01
curl http://localhost:8000/api/v1/reviews/MY-FIRST-01/report
```

Results appear in **`reports/`** as `MY-FIRST-01.json` and `MY-FIRST-01.md`.

**Pick the depth deliberately** — wall time is essentially the model's, not
the server's:

| Mode | LLM calls | What it covers | 123-file repo |
|---|---|---|---|
| `quick` | 1 | every file via a regex surface scan, plus one AI overview | ~30–60s |
| `brief` | ~10 | every file, head of each, in batches | ~5–15 min |
| `deep` | one per file | full detail per file | hours — always scope it |

Scope deep mode with `files`: `"files": ["settings.py", "*/views.py"]`
(full path or bare filename, globs allowed). The intended workflow is quick or
brief over everything, then deep on what those flagged.

### Reference repository

`https://github.com/hadsahar/Secured-e-voting-system` — 123 reviewable files,
with known real findings you can check the AI against: a hardcoded
`SECRET_KEY` and `DEBUG=True` in `settings.py`, an API key in
`utils/debug.py`, RSA with primes 3 and 11 in `SMPC/rsa.cpp`.

---

## 6. Sharing one API — and its reports — with the team

**Reports live on the machine that runs the API.** They are files in
`reports/`, not rows in a shared database. So a teammate running their own
API sees their own reports folder, which starts empty. To see *yours*, they
point their console at *your* API.

That means one machine — the one with the GPU — runs the API, and everyone
else connects to it. Two sides to set up.

### 6a. Host side — expose the API on your tailnet

There are two ways in, and it is worth setting up both: the HTTPS one is
nicer to hand out, the IP one keeps working if Tailscale Serve is ever off.

**The recommended way: Tailscale Serve.** One command, and teammates get a
real HTTPS URL with a valid certificate and no IP to remember. Traffic
arrives through `tailscaled`, so no firewall rule is involved at all:

```bash
tailscale serve --bg --https=8443 http://127.0.0.1:8000
```

```
https://<your-machine>.<your-tailnet>.ts.net:8443/
```

`tailscale serve status` lists it, and it survives a reboot. Port 8443
rather than 443 leaves 443 free — on this project's own host it is already
serving LM Studio. Turn it off with
`tailscale serve --https=8443 off`.

**The direct way: publish on the tailnet IP.** Find your address:

```bash
tailscale ip -4
```

Put it in `.env` together with an API key. **Which variable depends on how
you run the API** — setting the wrong one looks like it worked and changes
nothing:

| How you run it | Set this | Leave this alone |
|---|---|---|
| `docker compose up` | `REVIEW_API_BIND` — the address Docker **publishes** on | `REVIEW_API_HOST`: the image already sets it to `0.0.0.0`, which binds inside the container's own namespace and is not the same thing. Compose does not pass yours through. |
| `python -m review_api.app` | `REVIEW_API_HOST` — the address Flask **binds** | `REVIEW_API_BIND`: only compose reads it |

```ini
# docker compose
REVIEW_API_BIND=100.93.244.39
```

> **`LMSTUDIO_BASE_URL` must stay commented out**, or Docker breaks in a way
> that looks unrelated. Compose passes it into the container, where
> `localhost` means the container itself, not your PC — the API comes up
> fine and every review fails with the model "unreachable". Left unset, each
> context picks the right answer on its own: `host.docker.internal:1234` in
> the container, `localhost:1234` for a direct Python launch.

**On the API key.** There is no user authentication here, so an exposed API
with no key means anyone who can reach it can run reviews on your GPU, read
every report and delete them. On a private tailnet of people you already
trust that can be an acceptable trade — this project's own host runs it that
way deliberately, for zero-friction access. Decide it, do not drift into it.
To require one, set `REVIEW_API_KEY` and give teammates the value for the
console's **X-API-Key** box.

A port change needs the container **recreated**, not restarted:

```bash
docker compose up -d
```

Then confirm the mapping really moved — this is the one thing worth checking,
because the console just says "cannot reach API" either way:

```bash
docker ps --filter name=review-api --format "{{.Ports}}"
```

You should see **two** mappings:

```
100.93.244.39:8000->8000/tcp, 127.0.0.1:8000->8000/tcp
```

Sharing **adds** an address, it does not move one — `http://localhost:8000`
keeps working for you. Only `127.0.0.1:8000->8000/tcp` on its own means
`REVIEW_API_BIND` never reached compose; check it is in `.env` at the
repository root, beside `docker-compose.yml`.

> **Use your tailnet address, not `0.0.0.0`.** `100.x.y.z` is reachable only
> from your tailnet. `0.0.0.0` also exposes the API on every other network
> the machine ever joins — café wifi, a hotel, a conference hall.

Restart, and check the two lines it prints:

```
[review-api] auth: X-API-Key required
[review-api] bind: 100.93.244.39:8000 — REACHABLE FROM THE NETWORK
```

If you bind wide without a key, it prints a loud warning instead — take it
seriously. This API clones repositories, accepts GitHub tokens, runs work on
your GPU and **deletes reports**. There is no per-user identity: the shared
key is all that stands in front of it.

**Windows firewall.** Tailscale usually installs a rule that permits inbound
traffic on its own interface, in which case the port is already reachable and
there is nothing to do — test it from a teammate's machine first. If it is
blocked, add a rule scoped to the tailnet range rather than opening the port
to every network. Run this yourself in an **elevated** PowerShell; it changes
a system security setting, so it is deliberately not scripted anywhere in
this repository:

```powershell
New-NetFirewallRule -DisplayName "AI Review API (Tailscale)" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 8000 -RemoteAddress 100.64.0.0/10
```

`-RemoteAddress 100.64.0.0/10` is the CGNAT block Tailscale allocates from,
so the rule admits tailnet peers and nothing else. To remove it later:

```powershell
Remove-NetFirewallRule -DisplayName "AI Review API (Tailscale)"
```

If your tailnet uses ACLs, allow the port there too — by default Tailscale
permits all traffic between your own devices.

Ports this machine ends up sharing with the team:

| Port | Service | Who needs it |
|---|---|---|
| 8443 (Serve) | Review API, over HTTPS | Everyone — the address to hand out to code and to the gateway |
| 8000 | The same API, direct on the tailnet IP | Fallback, and anything that dislikes a non-443 HTTPS port |
| 8090 | The test console, direct on the tailnet IP | Teammates who want the console for **your** API. With Tailscale Serve: `tailscale serve --bg --https=10000 http://127.0.0.1:8090` |
| 1234 | LM Studio | Only teammates running their **own** API against your model (§4c) |

Sharing 8000 is usually the better of the two: teammates get the reports as
well as the model, and only one machine needs a configured LLM.

### 6b. Teammate side — connect to it

Nothing to install. The host's addresses serve **the API only**
(`:8443` over Tailscale Serve, `:8000` on the tailnet IP) — they answer with
JSON, not a page. For the console, either:

- open the **host's console**, if they run it with compose and share it:
  `http://100.93.244.39:8090/` — it already points at their API; or
- open **`frontend/index.html` from your own checkout**, straight from disk,
  and point it at their API as below.

Then in the header:

1. **API base** — type the host's address. A bare `100.93.244.39` is enough;
   the console fills in `http://` and `:8000`. It remembers the address, so
   this is a one-time step.
2. **X-API-Key** — the value the host set in `REVIEW_API_KEY`. Not remembered
   between reloads; it is a credential.
3. Press **Connect**.

The header then reads `remote API 100.93.244.39:8000 · lmstudio reachable`,
and **§4 Saved reports** lists the host's reports with the machine named
above them. View, Markdown and Delete all act on the host's files — deletion
included, which is the other reason the key matters.

From code, the same thing:

```bash
REVIEW_API_URL=http://100.93.244.39:8000 REVIEW_API_KEY=... python backend/examples/review_client.py https://github.com/owner/repo --wait
```

---

## 7. Private repositories

Put a GitHub personal access token in `.env`:

```ini
GITHUB_TOKEN=github_pat_...
```

Read-only scope, with an expiry date. For an organisation repository with SSO,
authorise the token for that organisation.

The token reaches git through `GIT_ASKPASS`, so it appears in **no command
line, no clone URL, no log and no report** — reports record only
`authenticated: true`. Check access before committing to a review:

```
http://localhost:8000/api/v1/repo-access?repo_url=https://github.com/org/private-repo
```

If it fails, the response explains why — including GitHub's misleading "not
found" for an anonymous request to a private repository.

> **Never paste a token into a chat with the model.** The MCP server takes it
> from the environment only and no tool accepts one as an argument, precisely
> so it cannot land in a transcript or a host log. The HTTP API also accepts a
> per-request `access_token` in the body, which is fine because that never
> reaches a model.

---

## 8. Testing — including with no LLM at all

```powershell
powershell -ExecutionPolicy Bypass -File scripts\test.ps1   # Windows
bash scripts/test.sh                                        # Linux / macOS
```

**128 tests, no LLM, no network, no GPU.** They cover the pipeline, both
front ends, config loading, token handling, and — the interesting ones —
crash a review mid-run and prove the checkpoint-and-resume cycle.

### The fake LLM

To exercise the *whole* pipeline without a model, run the mock server. It is
OpenAI-compatible and always returns valid review JSON, so clone, chunking,
progress, checkpoints, resume and report writing all run for real; only the
findings are fake.

```powershell
# terminal 1
powershell -ExecutionPolicy Bypass -File scripts\run-mock-llm.ps1
# terminal 2
$env:LMSTUDIO_BASE_URL = "http://localhost:8765/v1"
powershell -ExecutionPolicy Bypass -File scripts\run-api.ps1
```

```bash
# terminal 1
bash scripts/run-mock-llm.sh
# terminal 2
LMSTUDIO_BASE_URL=http://localhost:8765/v1 bash scripts/run-api.sh
```

Do this first on a new machine. It separates "my setup is wrong" from "the
model is slow or unhappy".

### Seeing disaster recovery work

1. Start a `brief` review and let a batch or two finish.
2. Kill the API — Ctrl+C, or `docker compose kill api`.
3. Start it again. `GET /api/v1/reports` now lists the review under
   `resumable_checkpoints`.
4. Press **Restart** in the console, or POST the same `review_id` again.

Completed batches are reused: `restored_units` is greater than zero and the
finished report carries `resumed: true`. Only the unit in flight was lost.

---

## 9. Optional: the MCP server in LM Studio or Claude Code

Only needed if you want the chat-driven workflow, where the model calls the
review tools itself.

```powershell
docker build -t code-review-mcp ./mcp-server
```

Register it in your host's `mcp.json` — copy from
[`.mcp.json`](../Code-Review-MCP/.mcp.json) and **fix the paths to your checkout**:

```json
{
  "mcpServers": {
    "code-review-docker": {
      "command": "docker",
      "args": ["run", "--rm", "-i",
               "--add-host=host.docker.internal:host-gateway",
               "-v", "/your/path/to/Final-Project/reports:/reports",
               "-e", "GITHUB_TOKEN",
               "code-review-mcp:latest"],
      "timeout": 300000
    }
  }
}
```

[Document 16](16-gemma-user-guide.md) is the full walkthrough, and
[`mcp-server/PROMPTS.md`](../Code-Review-MCP/mcp-server/PROMPTS.md) is a paste-ready prompt
library.

> After changing MCP code: rebuild the image **and toggle the server off and
> on in LM Studio.** Otherwise it keeps running the old container. This is the
> single most common "my change did nothing" cause.

---

## 10. What actually differs between machines

Everything machine-specific is in **`.env`**. The config file, the code and
the containers are identical for everyone, because
`backend/config/api.yaml` interpolates `${VAR}` from the environment:

```yaml
base_url: ${LMSTUDIO_BASE_URL:-http://localhost:1234/v1}
```

Nothing set → the default applies. One environment variable → the endpoint
moves. That is how the same file serves a laptop, a container and CI.

Relative paths in that file resolve **against the config file**, not your
shell's working directory, so `output_dir: ../../reports` is the repository's
`reports/` folder whether you launch from the root or from `backend/`.

A `${VAR}` with **no** fallback that is not set is a deliberate startup error
naming the variable — better than a mangled URL discovered mid-review.

---

## 11. Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `pip` cannot find `code-review-mcp` | You installed `backend` alone. It is not on PyPI — install both local paths in one command (§3). |
| `ModuleNotFoundError: code_review_mcp` | Same cause, or the wrong Python. Use `.venv`. |
| `http://localhost:8000` shows JSON, not the console | Expected: the API serves no UI. The console is its own service on **http://localhost:8090** (compose), or open `frontend/index.html` from disk. |
| Console says `cannot reach API` | The console container cannot reach its upstream. Open `http://localhost:8090/console.json` to see which one it uses (`REVIEW_API_URL`). |
| `health?probe=1` → `reachable: false` | **On-prem:** the LLM server is not running, or you used `localhost` from inside a container (use `host.docker.internal`). **Cloud:** the key env var is not set — `/api/v1/providers` shows `api_key_configured` per provider. |
| `409 busy` | One review runs at a time by design. Wait, or cancel the active one. |
| `404` for a review you started | The API restarted and lost its in-memory registry. The 404 body lists resumable checkpoints; POST the same `review_id` to resume. |
| Review fails on some files with JSON errors | Reasoning models spend hidden tokens from the same budget. Raise `REVIEW_MAX_TOKENS`, lower `REVIEW_MAX_CHUNK_CHARS`, or use a different model. Failures are per-file; the run continues. |
| `bad interpreter: ...^M` on Linux | CRLF line endings. `.gitattributes` prevents this — re-clone rather than copying the folder from Windows. |
| Port 8000 already in use | `scripts/run-api.sh --port 8010`, or `REVIEW_API_PORT=8010 docker compose up`. |
| Docker: cannot reach LM Studio on the host | Ensure `extra_hosts: host.docker.internal:host-gateway` (compose sets it) and that LM Studio listens on all interfaces, not just loopback. |

### Docker: "I changed something and nothing happened"

Each image **contains a copy** of its files, so a plain `docker compose up`
keeps serving whatever was baked in at build time — silently, with no error.
Compose mounts the two things you are most likely to edit over those copies:

| Changed | Command |
|---|---|
| `.env`, the API config file, or `frontend/index.html` | `docker compose up -d` — these are mounted, so a restart is enough |
| `frontend/nginx/` or `frontend/Dockerfile` | `docker compose up -d --build console` |
| Python source in `backend/` or `mcp-server/` | `docker compose up -d --build api` |

When in doubt, `--build` is always safe. To see what the container is actually
serving rather than what you think it is:

```bash
docker compose exec api sh -c "grep -c lmstudio_tailscale /app/config/api.yaml"
curl -s localhost:8000/api/v1/config
```

Still stuck? `GET /api/v1/config` prints the entire effective configuration
with secrets removed — config file path, reports folder, frontend folder,
every provider and whether its key is set. It answers most "why is it doing
that" questions in one look.
