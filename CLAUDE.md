# CLAUDE.md: notes for Claude Code in this repository

AI Scan Repos: AI code review of GitHub repositories. Deterministic scanners
(Semgrep, Gitleaks, Trivy) plus an LLM (local LM Studio by default, or a
cloud API chosen by a site admin), run as Kubernetes Jobs on K3s. Team course
final project (Elad Eytan, Sahar, Aviel), deadline 2026-10-14.

Read first: `HANDOFF.md` (current state and next steps), then
`docs/WORK-TRACKER.md` (what is done, what the lecturer asked for). Where
they disagree with older docs, the tracker wins.

## Layout

```
Code-Review-MCP/
  mcp-server/      the engine (clone, scan hook, chunk, LLM review, checkpoint, report) + MCP front end
  backend/         review-api (Flask): creates reviews, runs them in-process or as K8s Jobs
  scanner/         Semgrep/Gitleaks/Trivy wrapper used by the engine
  scanner-worker/  the image each review Job runs (contract: docs/job-container-contract.md)
  gateway/         FastAPI: GitHub OAuth, sessions/history/settings in Postgres, proxies to review-api
  frontend-react/  the website (React 19, Vite, TypeScript), served by nginx
  frontend/        old test console (separate service, not deployed on K3s)
  k8s/             every manifest, one namespace ai-scan-repos, `kubectl apply -k`
docs/              design docs, contract, tracker, review-api reference (17)
```

The browser talks only to the gateway; the gateway talks to review-api;
review-api creates one Job per review attempt. Reports, checkpoints, scan
results and pause records live on the shared `/reports` volume.

## Commands (Linux/Git Bash; one virtualenv with every package installed editable)

```bash
cd Code-Review-MCP/mcp-server     && python -m pytest -q
cd Code-Review-MCP/backend        && python -m pytest -q
cd Code-Review-MCP/gateway        && python -m pytest -q
cd Code-Review-MCP/scanner-worker && python -m pytest -q
cd Code-Review-MCP/frontend-react && npm ci && npm run build && npm run lint
```

`backend/tests/test_review_api.py::test_failed_review_is_restartable_and_resumes`
is flaky (about 1 run in 12, a timing race older than the pause work).

Images and deploy: `Code-Review-MCP/k8s/DEPLOY-K3S.md` (build on the K3s
server, `docker save | sudo k3s ctr images import -`, set `newTag` in
`k8s/kustomization.yaml`). scanner-worker uses a fixed tag (now `0.5.0`)
named in `k8s/review-api/deployment.yaml` `SCANNER_WORKER_IMAGE`; bump it
whenever the worker, engine or scanner changes.

## Conventions

- Conventional Commits, one concern per commit, body says what and why.
- Comment density matches the file: every line of every k8s manifest has a
  comment; Python modules open with a docstring explaining the why.
- Changing the Job's inputs or outputs means a new contract version in
  `docs/job-container-contract.md` and a new scanner-worker tag.
- Update README / docs in the same change that makes them outdated.
- No CI exists: run the suites above before pushing.

## Rules from the team (keep)

- Never ask anyone to paste secrets (tokens, OAuth secrets, kubeconfigs) into
  chat, and never put them in files meant for git. Secrets are created from
  the CLI (`kubectl create secret ... --from-literal=...$(openssl rand -hex 32)`).
- Elad commits and pushes himself unless he asks Claude to; give him a
  commit message otherwise. No merge to main and no PR unless asked.
- For every command, say which machine and shell it runs on (Windows
  PowerShell, Git Bash, or the K3s server over SSH), what it does and what
  output means it worked.
- Elad is learning DevOps: explain K8s/Helm/CI rather than hand over finished
  YAML, unless he asks for the files.
- Prefer reversible actions. Don't touch the running cluster unless asked.
- review-api on Elad's laptop is shared over the tailnet with no API key on
  purpose. Do not re-add one unless asked.
- Messages to teammates are in Hebrew.
- Exit codes reach review-api through `/reports/<id>.error.json`, not pod
  status (listing pods would expose Job env). Team decision, 2026-09-26.

## Traps

- The gateway DB URL must start `postgresql+psycopg://`.
- A resume only reuses the checkpoint when `repo_url` is the exact same
  string (no `.git`) and the commit matches; resume pins the original commit.
- review-api keeps live review state in memory; paused/finished state is
  rebuilt from the reports volume after a restart.
- The 15-minute stale-progress rule kills a Job whose `progress.json` stops
  updating; `spec.suspend` therefore cannot be used for pausing.
- Site admins come from the ConfigMap `gateway-admins` (re-read live);
  `GATEWAY_SETTINGS_KEY` encrypts the saved AI API key, and changing it makes
  the saved key unreadable.
