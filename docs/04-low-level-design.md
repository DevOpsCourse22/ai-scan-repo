# Low-Level Design (LLD)

**Project:** AI Scan Repos  
**Document Version:** 1.0  
**Status:** Draft  
**Owner:** AI Scan Repos Engineering Team

---

## 1. Purpose

This document translates the [High-Level Design](03-high-level-design/README.md) into implementable component boundaries, internal contracts, state transitions, validation rules, and failure behavior. It defines how the initial platform should be built without fixing every class name or cloud-specific detail.

The LLD covers the backend control plane, repository-provider adapters, scanner worker, normalization pipeline, AI provider adapters, report generation, persistence, and Kubernetes Job reconciliation.

## 2. Design Constraints

- Repository source exists only in a review-specific ephemeral workspace.
- Core services never infer repository ownership from Git data or user input.
- Deep Report access requires server-side repository authorization.
- Deterministic scanners are authoritative; AI output is advisory.
- Users cannot control scanner images, commands, namespaces, volumes, or arbitrary environment variables.
- Components depend on provider interfaces rather than vendor SDKs in business logic.
- Long-running reviews are asynchronous and idempotent.
- Sensitive data never appears in application logs, metrics, or audit details.

## 3. Backend Package Design

```text
backend/app/
├── api/                 # HTTP routing and request/response mapping
├── services/            # Use cases and domain orchestration
├── providers/           # Repository and AI adapters
├── models/              # Persistence models
├── schemas/             # External and internal validated contracts
├── database/            # Session, repositories, and migrations
├── config.py            # Validated runtime configuration
└── main.py              # Application composition and middleware
```

HTTP routes perform authentication binding, schema validation, service invocation, and response mapping. They do not contain authorization policy, SQL, provider calls, or Kubernetes manifest construction.

### 3.1 Core Services

| Service | Responsibilities |
|---|---|
| `AuthorizationService` | Evaluate platform role, report level, repository permission, tenant policy, and resource ownership |
| `RepositoryService` | Canonicalize repository references, select provider, verify access, and resolve immutable revisions |
| `ReviewService` | Create idempotent reviews, own lifecycle transitions, expose status, and coordinate cancellation |
| `KubernetesJobService` | Render server-owned Job specifications, create/delete Jobs, and read bounded status |
| `ReviewReconciler` | Compare durable review state with Kubernetes state and repair uncertain outcomes |
| `ReportService` | Retrieve report metadata, authorize access, and create the correct Summary or Deep representation |
| `AuditService` | Append security and lifecycle events without sensitive payloads |

Each service receives interfaces for persistence and external systems through dependency injection. Unit tests replace those interfaces with deterministic fakes.

## 4. Domain Model

### 4.1 Review

The Review aggregate contains:

- Opaque UUID.
- Tenant and requester identifiers.
- Canonical repository provider and external repository ID.
- Requested ref and resolved commit SHA.
- Requested report level and policy version.
- Lifecycle status and optimistic-lock version.
- Scanner coverage and terminal error classification.
- Report object references and integrity digests.
- Creation, start, completion, cancellation, and expiration timestamps.

Valid lifecycle transitions are:

```text
requested -> authorizing -> queued -> running -> finalizing -> completed
     |             |           |         |            |
     +-------------+-----------+---------+------------+-> failed
                               +-----------------------> cancelled
                               +-----------------------> timed_out
```

Only `ReviewService` and `ReviewReconciler` may transition state. Updates use an expected version so concurrent workers cannot overwrite a newer transition.

### 4.2 Finding

The normalized finding schema contains:

```text
finding_id, review_id, fingerprint, scanner, scanner_version,
rule_id, category, scanner_severity, confidence, title,
relative_path, start_line, end_line, redacted_evidence,
remediation, ai_priority, ai_explanation, provenance, schema_version
```

Paths must be relative, normalized, and contained within the repository root. Secret evidence is represented by a redacted value or evidence reference, never the detected secret.

## 5. API Request Processing

Every request passes through correlation-ID, trusted-proxy, authentication, rate-limit, body-size, and exception-mapping middleware. Request schemas reject unknown execution controls and embedded repository credentials.

Review creation performs these operations:

1. Validate request and idempotency key.
2. Map the authenticated or anonymous subject.
3. Canonicalize the provider and repository identity.
4. Apply Summary or Deep authorization policy.
5. Resolve the requested ref to an immutable commit SHA.
6. Insert the review and authorization evidence transactionally.
7. Enqueue or create the Kubernetes Job idempotently.
8. Return `202 Accepted` with a status resource URL.

Steps that call providers use explicit timeouts and return retryable versus permanent error classes. A provider outage never grants access.

## 6. Provider Interfaces

### 6.1 Repository Provider

