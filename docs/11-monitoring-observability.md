# Monitoring and Observability

**Project:** AI Scan Repos  
**Document Version:** 1.0  
**Status:** Draft

---

## 1. Objectives

Observability must show whether users can submit reviews, authorization is correct, work progresses within expected time, scanners provide complete coverage, reports remain available and intact, cleanup succeeds, and external dependencies are healthy—without collecting source code or secrets.

## 2. Telemetry Standards

- OpenTelemetry for traces and vendor-neutral instrumentation.
- Prometheus-compatible metrics.
- Structured JSON logs with centralized redaction.
- Separate tamper-evident security audit events.
- Opaque request and review correlation IDs across asynchronous boundaries.

Repository URLs, filenames, user identities, tokens, code, secret values, prompts, and model responses are forbidden as metric labels. Controlled opaque tenant identifiers are used only when policy and cardinality permit.

## 3. Metrics

### 3.1 API and Frontend

- Request count, error ratio, and latency by normalized route/method/status.
- Active requests, worker saturation, and response-size bounds.
- Authentication outcome and authorization denial counts by safe reason class.
- Rate-limit and quota rejection counts.

### 3.2 Review Pipeline

- Queue depth and age of oldest queued review.
- Reviews admitted, running, completed, failed, cancelled, and timed out.
- Duration of authorization, queue, clone, each scanner, normalization, AI, upload, and cleanup phases.
- Job admission failure, reconciliation repair, orphan, and stale Job counts.
- Review concurrency and resource profile utilization.

### 3.3 Scanner and Report Quality

- Scanner invocation status/duration by scanner and version.
- Parser/truncation errors and incomplete coverage.
- Findings aggregated by bounded severity/category, never repository path.
- Summary/Deep generation and retrieval errors.
- Report upload latency, object-size bounds, and integrity failures.
- AI policy state, latency, validation failures, and bounded usage/cost measures.

### 3.4 Cleanup and Retention

- Worker cleanup success/failure.
- Time from terminal review to Pod/Job removal.
- Stale resource count and oldest age.
- Report retention deletion and error counts.
- Authorization-evidence expiration/invalidation outcomes.

## 4. Structured Logging

Allowed fields include timestamp, level, service, environment, event name, request ID, opaque review ID, phase, safe error code, duration, and outcome. Free-form error text is length-limited and redacted.

Do not log:

- Authorization headers, cookies, access tokens, clone grants, or environment dumps.
- Repository source, snippets, filenames where sensitive, or raw scanner output.
- Detected secret values.
- AI prompts or responses.
- SQL parameters containing sensitive data.
- Direct object-store keys in broadly accessible logs.

Scanner stdout/stderr is converted into bounded status events rather than forwarded wholesale.

## 5. Distributed Tracing

Trace spans cover API intake, identity validation, repository authorization, durable state transitions, Job creation, scan phases, report finalization, and retrieval. Request/response bodies are not captured.

Trace context is propagated to Jobs through safe metadata. Error and high-latency spans may receive higher sampling while respecting data policy. Provider spans record provider class and safe outcome, not endpoint credentials or payloads.

## 6. Security Audit Events

Audited actions include:

- Authentication success/failure and session termination.
- Repository permission decisions.
- Review creation, cancellation, timeout, and deletion.
- Summary and Deep Report access.
- Tenant, provider, AI, retention, and scanner-policy changes.
- Privileged elevation and break-glass access.
- Credential rotation and provider installation events.
- Cleanup and data-retention completion/failure.

Audit records are append-only or tamper-evident and retained separately from operational logs.

## 7. Dashboards

| Dashboard | Audience | Primary content |
|---|---|---|
| Platform health | SRE/Platform | API RED metrics, capacity, dependencies, deployment health |
| Review operations | DevSecOps | Queue, phase latency, outcomes, scanner coverage, cleanup |
| Security operations | Security | Authorization anomalies, privileged changes, runtime policy, integrity |
| Product usage | Product/Engineering | Aggregate submissions, completion, feature adoption without repository details |
| Provider health | Platform | Identity, repository, AI, storage, and Kubernetes API latency/failure |

## 8. Alerting

Page-worthy conditions include sustained API unavailability, oldest queue age exceeding the objective, widespread Job failures, database/report-store unavailability, report-integrity mismatch, cleanup failure with source persistence risk, cross-tenant access anomaly, and loss of audit ingestion.

Ticket-level alerts include gradual scanner error increase, elevated provider latency, capacity approaching quota, retention backlog, and expiring credentials/certificates.

Every alert has an owner, severity, dashboard, diagnostic steps, containment action, escalation path, and recovery criteria. Alerts are tested through game days and failure injection.

## 9. Initial SLO Framework

Numeric targets require load and business validation. Candidate service-level indicators are:

- Successful authorized API requests / valid authorized API requests.
- Reviews reaching a truthful terminal state / admitted reviews.
- Reviews beginning within queue-time objective / admitted reviews.
- Completed reports with all mandatory scanners / completed reports.
- Workspaces removed within cleanup objective / terminal Jobs.
- Authorized reports retrieved without integrity failure / retrieval attempts.

Provider denial, invalid input, user cancellation, and policy rejection are separated from platform unavailability.

## 10. Retention and Access

Operational telemetry uses the shortest useful retention. Security audit retention follows compliance policy. Access is least-privilege and audited. Telemetry exports are encrypted and region-aware. Observability platforms never become an unofficial source-code or report store.

## 11. Runbooks

Required runbooks cover API outage, queue backlog, repository-provider outage, Kubernetes Job failure, scanner regression, AI degradation, database saturation, object-store failure, report-integrity mismatch, suspected credential leak, cross-tenant authorization alert, cleanup failure, and audit pipeline loss.

## 12. Verification

Automated tests confirm metric availability, safe label cardinality, log redaction with canary secrets, trace propagation, alert routing, dashboard queries, audit append/integrity, and behavior when the telemetry backend is unavailable. Telemetry failure must not leak data or silently block cleanup.
