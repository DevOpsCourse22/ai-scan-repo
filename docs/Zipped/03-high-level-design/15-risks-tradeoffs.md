# 15. Risks and Trade-offs

## 15.1 Risk Management Approach

The architecture reduces risk but cannot eliminate it. Each material risk requires an accountable owner, treatment decision, verification evidence, and review date. Acceptance of residual risk must be explicit rather than implied by implementation.

## 15.2 Principal Risks

| ID | Risk | Likelihood | Impact | Architectural treatment | Residual concern |
|---|---|:---:|:---:|---|---|
| R-01 | Malicious repository exploits a scanner | Medium | Critical | Patched tools, isolated non-root Job, no code execution, resource/network limits, optional sandbox runtime | Scanner zero-days remain possible |
| R-02 | Source or secret reaches logs, reports, or AI | Medium | Critical | Data classification, redaction, context minimization, output validation, hosted-AI opt-in | Automated redaction may miss novel secrets |
| R-03 | Incorrect authorization exposes a Deep Report | Low | Critical | Provider verification, server-side resource checks, fail-closed behavior, denial tests, audit | Provider identity mapping errors require monitoring |
| R-04 | Large repositories exhaust capacity | High | High | Size/file limits, quotas, deadlines, queue fairness, autoscaling | Strict limits may reject legitimate monorepos |
| R-05 | Provider outage blocks reviews | Medium | Medium | Timeouts, circuit breakers, retryable states, deterministic AI degradation | Ownership checks must not fail open |
| R-06 | AI produces incorrect remediation | High | Medium | Advisory labeling, deterministic provenance, schemas, evaluation and feedback | Human judgment remains required |
| R-07 | Cleanup fails and source persists temporarily | Low | High | Explicit cleanup, ephemeral volume, Pod deletion, TTL controller, stale-resource alert | Node/storage failure can delay physical deletion |
| R-08 | Supply-chain compromise affects scanner image | Low | Critical | Pinning, SBOM, scanning, provenance, signing, admission policy | Trusted upstream compromise remains possible |
| R-09 | Multi-tenancy allows resource or data interference | Medium | High | Authorization, quotas, network policy, workload isolation, tenant-aware storage | Strong isolation may require separate namespaces/clusters |
| R-10 | Scanner false positives reduce trust | High | Medium | Provenance, deduplication, suppression governance, AI explanation, feedback metrics | Tuning varies by language and organization |

## 15.3 Architectural Trade-offs

### Full Scans versus Incremental Scans

Full-repository scanning maximizes coverage and produces a reproducible baseline, but increases duration and cost. The initial authoritative review remains a full scan. Future diff-aware scans may provide faster feedback but cannot silently replace periodic full coverage.

### Kubernetes Jobs versus a Persistent Worker Pool

One Job per review provides clear isolation, resource accounting, and cleanup semantics. It adds startup latency and Kubernetes object churn. A persistent pool would improve warm-start performance but expands cross-review isolation and cleanup risk; it is not selected initially.

### Modular Monolith versus Microservices

A modular backend reduces network complexity, deployment overhead, and distributed transaction risk while interfaces are still evolving. Internal boundaries allow later extraction. Premature microservices are avoided until scaling, security, ownership, or release independence justifies them.

### Hosted AI versus Local AI

Hosted models may provide higher capability and lower operational burden, but introduce data residency, retention, and external disclosure concerns. Local models provide stronger deployment control but require capacity, patching, evaluation, and model operations. The policy-driven provider abstraction supports both; AI remains optional.

### Public Summaries versus Privacy

Public summaries can make the platform accessible and useful for public repositories, but even aggregate results can reveal sensitive posture. Summary publication therefore uses an allowlist, public-repository verification, anti-enumeration controls, and tenant policy. Private repository summaries are never public.

### Immediate Deletion versus Auditability

Source and AI context are deleted immediately, while normalized findings, reports, and audit events may be retained for business and compliance value. The design separates these data classes so auditability does not justify retaining complete source.

## 15.4 Known Limitations

- Static scanners cannot identify every vulnerability or prove absence of defects.
- The initial platform does not safely execute builds or dynamic tests.
- NetworkPolicy alone cannot provide portable hostname-level egress control.
- Ownership verification depends on provider availability and correct application installation.
- Repository size and scan deadlines may exclude very large monorepos until partitioning is designed.
- AI explanations may be incomplete, outdated, or wrong and require human validation.
- Cross-cloud portability does not mean identical operational behavior or cost.

## 15.5 Decision and Review Process

Risks are reviewed at architecture milestones and after incidents, major provider additions, new execution capabilities, tenancy changes, or data-policy changes. Decisions that alter mandatory principles are recorded in ADRs and reflected in the threat model, security design, and tests.

Next: [Future Architecture](16-future-architecture.md).
