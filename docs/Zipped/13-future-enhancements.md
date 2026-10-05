# Future Enhancements

**Project:** AI Scan Repos  
**Document Version:** 1.0  
**Status:** Candidate Backlog

---

## 1. Purpose

This document records potential capabilities beyond the initial production scope. Inclusion here is not an implementation commitment. Every enhancement must demonstrate user value, preserve the platform's security invariants, and pass architecture, privacy, threat-model, operational, and cost review.

## 2. Repository Provider Expansion

- GitLab application and merge-request integration.
- Azure DevOps organization/project adapter.
- Bitbucket Cloud and Data Center adapters.
- Enterprise/self-hosted provider endpoint support.
- Provider webhooks for membership, repository visibility, and installation revocation.

All adapters must implement canonical repository identity, normalized permissions, immutable revision resolution, short-lived clone authorization, safe endpoint policy, rate handling, and conformance tests.

## 3. Scanner and Analysis Expansion

- Infrastructure-as-Code scanning for Terraform, Kubernetes, Helm, and cloud policy.
- SBOM generation using standard formats.
- License inventory and compliance policy.
- Container image and build-artifact scanning.
- API and configuration security analysis.
- Language-specific quality and architecture rules.
- Malware and suspicious repository-content detection.
- Cross-scanner correlation and root-cause grouping.

New scanners run only from signed, administrator-approved images or built-in adapters. User-uploaded executable plugins are not accepted.

## 4. Faster Developer Feedback

- Diff-aware pull-request analysis for changed-code feedback.
- Cached immutable dependency metadata and approved scanner databases.
- Language-aware repository partitioning for large monorepos.
- Scanner parallelism tuned to resource profile.
- Incremental result comparison against the previous immutable revision.

Incremental analysis supplements rather than silently replaces scheduled or policy-required full scans.

## 5. Finding Lifecycle

- Finding fingerprinting across revisions.
- Status workflow: open, acknowledged, fixed, accepted risk, false positive.
- Time-bounded suppression with reason, approver, and expiry.
- Ownership assignment and service/team routing.
- Remediation verification on a new commit.
- SLA and exception-policy tracking.
- Historical trends and recurrence detection.

Every disposition is audited and does not alter the original scanner evidence.

## 6. Reporting and Integration

- SARIF export.
- Signed PDF and organization-branded reports.
- Webhooks and event subscriptions.
- GitHub/GitLab status checks.
- Jira, Slack, Teams, email, and SIEM integrations.
- Portfolio-level dashboards using aggregate non-sensitive data.
- Policy-controlled report sharing and expiration.

Pull-request annotations must respect repository authorization and avoid publishing Deep evidence to unauthorized viewers.

## 7. AI Router

A policy-driven AI Router could select among local and hosted models using:

- Tenant-approved provider list.
- Data classification and residency.
- Task type and required capability.
- Availability, latency, token budget, and cost.
- Model evaluation status and safety profile.

Fallback must never weaken a privacy or residency rule. Routing, prompt, provider, and model provenance must be recorded. Implementation requires acceptance of ADR-008 and a dedicated threat-model review.

## 8. Multi-Agent Reasoning

Specialized bounded agents may focus on security, code quality, architecture, dependencies, and remediation before a synthesis stage. Agents would receive immutable minimized inputs, no arbitrary tools, no repository-write permission, and schema-constrained outputs.

This capability has elevated prompt-injection, disclosure, latency, cost, and provenance risk and remains exploratory.

## 9. Remediation Assistance

- Contextual secure-code examples.
- Suggested patch generation in an isolated view.
- Verification steps and regression-test suggestions.
- Links to approved internal standards.
- Optional draft pull request after explicit human approval.

The platform must never push, merge, or execute a suggested change automatically. Generated patches are untrusted output and require authorization, review, and scanning.

## 10. Enterprise Governance

- Tenant-specific rule packs and policy-as-code.
- Data residency and regional processing controls.
- Customer-managed encryption keys.
- Fine-grained delegated administration.
- SCIM lifecycle and enterprise identity federation.
- Compliance evidence packages and immutable audit export.
- Organization-level quotas, budgets, and chargeback.
- Dedicated namespace, node pool, or cluster isolation tiers.

## 11. Platform Scale

- Multi-cluster scanner scheduling.
- Regional control planes and report stores.
- Queue partitioning by tenant and resource profile.
- Warm scanner infrastructure without shared writable state.
- Capacity prediction based on repository metadata that does not expose source.
- Disaster recovery across regions.

Scaling changes must preserve review idempotency, tenant isolation, immutable provenance, and cleanup guarantees.

## 12. Safe Dynamic Analysis

Potential future dynamic capabilities include controlled builds, tests, software composition resolution, and runtime analysis. These are explicitly excluded from the current architecture because they execute attacker-controlled code.

Any implementation requires a separate sandbox with disposable compute, no ambient credentials, tightly controlled dependency proxy, syscall and network restriction, output limits, and a new threat model. It must not reuse the static scanner trust assumptions.

## 13. Developer and Operator Experience

- Repository onboarding wizard and permission diagnostics.
- Review cost/duration estimate by resource profile.
- Accessible report navigation and remediation workflow.
- Scanner policy simulation before activation.
- Provider health and rate-limit dashboards.
- Self-service retention and deletion controls.
- Runbook automation for safe repetitive recovery actions.

## 14. Evaluation Criteria

Before prioritization, each enhancement is evaluated for:

1. User and business value.
2. Security and privacy impact.
3. Data classification and external transfer.
4. Tenant isolation and authorization behavior.
5. Operational ownership, SLO, and failure modes.
6. Cost and capacity profile.
7. Portability and provider coupling.
8. Testability and rollback.
9. Documentation, ADR, and threat-model changes.

Enhancements that violate ephemeral source handling, deterministic scanner authority, ownership-verified Deep access, or replaceable interfaces are rejected or redesigned.
