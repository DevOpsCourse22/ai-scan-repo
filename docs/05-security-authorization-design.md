# Security and Authorization Design

**Project:** AI Scan Repos  
**Document Version:** 1.0  
**Status:** Draft  
**Owner:** Security Architecture

---

## 1. Purpose and Scope

This document defines identity, authorization, data protection, workload isolation, secret handling, secure delivery, and audit controls for AI Scan Repos. It expands the HLD security chapters into controls that can be implemented and verified.

Repository source, clone credentials, identity tokens, Deep Reports, and detected secret evidence are sensitive assets. Source repositories are also untrusted inputs that may attempt to exploit scanners, exhaust resources, redirect network traffic, or influence AI output.

## 2. Security Principles

- Deny by default and grant the minimum operation for the minimum time.
- Authenticate identity and authorize repository access as separate decisions.
- Treat every repository, filename, tool result, and AI response as untrusted.
- Keep source ephemeral and out of control-plane services.
- Enforce report authorization on the server and at storage access boundaries.
- Prefer short-lived workload identity over static credentials.
- Fail closed when authorization evidence cannot be obtained.
- Make security-sensitive decisions attributable and auditable.

## 3. Identity Architecture

Human users authenticate through an approved OpenID Connect provider using Authorization Code Flow with PKCE. Token validation includes signature, issuer, audience, expiry, not-before, state, nonce, and required claims. External subjects are mapped by `(issuer, subject)` to an internal opaque user ID.

CI integrations use federated workload identities or narrowly scoped application credentials. Runtime services use distinct Kubernetes service accounts and cloud workload identities. Scanner Jobs do not receive Kubernetes API tokens.

Sessions use secure, HTTP-only, same-site cookies when managed by a backend-for-frontend. Bearer tokens are not stored in browser local storage. Privileged administration requires stronger authentication and time-bounded elevation.

## 4. Authorization Model

Authorization combines platform role, tenant membership, canonical repository identity, repository-provider permission, requested action, report level, review ownership, and policy version.

| Resource/action | Required authorization |
|---|---|
| Public Summary Report | Repository confirmed public and publication policy permits it |
| Private Summary Report | Authenticated subject with provider-approved repository access |
| Deep Report | Authenticated subject with current or policy-valid provider permission |
| Create private review | Provider-approved repository read permission |
| Cancel review | Request owner or delegated tenant role |
| Change tenant policy | Tenant administrator with audited privileged session |
| Platform support access | Explicit, time-bound, reasoned break-glass policy |

Platform administrators do not automatically receive repository-content access. Private resource denials use non-disclosing behavior, normally `404`, to reduce enumeration.

## 5. Repository Ownership Verification

Ownership and access are verified through the repository-provider API using the canonical external repository ID. Uploaded source, `.git` contents, commit authorship, organization names, URLs, and repository names are never proof of ownership.

Authorization evidence records only:

- Internal subject and tenant.
- Provider and canonical repository ID.
- Normalized permission class.
- Requested operation.
- Decision and policy version.
- Provider check timestamp and expiry.

Provider tokens and complete provider responses are not stored. Permission evidence has a short policy-defined lifetime and is invalidated on membership, installation, or tenant changes where provider events support it.

## 6. Data Classification and Handling

| Classification | Examples | Control baseline |
|---|---|---|
| Restricted | Source, clone grants, tokens, detected secret values, AI context | Ephemeral/secret-managed, never logged, narrow network and identity scope |
| Confidential | Deep Reports, paths, snippets, authorization evidence | Encryption, repository authorization, bounded retention |
| Internal | Review state, aggregate operational metadata, audit events | Authenticated access, integrity, retention policy |
| Public | Explicitly approved sanitized summaries | Allowlisted publication fields and repository visibility check |

Summary Reports are generated from an allowlisted schema. They never include file paths, line numbers, snippets, secret evidence, private dependency names, prompts, or detailed remediation tied to source.

## 7. Secrets and Credentials

Secrets reside in an approved secret manager. Workloads obtain them through workload identity or short-lived mounts. No secret is committed to Git, Helm values, container images, Terraform state in plaintext, Job labels, command arguments, or logs.

