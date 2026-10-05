# ADR-004 — AI augments deterministic findings; it does not replace them

**Status:** Accepted, partially implemented
**Date decided:** Baseline architecture; enforced in code 2026-08-27/28
**Implemented in:** `mcp-server/src/code_review_mcp/findings.py` (intake),
`backend/src/review_api/prompts.py` (the enrichment duty in the prompt),
`Code-Review-MCP/scanner/` (the deterministic tier that feeds it)
**Related:** FR-007, FR-008, FR-009, FR-010, ADR-007

## Context

An LLM can read code and describe problems in it, which makes it
tempting to treat the model as the analysis engine. Two properties make
that unacceptable as the primary mechanism:

- **It is not reproducible.** The same repository can yield different
  findings on two runs. A security control that cannot be re-run to the
  same answer cannot be audited.
- **It cannot support a coverage claim.** "The model did not mention SQL
  injection" is not evidence that none exists.

Pinned scanners have the opposite profile: reproducible, versioned,
explainable, and weak exactly where the model is strong — they cannot
tell whether a match matters in context, and their output is
unreadable to a developer in a hurry.

## Decision

The deterministic tier is the source of truth about **what was found**.
The model's job is **enrichment** of that evidence: verify it, explain
why it matters here, prioritize it, and suggest a fix.

Mechanically: scanners write a findings file, the review references it by
name (`findings_file`), and the prompt carries an explicit enrichment
duty telling the model that findings are supplied with the code and its
role is not replacement. Findings are matched to the files actually under
review; anything that does not match is reported in `unmatched` rather
than dropped, so evidence is never lost silently.

Consequently **AI failure must never prevent a deterministic report from
completing**. Scanner results stand on their own.

## Consequences

**Accepted:**
- Two tiers to operate and two failure modes to distinguish.
- The deterministic tier has to run before the AI tier, which adds a
  step to the pipeline.
- Users must be able to tell scanner facts from AI inference. Findings
  carry a `source` field for exactly this; the report keeps them in
  `static_findings` separate from the model's prose.

**Verified 2026-08-28:** a Semgrep finding (`pickles-in-pandas`, high,
`voter/views.py:45`) travelled from the scanner through review-api into
the report's `static_findings` with file, line and source intact, and the
prompt contained the enrichment instruction.

**Not yet true:** the roadmap's Phase 5 gates — prompt-injection and
secret-reflection evaluation thresholds, provider/retention governance,
and a model evaluation dataset — are not met. What exists is the
architecture, not the assurance.

## Alternatives considered

| Option | Why rejected |
|---|---|
| AI as the primary analysis engine | Not reproducible; cannot support a coverage claim |
| Scanners only, no AI | Output is accurate but hard to act on; no prioritization or remediation advice |
| Run both independently and concatenate | Duplicate and contradictory findings, with no way to tell which to trust |
| Let the model re-derive severity from scratch | Discards CVSS evidence Trivy already provides (see ADR-007) |
