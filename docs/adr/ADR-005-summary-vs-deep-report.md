# ADR-005 — Two report levels: Summary and Deep

**Status:** Accepted and implemented
**Date decided:** 2026-08-15
**Implemented in:** review-api's `?view=` parameter, surfaced through the
gateway's `GET /reviews/{id}/report` and the dashboard's view selector
**Related:** FR-011, FR-012, ADR-006

## Context

BRD lines 102–106 define two report tiers, with Deep Reports available
only after ownership verification. The design question was where the line
between them falls, and what each tier is allowed to disclose.

## Decision

**Summary** returns aggregate information only: which files were
reviewed, how many findings each has, severity counts, and the overall
verdict. No finding text, no line numbers, no suggested fixes.

**Deep** returns the full detail: every finding with its severity, line,
description and suggested remediation.

Access follows ADR-006: Summary is available to anonymous callers
(`quick` and `brief` modes), Deep requires a session plus verified
permission on that specific repository.

A third rendering, `markdown`, is a *format* of the same content rather
than a third disclosure tier — it exists so a human can read a report
without parsing JSON.

## Consequences

**Accepted:**
- Anonymous users get something genuinely useful (a truthful shape of the
  problem) without being handed a map of exploitable detail in a
  repository they may not own.
- Two shapes to keep consistent. A field added to Deep must be
  deliberately considered for Summary, not inherited.

**A related decision, taken separately (2026-08-16):** whether `view=full`
should additionally require repository permission — i.e. whether the
*content depth* should be gated the same way the *report tier* is. Decided
**no, leave it open**, on the grounds that a public repository can be
analysed with any number of other tools, so gating our own full view buys
little. Recorded here as a decision rather than an oversight; revisit if
abuse of the full view ever becomes real.

**Known gap at the time of writing:** the roadmap's Phase 3 exit criteria
require that "anonymous Summary cannot expose Deep fields" be proven by
an authorization test matrix. That matrix does not exist yet. The
separation is implemented and manually verified, not test-enforced.

## Alternatives considered

| Option | Why rejected |
|---|---|
| One report level for everyone | Either discloses exploitable detail to anonymous callers, or makes the tool useless without login |
| Three or more tiers | No requirement asks for it; each tier doubles the consistency burden |
| Gate by repository visibility instead of report tier | A public repo can still contain a leaked credential worth not broadcasting |
