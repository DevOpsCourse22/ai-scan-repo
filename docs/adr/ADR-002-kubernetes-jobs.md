# ADR-002 — One isolated Kubernetes Job per repository review

**Status:** Accepted; not yet implemented
**Date decided:** Baseline architecture
**Grounding:** `docs/09-kubernetes-design.md` ("The central design
decision is one isolated Kubernetes Job per repository review"),
`docs/03-high-level-design/12-kubernetes-architecture.md`
**Related:** ADR-001, ADR-007

## Context

A review clones untrusted third-party source and runs several scanners
over it. That work is bursty, occasionally long, and occasionally
hostile: a repository can be enormous, malformed, or crafted to make a
scanner misbehave.

Running that inside a long-lived service process means one bad repository
can exhaust or destabilise the process that also serves every other
request.

## Decision

Each review runs as its own Kubernetes Job. The Job gets a restricted
security context, no Kubernetes token, no database access, no inbound
Service, and an active deadline. Its workspace is a size-limited
ephemeral volume (ADR-001).

Two consequences of the design are worth stating explicitly, because both
are already written into the Kubernetes design and both are easy to get
wrong:

- **Job success is not review success.** The report must exist, its
  digest must match, and cleanup must have finished. A green Job with no
  report is a failure.
- **Pod names are observations, not identities.** Kubernetes may replace
  an evicted or failed Pod, so records are keyed by a backend-generated
  run name and the Pod name is recorded as an observation.

A reconciler compares requested state in the database against actual Job
state and repairs missed events — missing Jobs, orphaned Jobs, replaced
Pods, expired Jobs, stuck deletion, and API restarts.

## Consequences

**Accepted:**
- Per-review startup cost (image pull, scheduling) versus a warm process.
- A reconciler is mandatory, not optional: without it, a missed watch
  event leaves a review stuck forever.
- Two very different deployment shapes in one system — the gateway is a
  long-running Deployment because it must always be listening for
  webhooks and logins; the scanner is a run-to-completion Job. This is
  deliberate and follows from their different work profiles.

**Current state, honestly:** none of this exists yet. Reviews run
in-process inside review-api, which serialises them one at a time and
returns 409 while busy. That is a reasonable local-development stand-in
and it makes the isolation gap explicit rather than hidden, but it is not
this ADR. Phase 2 in `docs/12-roadmap.md` is where this gets built.

The scanner tier (ADR-007) was written with this in mind: it takes a
directory path and writes a findings file, with no assumption about who
runs it or where. That is exactly the shape a Job needs.

## Alternatives considered

| Option | Why rejected |
|---|---|
| Run reviews in the API process | One hostile repository can destabilise the service for everyone |
| A long-lived worker pool | Shared state between reviews of different repositories; harder to bound and clean up |
| One Job for all queued reviews | Loses per-review isolation, deadline, and cleanup boundaries |
| Trust Job success as review success | A green Job with no report would be reported as a completed review |