```python
class RepositoryProvider(Protocol):
    def canonicalize(self, repository_url: str) -> CanonicalRepository: ...
    async def verify_access(
        self, subject: Subject, repository: CanonicalRepository, permission: Permission
    ) -> AuthorizationEvidence: ...
    async def resolve_revision(
        self, repository: CanonicalRepository, ref: str
    ) -> ResolvedRevision: ...
    async def issue_clone_grant(
        self, repository: CanonicalRepository, revision: ResolvedRevision
    ) -> CloneGrant: ...
```

Adapters normalize provider permissions and errors. `CloneGrant` is short-lived, read-only, and excluded from serialization and logs.

### 6.2 AI Provider

```python
class AIProvider(Protocol):
    async def enrich(self, request: EnrichmentRequest) -> EnrichmentResponse: ...
    async def health(self) -> ProviderHealth: ...
```

The request contains normalized findings, minimized redacted context, task type, schema version, and policy—not a repository archive. The response must pass strict schema, size, and safety validation.

### 6.3 Scanner Adapter

```python
class Scanner(Protocol):
    name: str
    version: str
    async def scan(self, repository: Path, policy: ScannerPolicy) -> ScannerResult: ...
```

Commands use fixed argument arrays with no shell. Results contain findings, tool metadata, duration, exit classification, truncation state, and coverage errors.

## 7. Scanner Worker Design

The scanner container follows a fixed pipeline:

```text
validate input -> obtain clone grant -> resolve/confirm SHA -> clone
-> run scanners -> normalize -> redact -> optional AI enrichment
-> generate reports -> upload -> notify result -> clean workspace
```

The worker creates no inbound listener. It uses a review-scoped workspace below an allowed root and refuses cleanup outside that root. Git hooks are disabled. Submodules and large-file storage are denied unless policy explicitly enables their canonical endpoints.

Each scanner receives a deadline and output limit. Tool failure produces an incomplete-coverage record; it does not become “zero findings.” Parallel execution is bounded by the Job CPU/memory budget.

## 8. Job Specification Construction

`KubernetesJobService` begins from a typed server-owned template. Variable fields are limited to opaque review ID, immutable image digest, approved resource profile, policy ID, repository identity, revision SHA, and secret references. Kubernetes names are derived from safe opaque identifiers.

Required Job controls include non-root execution, no privilege escalation, dropped capabilities, runtime-default seccomp, read-only root filesystem, no service-account token, size-limited ephemeral volumes, resource limits, active deadline, zero or bounded retry, and TTL after completion.

## 9. Persistence and Transactions

Repositories expose domain-specific methods rather than generic database sessions to services. Review creation, state transition, authorization evidence, and audit append use a single transaction where they must remain consistent.

Report artifact upload uses an idempotent finalization protocol:

1. Upload to a review-scoped temporary object key.
2. Validate size, schema, and content digest.
3. Promote or reference the immutable object.
4. Commit report metadata and final review state.
5. Delete temporary upload state.

The report is invisible until metadata and artifact integrity agree.

## 10. Error Model

Internal exceptions map to stable error codes:

| Error class | HTTP behavior | Retry |
|---|---|---|
| Invalid input | `400` or `422` | No |
| Authentication required | `401` | After authentication |
| Unauthorized/private resource | `404` where disclosure matters | No without permission change |
| Conflict/idempotency mismatch | `409` | Request correction |
| Rate or quota limit | `429` | After indicated delay |
| Provider unavailable | `503` | Bounded retry |
| Internal failure | `500` with opaque ID | Server/reconciler decision |

Responses follow a consistent problem-details schema and never include tokens, source, provider response bodies, or scanner stdout.

## 11. Configuration

Configuration is validated at startup and divided into non-secret settings, secret references, and policy documents. Invalid production defaults fail startup. Dynamic policy changes are versioned, audited, and attached to new reviews; completed reports retain the policy version used.

## 12. Testing Strategy

- Unit tests cover state transitions, authorization, canonicalization, normalization, redaction, and report sanitization.
- Contract tests validate every repository, scanner, AI, and storage adapter against its common interface.
- Integration tests cover PostgreSQL transactions, Kubernetes Job rendering/reconciliation, and object finalization.
- Security tests cover IDOR, SSRF, command injection, malicious paths, prompt injection, output leakage, and cleanup faults.
- End-to-end tests exercise Summary and Deep workflows with provider sandboxes.
- Fault injection verifies timeouts, partial scans, provider outages, Job loss, and failed cleanup.

## 13. Open Implementation Decisions

The initial implementation must still select the durable queue mechanism, authorization-evidence cache duration, report object format, exact tenant isolation level, supported maximum repository profiles, and whether the reconciler runs inside the API deployment or as a separate process. Each decision must preserve the HLD constraints and receive an ADR when it creates a durable commitment.
