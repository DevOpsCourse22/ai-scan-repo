# ADR-008 — Policy-driven AI routing is deferred, and the seam is kept

**Status:** Deferred by decision — no routing logic will be built now
**Date decided:** Baseline architecture; reaffirmed 2026-08-28
**Grounding:** `docs/13-future-enhancements.md` §7 "AI Router",
`docs/12-roadmap.md` Phase 7 ("policy-based AI routing")
**Related:** ADR-004

## Context

Different reviews plausibly deserve different models: a small public
repository could go to a cheap hosted model, while anything private stays
on-premises. A router selecting among providers by policy — data
sensitivity, cost, latency, availability — is an obvious eventual want,
and appears in the future-enhancements document.

It is also the kind of component that is easy to build early and wrong,
because the policy it enforces depends on governance decisions the
project has not made: which providers are approved for which data
classes, and what the retention terms are.

## Decision

**Do not build routing logic now.** Keep the seam that makes it possible
later, and nothing more.

That seam already exists and is worth naming, because it is the whole
value of this ADR: review-api addresses every model through one
OpenAI-compatible provider abstraction. `backend/config/api.example.yaml`
already defines on-premises entries (LM Studio local, LM Studio over
Tailscale, Ollama, vLLM) and cloud entries (Google AI Studio, OpenAI,
Groq, Azure) behind a single `kind: openai_compatible` interface, with
`location: on_prem | cloud` recorded per provider.

Selecting a provider is therefore already a one-field change per request,
and `location` is already the field a future policy would key on. A
router becomes a policy layer over an existing choice rather than a
refactor.

## Consequences

**Accepted:**
- Provider choice is currently manual: a config default, overridable per
  request. Nobody is automatically routed anywhere.
- Adding a provider stays cheap; adding *policy* is deliberately still
  future work.

**The governance dependency, stated plainly:** sending repository source
to a cloud provider is a disclosure decision, not a performance tuning
choice. The `location` field exists so an operator can see whether code
leaves the premises. Until data governance approves provider, region,
retention and training behaviour — a Phase 5 exit criterion — cloud
providers remain an explicit, human choice, and routing to them
automatically would bypass exactly the decision that needs a human.

**Operational note that pushed this decision from theoretical to
practical (2026-08-28):** the on-premises model ran on a teammate's
laptop over Tailscale, and during real use it went offline mid-review
("Model stalled: no output for 300s"), then stopped responding entirely.
The fallback was manual: point the config at a local mock and re-run.
That is exactly the "clear degradation state when enrichment is
unavailable" the roadmap asks for — and today it is a human decision, not
an automatic failover. A router would have to make that call safely,
which is more design than this project needs before submission.

## Alternatives considered

| Option | Why rejected |
|---|---|
| Build the router now | Encodes governance decisions that have not been made |
| Automatic failover to a cloud provider on local failure | Silently turns an availability event into a data-disclosure event |
| Hardcode one provider and drop the abstraction | Loses the seam for no gain; the abstraction already exists and costs nothing |
