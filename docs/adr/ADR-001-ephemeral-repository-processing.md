# ADR-001 — Repository source is processed ephemerally

**Status:** Accepted; partially implemented
**Date decided:** Baseline architecture
**Grounding:** `docs/05-security-authorization-design.md` ("Keep source
ephemeral and out of control-plane services"; Restricted data class),
`docs/03-high-level-design/06-repository-analysis-flow.md`
**Related:** ADR-002, ADR-003

## Context

Reviewing a repository requires having its source on disk. Keeping that
source afterwards would make this system a copy of every repository it
has ever analysed — including private ones — and therefore a far more
attractive target than the review function alone justifies.

The security design classifies source, clone credentials, tokens,
detected secret values and AI context as **Restricted**: ephemeral,
secret-managed, never logged, narrow network and identity scope.

## Decision

Source code is fetched into a review-specific ephemeral volume, analysed,
and discarded. It is never stored in control-plane services, never
written to a database, and never logged. Clone credentials are
short-lived and scoped to the single repository being reviewed.

What persists is **derived, not source**: findings, reports, and the thin
metadata needed to map a trigger to a review.

## Consequences

**Accepted:**
- A re-review re-clones. No incremental or cached checkout.
- Nothing can be re-analysed after the fact from our own storage; if the
  upstream repository changes, the old state is gone.
- Cleanup is a correctness requirement, not housekeeping. The roadmap
  makes cleanup success a measured objective for exactly this reason.

**Where the current implementation falls short, honestly:**

The *source* side holds — repositories are cloned to a temporary
directory and removed. The *derived* side does not yet. Report files
accumulate indefinitely in the shared `reports/` directory: 46 files had
built up within two days of local testing, from three different
producers (the gateway, the demo console, and the MCP server directly).

Meanwhile the dashboard tells users we keep only metadata. That statement
is currently ahead of the implementation, which is the gap to close.

**Decided 2026-08-17, not yet built:** retention belongs to whoever owns
the files, which is review-api — it created them, knows their age, and is
the single component every producer goes through. A gateway-side TTL was
considered and rejected: it would only clean up reports the gateway knows
about, leaving anything created via the demo console or MCP untouched,
and a remote delete that fails between TTL expiry and the call leaks the
file silently. The plan is a sweeper in review-api plus a delete endpoint
so the UI can offer an explicit "delete now".

Immediate deletion after first fetch was also considered and rejected: the
report view selector fetches the same report up to three times
(summary/full/markdown), and download and page refresh fetch again.
Session-scoped deletion was rejected too — anonymous callers and webhook
triggers have no session, and most users never explicitly sign out.

## Alternatives considered

| Option | Why rejected |
|---|---|
| Cache clones for faster re-review | Makes the system a mirror of every private repo it touches |
| Persist source for audit | The audit need is served by findings, which are derived |
| Keep reports indefinitely | Contradicts what the product tells users, and grows without bound |
| Delete the report on first read | Breaks view switching, download and refresh |
