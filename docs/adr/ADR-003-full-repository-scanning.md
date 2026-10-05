# ADR-003 — Full repository scans, not diff-only scans

**Status:** Accepted and implemented
**Date decided:** Baseline architecture
**Grounding:** FR-007 ("deterministic analysis across the entire
repository"), `docs/03-high-level-design/15-risks-tradeoffs.md` §"Full
Scans versus Incremental Scans"
**Related:** ADR-001, ADR-007

## Context

A pull-request review naturally suggests scanning only what changed —
it is faster, cheaper, and the diff is what the author is asking about.

But a diff-only scan can only report problems introduced by that diff. It
cannot answer "is this repository secure", and it will report a
repository as clean while a credential committed six months ago sits
untouched in a file nobody edited.

## Decision

Deterministic analysis covers the entire repository, at the reviewed
commit. A full scan is repeatable against that same commit, which is what
makes a coverage claim meaningful.

Diff-aware feedback is explicitly a future addition **alongside**
scheduled full scans, never a replacement for them. The risks document
states this directly: a future changed-files scan must not replace
scheduled full scans without an explicit decision. This ADR is that
constraint written down.

## Consequences

**Accepted:**
- Reviews take longer and cost more than a diff scan would.
- The same pre-existing finding reappears on every review until it is
  fixed. That is honest but repetitive, and it is the main reason a
  suppression workflow appears in the roadmap.
- Repository size and file-count limits are required, because "the whole
  repository" is unbounded input.

**Confirmed in practice (2026-08-28):** a full three-scanner run over a
Django repository produced 200 findings, of which 187 came from Trivy
scanning `requirements.txt` — dependency CVEs in files that a typical
pull request would never touch. A diff-only scan of that same repository
would have reported almost nothing. This is the concrete argument for the
decision, not a hypothetical one.

**Interaction with review modes:** `quick`, `brief` and `deep` differ in
how much *AI* attention each file receives, not in how much of the
repository the *scanners* cover. The deterministic tier is full-coverage
in every mode.

## Alternatives considered

| Option | Why rejected |
|---|---|
| Diff-only scanning | Cannot support a coverage claim; misses everything not touched by the diff |
| Full scan on a schedule, diff on each PR | Reasonable end state, but the roadmap requires it as an addition, not a substitution |
| Let the user choose the scope | Makes the report's meaning depend on an option most users will not understand |
