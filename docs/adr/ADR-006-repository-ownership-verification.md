# ADR-006 — Repository ownership verification for Deep Reports

**Status:** Accepted and implemented
**Date decided:** 2026-08-15
**Implemented in:** `Code-Review-MCP/gateway/src/review_gateway/github_oauth.py`
(`check_repo_permission`), enforced in `app.py` (`_require_deep_access`)
**Related:** FR-006, FR-011, ADR-005

## Context

A Deep Report exposes per-file detail: every finding, its line, and a
suggested fix. FR-006 requires validating repository permissions through
provider APIs, and BRD line 106 states Deep Reports are available only
after successful ownership verification.

The question was what "verified" has to mean in practice. Two weaker
readings were available and both were rejected:

- **"Signed in" is enough.** Any authenticated user could then request a
  Deep Report on any repository, including a private one belonging to
  someone else.
- **"The URL is under the user's own account."** Trivially bypassed by
  string manipulation, and wrong in both directions: it denies legitimate
  access to organisation repositories the user is a member of.

## Decision

For `mode=deep`, the gateway requires a live session **and** performs a
per-request permission check against GitHub for **that specific
repository**, using the signed-in user's own token:

```
GET /repos/{owner}/{repo}/collaborators/{username}/permission
```

Anything other than `admin`, `write`, or `read` is refused with 403.
`quick` and `brief` are deliberately left open to anonymous callers,
matching the Summary Report tier in ADR-005.

The check asks GitHub at request time rather than trusting anything
cached. GitHub is the authority on who may read a repository; a stored
answer would keep granting access after the real permission was revoked.

## Consequences

**Accepted:**
- Deep Reports cost one extra GitHub API call per request.
- A GitHub outage fails Deep Reports closed. This is the correct
  direction — the alternative is granting access we cannot verify.
- Organisation repositories work correctly, because GitHub's answer
  accounts for team membership.

**Related decision, made separately (2026-08-17):** the same reasoning
extends to *reading* a review, not only creating one. Reviews started
while signed in are readable only by that user. Reviews with no owner —
anonymous `quick`/`brief`, or webhook-triggered — remain readable by
`trigger_id`, because those flows have no identity to compare against and
the id is an unguessable UUID. Denial returns 404 rather than 403 so the
endpoint cannot be used to probe which ids exist.

**Not covered by this ADR:** this is a GitHub OAuth App checking a single
provider. Full OIDC, GitHub App installation identity, tenant roles,
audit events, and revocation propagation are Phase 3 work in
`docs/12-roadmap.md`. What exists today is a working spike of that phase's
authorization model, not its final architecture.

## Alternatives considered

| Option | Why rejected |
|---|---|
| Session presence only | Grants Deep access to repositories the user cannot read |
| Owner-string match on the URL | Bypassable, and wrongly denies org repos |
| Cache the permission result | Keeps granting access after revocation |
| Gate all modes behind login | Contradicts the anonymous Summary tier (ADR-005) |
