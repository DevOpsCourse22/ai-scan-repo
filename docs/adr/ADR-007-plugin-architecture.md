# ADR-007 — Scanner plugin architecture

**Status:** Accepted and implemented
**Date decided:** 2026-08-27
**Implemented in:** `Code-Review-MCP/scanner/src/repo_scanner/`
**Related:** FR-007, FR-008, ADR-004

## Context

FR-007 names three scanners — Semgrep, Gitleaks, Trivy — and adds that
"the architecture shall support additional scanners through plugins."
FR-008 requires normalizing their findings into one internal format.

The three tools have nothing in common at the boundary. Semgrep emits
`{results: [...]}` with `ERROR/WARNING/INFO`; Gitleaks emits a bare JSON
array with no severity at all; Trivy nests `{Results: [{Vulnerabilities,
Misconfigurations, Secrets}]}` with CVSS severities. Writing three
bespoke integrations would mean three different answers to the same
questions — how a path is spelled, what "high" means, what gets redacted.

## Decision

One registry of adapters. Each adapter owns exactly the two things only
it knows: **how to invoke its tool** (`command()`) and **how to read its
output** (`parse()`), plus `coverage()` for what the tool actually looked
at. Everything shared lives in `findings.py`: the finding shape, the
severity vocabulary, path normalization, and the redaction rule.

Adding a scanner is one module plus one line in `ADAPTERS`. Nothing in
the runner, the CLI, or the review pipeline changes.

Parsing is deliberately separated from running, so an adapter can be
tested against a recorded sample of real tool output with the tool not
installed. That is what makes these testable in CI.

### Three rules the shared layer enforces

**1. Matched source text is never copied into a finding.** Every one of
these tools can return the text it matched — Semgrep's `extra.lines`,
Gitleaks' `Secret` and `Match`, Trivy's secret matches. That text is
exactly what is most likely to *be* a credential. A finding says which
rule fired and where. Reporting a leak by reprinting it would write the
secret into a findings file, an LLM prompt, and a stored report —
publishing it three more times as a side effect of reporting it.
Verified against real Gitleaks 8.30.1 output on planted credentials:
rule name, file, line and CWE present; the secret values absent.

**2. Paths are rewritten to repo-relative forward slashes.** Scanners
report paths relative to the directory they were pointed at, using the
host separator (`myclone\app\views.py` on Windows). The review matches
findings against its own manifest of repo-relative paths. A mismatch does
not raise — every finding lands in `unmatched`, the model receives
nothing, and the run still reports success. This is the most dangerous
failure mode in the component because it is completely silent.

**3. A missing scanner degrades the run; it never passes as a clean
one.** A tool that is absent, crashes, or times out is recorded by name
in `scanners_failed` and printed loudly to stderr. Letting one missing
binary abort everything would mean a machine without Trivy gets no
Semgrep results either. But "0 findings" and "the scanner never ran" are
different claims, and `docs/12-roadmap.md` makes telling them apart a
release gate.

### Severity is not uniform, on purpose

Semgrep maps `ERROR/WARNING/INFO` to `high/medium/low` and **never to
`critical`** — a pattern match is evidence that a pattern fired, not a
judgement about blast radius. Trivy's `CRITICAL` passes through
unchanged, because it derives from a CVSS score published with a known
CVE: an external, evidence-backed claim about a specific vulnerability.
Gitleaks has no severity of its own; every hit is `high`, since a
credential in a repository is never minor.

Escalation to `critical` otherwise belongs to the AI enrichment tier
(ADR-004), which can see whether a secret is live, scoped, or already
rotated.

## Consequences

**Accepted:**
- One more component to install and version.
- Scanner output shape is now our compatibility surface: a tool changing
  its JSON breaks one adapter, which is the point of the isolation.
- Gitleaks scans commit *history*, so it needs a clone with history — a
  shallow clone would silently reduce its coverage.

**Verified end to end (2026-08-28):** all three scanners ran over a
Django repository producing 200 findings (9 critical, 83 high, 89 medium,
19 low) with no scanner failures; a high-severity Semgrep finding was
carried through review-api into the report's `static_findings` with file,
line and source intact.

## Alternatives considered

| Option | Why rejected |
|---|---|
| One module per scanner, no shared layer | Three different answers to path, severity and redaction |
| Normalize inside review-api | The engine explicitly runs no scanner; it only ingests findings |
| Use each tool's native format and let the LLM sort it out | Puts raw secrets in the prompt and makes severity meaningless |
| Fail the whole run if any scanner is missing | One absent binary blocks every other scanner's results |
