# 13. Observability

**Document Version:** 1.1

**Status note (updated 2026-09-24, see [Scope Decisions Log](../SCOPE-DECISIONS.md)):** The Job/Pod/run-name tracking described here is still target state: nothing in the cluster is being watched or scraped yet, and no metrics or dashboards exist. Reviews running today report progress through the control plane's own in-memory state, which does not survive a restart. This chapter is the least-built of the Kubernetes chapters - worth saying plainly rather than letting the detail imply otherwise.

## 13.1 Objectives

Monitoring must show whether the API is available, reviews are moving, token checks are working, Semgrep is completing, reports are valid, and cleanup succeeds. Metrics and logs must not contain source code, secrets, repository URLs, or model tokens.

## 13.2 Telemetry Model

The platform uses metrics, structured logs, traces, audit events, and test requests. Each request has a request ID. Each review has a review ID. Each scanner execution has a backend-generated run name. The Job and Pod carry the run name in the `ai-scan.io/run-name` label.

The run name, Job name, and Pod name may appear in restricted logs and traces. They must not be metric labels because every value is unique. Repository URLs, filenames, tokens, source code, secret matches, prompts, and model responses are not logged.

## 13.3 Service-Level Indicators

| Area | Indicators |
|---|---|
| API | Request rate, error ratio, latency, saturation, authorization denials |
| Review queue | Queue depth, oldest queued age, admission rejections, per-tenant concurrency |
| Review execution | Queue time, scheduling time, Pod running time, terminal outcome, timeout, cancellation |
| Semgrep | Image digest, rule-policy version, exit status, parser errors, incomplete coverage |
| Reporting | Generation latency, upload failures, integrity mismatch, authorized download errors |
| Cleanup | Workspace cleanup result, stale Job count, Job removal delay, leftover-resource detection |
| Dependencies | Identity, repository provider, Kubernetes API, database, object store, and queue health |

Targets are set after load testing. Metrics distinguish platform failure from an invalid token, invalid request, provider denial, or user cancellation.

For each run, PostgreSQL stores the run name, Job name, actual Pod name, start time, completion time, outcome, and `duration_ms`. Statistics use these stored values while dashboards show grouped results rather than individual run names.

## 13.4 Logging

Logs use structured fields such as timestamp, severity, service, environment, request ID, review ID, phase, event name, outcome, and sanitized error class. Free-form exception content passes through redaction and length limits.

Semgrep stdout and stderr are not stored in full. The scanner records a bounded status and safe error code. Debug logging is disabled in production.

## 13.5 Distributed Tracing

Traces cover request intake, token validation, repository authorization, database state changes, Job creation, Semgrep execution, report upload, digest verification, and report retrieval. Request and response bodies are excluded.

The control plane passes a safe trace reference and the run name in Job metadata. This links control-plane and scanner traces without a direct network connection between their namespaces.

## 13.6 Audit Events

Audit records are separate from operational logs. They record sign-in and model-token results, repository authorization, review creation and cancellation, report access, policy changes, administrator access, credential rotation, and retention deletion.

Audit records identify actor, action, resource, decision, policy version, time, correlation ID, and outcome. They are tamper-evident and accessible only to authorized security and compliance roles.

## 13.7 Alerts and Runbooks

High-priority alerts include API outage, an old review queue, failure to create or watch Jobs in `ai-scan-jobs`, widespread Semgrep failure, report digest mismatch, cleanup failure, database exhaustion, and loss of audit events.

Every alert has an owner, severity, dashboard, investigation steps, action, and recovery condition. A single rejected token or failed scan does not trigger a page.

## 13.8 Dashboards

Dashboards serve separate audiences:

- Platform operations: API health, database, object storage, Kubernetes API, and capacity.
- Review operations: queue time, Job creation, Pod duration, Semgrep outcome, report upload, and cleanup.
- Security operations: authorization anomalies, administrative activity, runtime policy, and secret-handling controls.
- Product and engineering: aggregated usage and completion trends without exposing repository content.

## 13.9 Retention and Access

Keep telemetry only as long as required for operations and audit policy. Limit and audit access. Monitoring systems are not report storage.

Next: [Technology Stack](14-technology-stack.md).
