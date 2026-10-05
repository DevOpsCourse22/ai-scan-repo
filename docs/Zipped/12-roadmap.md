# Product and Architecture Roadmap

**Project:** AI Scan Repos  
**Document Version:** 1.0  
**Status:** Draft

---

## 1. Purpose

This roadmap sequences delivery from validated architecture to a production-ready repository analysis platform. Dates are intentionally excluded until team capacity, environment lead time, and security review availability are confirmed. Each phase has outcome-based exit criteria.

Security, source-code privacy, ownership verification, and truthful scanner coverage are release gates, not deferred enhancements.

## 2. Delivery Principles

- Deliver a narrow, end-to-end secure workflow before adding providers or advanced AI.
- Keep deterministic scanning functional without AI.
- Automate verification and cleanup from the first executable increment.
- Promote immutable artifacts through development, staging, and production.
- Record durable decisions in ADRs and update threat models with architecture changes.
- Base scaling and SLO decisions on measured representative repositories.

## 3. Phase 0 — Architecture Baseline

### Outcomes

- Executive Summary and Business Requirements approved.
- HLD, LLD, security design, threat model, API, database, Kubernetes, deployment, and observability documents reviewed.
- Core ADRs accepted for ephemeral processing, Jobs, full scans, AI augmentation, report levels, ownership verification, plugin architecture, and future AI routing.
- Initial data classification, risk register, and provider assumptions established.

### Exit criteria

- Requirements-to-design traceability reviewed.
- No unresolved contradiction in source retention or report authorization.
- Owners assigned for all critical risks and open decisions.

## 4. Phase 1 — Local Vertical Slice

### Scope

- FastAPI review endpoints and validated schemas.
- In-memory or development PostgreSQL review lifecycle.
- Local scanner worker invoking pinned Semgrep, Gitleaks, Trivy, and quality adapters.
- Common finding normalization and redaction.
- JSON/Markdown Summary and Deep report generation.
- Basic React submission and status workflow.
- AI disabled by default; optional local adapter behind policy.

### Exit criteria

- One repository can move from request to truthful terminal report.
- Secret values do not appear in logs or reports.
- Unit tests cover state, normalization, sanitization, and authorization behavior.

## 5. Phase 2 — Kubernetes MVP

### Scope

- One server-owned Kubernetes Job per review.
- Job reconciliation, timeout, cancellation, and layered cleanup.
- Restricted security context, RBAC, quotas, and default-deny networking.
- PostgreSQL persistence and encrypted report object storage.
- Helm deployment and Terraform development environment.
- Metrics, structured logs, traces, dashboards, and baseline alerts.

### Exit criteria

- Job and workspace isolation verified under success and forced failure.
- No scanner Kubernetes token or database access.
- Report integrity and idempotent review finalization tested.
- Development environment can be recreated from version-controlled infrastructure.

## 6. Phase 3 — Identity and Repository Trust

### Scope

- OIDC human authentication and federated workload identity.
- GitHub App integration and canonical repository model.
- Live provider permission verification for private and Deep workflows.
- Public Summary allowlist and non-disclosing denial behavior.
- Tenant roles, authorization evidence, audit events, and revocation handling.

### Exit criteria

- Complete cross-user and cross-tenant authorization test matrix passes.
- GitHub permission grant and revocation propagate within policy objective.
- Provider outage fails closed and produces a safe retryable state.
- Anonymous Summary cannot expose Deep fields.

## 7. Phase 4 — Production Hardening

### Scope

- External secrets and workload identities for all dependencies.
- Signed images, SBOM, provenance, and admission enforcement.
- Controlled hostname-aware scanner egress.
- Production quotas, dedicated scanner capacity, disruption policy, and autoscaling.
- Backup/restore, retention deletion, disaster recovery, and incident runbooks.
- Load, penetration, prompt-injection, cleanup, and chaos testing.
- Initial SLOs and on-call readiness.

### Exit criteria

- Independent security review has no unresolved critical/high issues.
- Restore and cleanup failure exercises meet approved objectives.
- Production alerts route to owned runbooks.
- Capacity test covers representative small, medium, large, and adversarial repositories.

## 8. Phase 5 — Controlled AI Enrichment

### Scope

- Versioned Context Builder and redaction policy.
- Approved local and/or hosted provider adapters.
- Structured response validation and deterministic-finding preservation.
- Model/prompt evaluation dataset, provenance, latency, and cost monitoring.
- Clear degradation states when enrichment is disabled or unavailable.

### Exit criteria

- Data governance approves provider, region, retention, and training behavior.
- Prompt-injection and secret-reflection evaluations meet acceptance thresholds.
- AI failure never prevents deterministic report completion.
- Users can distinguish scanner facts from AI inference.

## 9. Phase 6 — Enterprise Expansion

### Candidate scope

- GitLab, Azure DevOps, and Bitbucket adapters.
- Tenant-specific policy packs, retention, encryption, and capacity profiles.
- GitHub/GitLab status checks and carefully authorized annotations.
- SARIF export, suppression workflow, and historical trend analysis.
- Multi-cluster scanner capacity and regional placement where required.

Each provider must pass the same conformance and authorization suite before release.

## 10. Phase 7 — Advanced Assurance

Candidate capabilities include infrastructure-as-code analysis, SBOM and license compliance, artifact scanning, cross-scanner correlation, diff-aware feedback alongside periodic full scans, and policy-based AI routing.

Repository build or dynamic execution remains a separate high-risk program requiring sandbox architecture and a new threat model.

## 11. Cross-Cutting Workstreams

| Workstream | Continuous responsibility |
|---|---|
| Security | Threat model, dependency response, penetration testing, incident exercises |
| Architecture | ADRs, interface conformance, fitness functions, documentation consistency |
| Platform | Capacity, upgrades, cost, backup, recovery, and environment parity |
| Data governance | Classification, retention, provider terms, residency, deletion |
| Product quality | Scanner evaluation, false-positive workflow, usability, accessibility |
| Operations | SLOs, dashboards, alerts, runbooks, game days, post-incident learning |

## 12. Success Measures

- Median and tail review queue/processing time by repository size profile.
- Percentage of admitted reviews with complete mandatory scanner coverage.
- Cleanup success within the approved objective.
- Authorization and report-sanitization test pass rate.
- Finding triage time and accepted-remediation usefulness.
- False-positive disposition and recurrence trends.
- Production availability, recovery performance, and security incident rate.
- AI enrichment quality, validation failure, latency, and cost when enabled.

## 13. Roadmap Governance

Roadmap movement depends on exit criteria, not only feature completion. Material scope changes update the BRD, architecture, threat model, and test plan. Security or privacy debt that weakens a mandatory invariant blocks promotion unless an explicit time-bound risk exception is approved.
