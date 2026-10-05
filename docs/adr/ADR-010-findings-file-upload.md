# ADR-010 — How a user-supplied findings file reaches the review

**Status:** Open — decision needed
**Raised:** 2026-09-06
**Related:** ADR-007, ADR-009, FR-007

## Context

Since review-api runs Semgrep, Gitleaks and Trivy on every review, the
dashboard's findings-file field is no longer needed for a normal scan. It
remains useful for the case ADR-009 describes: findings produced by a tool
we do not run.

But the field names a file **on the server**, and that makes it unusable
by the people it is meant for. A user of a deployed instance has no
filesystem access at all; even naming an exact path would only help local
development, since the path differs per deployment (`FinalProject/reports`
locally, `/reports` in a container, a PVC in Kubernetes).

The field's help text now says this plainly rather than giving an
instruction the reader cannot follow, but that is a stopgap.

Uploading is the only mechanism that works regardless of where the server
runs. The obstacle is not the upload handling — it is that **the file has
nowhere to land**:

- `review-api` exposes no upload endpoint (only reviews, restart, cancel).
- The gateway has no path to review-api's reports directory, and no
  knowledge that it exists. That is deliberate: the gateway has never
  touched another service's filesystem.

## Options

**A. A new upload endpoint in review-api.** The file lands where it is
read from, and the gateway keeps proxying rather than writing. Costs a
change in a component another track owns, and one more coordination point.

**B. The gateway writes into a shared reports directory.** No dependency
on another track. Costs the boundary: the gateway would need to know a
second service's storage layout, and in Kubernetes the two Pods would have
to share a PersistentVolumeClaim — coupling two components that are
otherwise joined only by HTTP.

## Whichever is chosen, the upload path must

- Sanitise the filename. Accepting `../../etc/x` is arbitrary file write.
- Cap the size, since the content is destined for a prompt budget.
- Validate that it parses before it is accepted, so a malformed file fails
  at upload rather than midway through a review.
- Treat the content as data, never as instruction — the reasoning in
  ADR-009 applies unchanged.

## Decision

Deferred to a joint decision, as it changes a boundary that has held for
the whole project. Not blocking: the automatic scan covers the normal
path, and the manual field still works for anyone with server access.
