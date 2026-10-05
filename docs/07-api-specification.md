# API Specification

**Project:** AI Scan Repos  
**API Version:** v1  
**Document Status:** Draft

---

## 1. Overview

The AI Scan Repos REST API exposes authentication context, repository reviews, review cancellation, report retrieval, and health endpoints. The runtime OpenAPI document is authoritative for concrete schemas; this document defines the intended resource model and security behavior.

Base path:

```text
/api/v1
```

All traffic uses HTTPS. JSON is the default representation. Timestamps use UTC RFC 3339. Identifiers are opaque UUIDs and must not encode repository or tenant information.

## 2. Common Headers

| Header | Direction | Purpose |
|---|---|---|
| `Authorization: Bearer <token>` | Request | OIDC access token where authentication is required |
| `Idempotency-Key` | Request | Prevent duplicate review creation; scoped to subject and operation |
| `X-Request-ID` | Both | Optional client correlation ID; server validates or replaces it |
| `Retry-After` | Response | Delay for quota, rate, or temporary capacity conditions |
| `ETag` / `If-None-Match` | Both | Conditional status/report retrieval |

Browser session deployments may use secure cookies instead of exposing bearer tokens to JavaScript.

## 3. Resource Model

### 3.1 Create Review

```http
POST /api/v1/reviews
```

```json
{
  "repository_url": "https://github.com/example/project",
  "ref": "main",
  "report_level": "deep",
  "pull_request": null
}
```

Validation rules:

- URL must use HTTPS, contain no credentials, and match a configured provider.
- `ref` is optional, length-limited, and limited to an approved character set.
- `report_level` is `summary` or `deep`.
- Pull-request identifier and ref semantics are provider-normalized.
- Execution image, command, environment, namespace, and arbitrary scanner flags are not accepted.

Successful response:

```http
HTTP/1.1 202 Accepted
Location: /api/v1/reviews/018f...
```

```json
{
  "id": "018f4f02-7f89-7e6c-ae21-6abed09e0f74",
  "status": "queued",
  "report_level": "deep",
  "repository": {
    "provider": "github",
    "display_name": "example/project"
  },
  "revision": {
    "requested_ref": "main",
    "commit_sha": "7b4f..."
  },
  "created_at": "2026-07-14T10:00:00Z",
  "links": {
    "self": "/api/v1/reviews/018f...",
    "report": "/api/v1/reviews/018f.../report"
  }
}
```

Deep review creation requires authenticated identity and provider permission. Public Summary policy may permit anonymous creation with stricter rate and publication controls.

### 3.2 Get Review Status

```http
GET /api/v1/reviews/{review_id}
```

Response status is one of:

```text
requested, authorizing, queued, running, finalizing,
completed, failed, cancelled, timed_out
```

The response may include phase, progress category, scanner coverage, timestamps, and sanitized error code. It never returns raw Pod state, scanner stdout, credentials, or private failure details.

### 3.3 List Reviews

```http
GET /api/v1/reviews?status=completed&limit=25&cursor=<opaque>
```

Results are always scoped to the authorized subject and tenant. Cursor pagination is preferred over offsets. Maximum page size is server-defined. Sort order defaults to creation time descending.

### 3.4 Cancel Review

```http
POST /api/v1/reviews/{review_id}/cancel
```

Cancellation is idempotent. The requester must own the review or have delegated tenant permission. A completed review remains completed; an in-progress cancellation returns `202` until reconciliation confirms Job termination and cleanup.

### 3.5 Retrieve Report

```http
GET /api/v1/reviews/{review_id}/report?view=summary
GET /api/v1/reviews/{review_id}/report?view=deep
```

Summary and Deep views are distinct server-generated schemas. Deep retrieval re-evaluates resource authorization according to policy. A private or unauthorized report normally returns `404` to avoid disclosing existence.

Summary response example:

```json
{
  "review_id": "018f...",
  "revision": "7b4f...",
  "status": "complete",
  "scores": {"security": 72, "quality": 84},
  "counts": {"critical": 1, "high": 3, "medium": 8, "low": 12},
  "coverage": ["semgrep", "gitleaks", "trivy", "quality"],
  "generated_at": "2026-07-14T10:10:00Z"
}
```

Deep response adds authorized normalized findings:

```json
{
  "findings": [
    {
      "id": "fnd_018f...",
      "scanner": "semgrep",
      "rule_id": "python.security.example",
      "severity": "high",
      "title": "Unsafe operation",
      "location": {"path": "app/service.py", "start_line": 42, "end_line": 44},
      "evidence": "Redacted evidence",
      "remediation": "Use the safe API and validate input.",
      "ai": {"status": "completed", "priority": "high", "explanation": "..."},
      "provenance": {"scanner_version": "...", "schema_version": "1"}
    }
  ]
}
```

Detected secret values are never included.

### 3.6 Current Identity

```http
GET /api/v1/auth/me
```

Returns the internal display identity, tenant memberships, and platform roles safe for the client. It does not return provider access tokens or a complete provider permission cache.

### 3.7 Health and Metrics

```http
GET /healthz
GET /readyz
GET /metrics
```

Liveness proves the process is running. Readiness verifies dependencies required to accept new traffic according to deployment policy. Metrics are not publicly exposed and do not contain repository or user labels.

## 4. Problem Details

Errors follow an RFC 9457-style structure:

```json
{
  "type": "https://docs.example.invalid/problems/repository-access-denied",
  "title": "Resource not found",
  "status": 404,
  "code": "RESOURCE_NOT_FOUND",
  "request_id": "req_018f..."
}
```

Validation errors may add a bounded `errors` array containing safe field names and codes. Stack traces, provider bodies, source paths, SQL, tokens, and internal object keys are excluded.

## 5. Status Codes

| Code | Meaning |
|---:|---|
| `200` | Successful retrieval or completed idempotent action |
| `202` | Review accepted or cancellation still reconciling |
| `400` | Semantically invalid request |
| `401` | Authentication is required or invalid |
| `404` | Resource absent or intentionally non-disclosed |
| `409` | State conflict or idempotency-key payload mismatch |
| `422` | Schema validation failure |
| `429` | Rate, quota, or concurrency limit exceeded |
| `500` | Opaque internal failure |
| `503` | Required provider or capacity temporarily unavailable |

## 6. Idempotency and Concurrency

Review creation stores a hash of the canonical request with the idempotency key. Reusing the same key and payload returns the original result; reusing it with a different payload returns `409`. State-changing operations use durable state versions internally. Clients use ETags for efficient polling, not for bypassing service transition rules.

## 7. Rate Limits and Quotas

Limits apply by IP for anonymous traffic and by subject, tenant, repository, and operation for authenticated traffic. Review admission also enforces concurrent Job and resource-profile quotas. Headers communicate safe remaining/request-reset information without exposing other tenants.

## 8. API Security

- CORS uses an explicit origin allowlist.
- Cookie-based state changes use CSRF protection.
- Request bodies, strings, result pages, and output objects have size bounds.
- Repository and report access are authorized on every endpoint.
- Signed report URLs are short-lived, single-purpose, and issued only after authorization.
- API logs contain request IDs and safe error codes, not request bodies or authorization headers.

## 9. Versioning

Breaking changes create a new API major version. Additive fields may be introduced in v1; clients must ignore unknown response fields. Report and finding schemas have independent explicit versions because stored artifacts outlive API deployments.