Clone authorization is read-only, limited to one repository and review where supported, and expires within the review window. Credentials are revoked after cancellation or suspicious activity where provider capability permits it. Redaction filters cover headers, URLs, exception strings, scanner output, and environment diagnostics.

## 8. Workload and Runtime Security

Scanner Jobs run with:

- Non-root user and group.
- `allowPrivilegeEscalation: false`.
- All Linux capabilities dropped.
- Runtime-default seccomp profile.
- Read-only root filesystem.
- No host namespaces, sockets, devices, or paths.
- No automounted service-account token.
- Size-limited ephemeral volumes.
- CPU, memory, storage, process, time, and retry bounds.

The platform does not run repository build scripts, package installers, Git hooks, or repository-supplied executables in the initial design. High-assurance environments should use a sandboxed runtime or dedicated scanner nodes.

## 9. Network Security

The namespace starts with default-deny ingress and egress. Scanner Jobs accept no inbound connections and cannot reach the platform database or other Jobs. Egress is restricted to DNS, configured repository providers, approved vulnerability databases, review-scoped report upload, telemetry, and the selected AI endpoint.

Repository URLs, redirects, submodules, and large-file endpoints are canonicalized and checked against provider and address policy. Private, loopback, link-local, metadata-service, and cluster networks are blocked for repository-controlled destinations. A hostname-aware egress proxy is recommended because portable Kubernetes NetworkPolicy cannot express domain allowlists.

## 10. AI Security

AI is disabled until tenant and platform policy approve a provider, model class, region, retention/training terms, and data classification. The Context Builder sends only redacted findings and minimum relevant code fragments within strict limits.

Repository content is delimited as data and cannot provide model instructions. AI receives no autonomous shell, repository-write, network, deployment, or secret tools. Responses must match a versioned schema and are scanned for reflected secrets, unexpected instructions, unsafe links, and unsupported output size. AI never suppresses deterministic findings or decides authorization.

## 11. Application and API Security

APIs apply strict schemas, body limits, repository host allowlists, ref validation, output encoding, parameterized data access, rate limits, and idempotency. Commands use argument arrays rather than a shell. Browser endpoints apply CSRF controls appropriate to the session model and restrictive CORS.

Errors use stable codes and opaque correlation identifiers. Internal provider messages, SQL details, filesystem paths, source, and credentials are never returned to clients.

## 12. Supply-Chain Security

CI performs secret detection, SAST, dependency review, license policy, IaC validation, container scanning, and tests. Builds use pinned dependencies and minimal base images, generate SBOM and provenance, sign images, and publish immutable digests. Admission policy permits only approved registries and verified digests with restricted workload settings.

Production deployment identity is separated from build identity. Prefer federated CI authentication over long-lived cloud keys. Critical dependency and image vulnerabilities follow an owned remediation SLA.

## 13. Audit and Monitoring

Audit events cover authentication, repository access decisions, review creation/cancellation, report access, policy changes, provider configuration, privileged elevation, credential lifecycle, and deletion. Records include actor, action, resource, policy, decision, timestamp, correlation ID, and outcome without sensitive content.

Alerts cover cross-tenant access attempts, unusual authorization failure, credential misuse, unexpected scanner network/process activity, report-integrity failure, cleanup failure, and administrative changes. Audit storage is tamper-evident and access-controlled separately from ordinary logs.

## 14. Verification Requirements

Production readiness requires:

- Unit and integration tests for every authorization rule.
- IDOR and tenant-boundary testing.
- SSRF, redirect, malicious ref, and path traversal tests.
- Scanner escape and resource-exhaustion testing.
- Secret leakage tests across logs, reports, AI, and errors.
- Network-policy and Kubernetes RBAC validation.
- Signed-image admission and dependency policy tests.
- Forced-failure cleanup and retention verification.
- Backup restore and incident-response exercises.
- Threat-model review after material design changes.

## 15. Security Exceptions

Any exception to a mandatory control requires documented scope, business justification, risk owner, compensating controls, expiration date, and approval. Exceptions do not silently become architecture defaults and are reviewed before renewal.
