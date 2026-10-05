# 13. Observability

## 13.1 Objectives

Observability must allow operators to answer whether the platform is available, reviews are progressing, authorization is behaving correctly, scanners are providing expected coverage, cleanup is succeeding, and external dependencies are healthy. Telemetry must achieve this without collecting repository source or secrets.

## 13.2 Telemetry Model

The platform uses metrics, structured logs, distributed traces, security audit events, and synthetic checks. Every review and request receives an opaque correlation identifier propagated across the API, orchestrator, Job, provider calls, and report finalization.

Repository URLs, filenames, user tokens, code fragments, secret matches, prompts, and AI responses are prohibited as metric labels. Tenant and repository identity use controlled opaque values only where operational policy permits them.

## 13.3 Service-Level Indicators

| Area | Indicators |
|---|---|
| API | Request rate, error ratio, latency, saturation, authorization denials |
| Review queue | Queue depth, oldest queued age, admission rejections, per-tenant concurrency |
| Review execution | Duration by phase, terminal outcome, timeout, cancellation, incomplete coverage |
| Scanners | Invocation duration, tool exit status, parser errors, findings by bounded category/severity |
| AI enrichment | Policy outcome, availability, latency, validation failures, bounded token/cost measures |
| Reporting | Generation latency, upload failures, integrity mismatch, authorized download errors |
| Cleanup | Workspace cleanup result, stale Job count, TTL delay, leaked-resource detection |
| Dependencies | Identity, repository provider, Kubernetes API, database, object store, and queue health |

Numeric service-level objectives are set after representative performance testing. Availability metrics must distinguish platform failure from provider rejection, invalid user input, and policy denial.

## 13.4 Logging

Logs use structured fields such as timestamp, severity, service, environment, request ID, review ID, phase, event name, outcome, and sanitized error class. Free-form exception content passes through redaction and length limits.

Scanner stdout and stderr are not forwarded wholesale. Adapters convert relevant execution conditions into bounded operational events. Debug logging is disabled in production and cannot be used to bypass data-handling policy.

## 13.5 Distributed Tracing

Traces cover request intake, identity validation, repository authorization, durable state transitions, Job admission, scan phases, report upload, and report retrieval. Sensitive request/response bodies are excluded. Sampling is configurable, with error and high-latency traces retained at a higher rate under policy.

Asynchronous boundaries propagate trace context through safe metadata. The scanner Job creates child spans linked to the originating review without including clone credentials or source details.

## 13.6 Audit Events

Security audit records are distinct from operational logs. Audited actions include sign-in outcomes, repository authorization, review creation/cancellation, report access, policy and provider changes, administrative elevation, secret rotation events, and retention deletion.

Audit records identify actor, action, resource, decision, policy version, time, correlation ID, and outcome. They are tamper-evident and accessible only to authorized security and compliance roles.

## 13.7 Alerts and Runbooks

High-priority alerts include sustained API unavailability, excessive queue age, widespread Job failure, provider authorization anomalies, report-integrity failure, repeated cross-tenant denial, cleanup failure, stale workspaces, database exhaustion, and telemetry loss.

Every actionable alert has an owner, severity, diagnostic dashboard, safe investigation steps, containment action, escalation path, and closure criteria. Alerts are tested through controlled failure exercises and tuned to avoid paging on a single expected user or scanner error.

## 13.8 Dashboards

Dashboards serve separate audiences:

- Platform operations: service health, capacity, dependencies, and deployment state.
- Review operations: queue, phase duration, coverage, failures, and cleanup.
- Security operations: authorization anomalies, administrative activity, runtime policy, and secret-handling controls.
- Product and engineering: aggregated usage and completion trends without exposing repository content.

## 13.9 Retention and Access

Telemetry retention is purpose-limited and shorter for high-volume operational data. Access follows least privilege and is audited for security data. Exported telemetry remains encrypted and subject to tenant and regional policy. Observability systems are not an alternative report store.

Next: [Technology Stack](14-technology-stack.md).
