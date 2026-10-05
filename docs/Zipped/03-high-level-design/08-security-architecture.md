# 8. Security Architecture

## 8.1 Security Model

AI Scan Repos assumes that users, repositories, scanner output, provider responses, and AI responses can all be malicious or malformed. Security controls therefore use defense in depth across identity, input handling, workload isolation, network policy, secrets, data protection, software supply chain, and monitoring.

## 8.2 Data Classification

| Data class | Examples | Handling |
|---|---|---|
| Restricted | Source code, detected secrets, clone credentials, identity tokens | Ephemeral or secret-managed; never logged; tightly scoped access |
| Confidential | Deep Reports, file paths, snippets, authorization evidence | Encrypted; repository-authorized access; retention policy |
| Internal | Review metadata, scanner coverage, operational audit events | Authenticated access; integrity and retention controls |
| Public | Approved Summary Reports and product documentation | Explicit sanitization and publication decision |

Classification follows the most sensitive included field. A report does not become public merely because its source repository is public; publication remains a policy decision.

## 8.3 Control Domains

### Identity and Access

OIDC, workload identity, provider-level repository checks, least-privilege roles, server-side report authorization, and audited elevation protect platform actions. Service accounts are unique per workload.

### Input and Execution Safety

Repository URLs are canonicalized and restricted to configured providers. Refs follow a strict format and resolve to immutable SHAs. Scanner commands use fixed executables and argument arrays. The platform does not run Git hooks, repository builds, package installation, or repository-supplied plugins in the initial scope.

### Workload Isolation

Scanner Jobs run as non-root with dropped capabilities, runtime-default seccomp, read-only root filesystems, no Kubernetes token, bounded resources, and ephemeral storage. High-risk deployments may add sandboxed runtimes or dedicated nodes.

### Network Security

Namespaces use default-deny network policy. Scanner Jobs accept no inbound traffic. Egress is limited to approved provider, vulnerability database, object-store, AI, DNS, and telemetry endpoints through controlled paths. Private and link-local address ranges are blocked for repository-originated destinations to reduce SSRF risk.

### Secrets Management

Secrets are generated, stored, and rotated through an external secret manager. Short-lived workload identities are preferred. Clone credentials are read-only, repository-scoped, and review-scoped where supported. Secrets never appear in environment dumps, command arguments, URLs, logs, metrics, reports, or Job specifications visible to broad readers.

### Data Protection

All external and service-to-service traffic uses authenticated encryption appropriate to the environment. Persistent data and backups are encrypted with controlled keys. Deep Reports use repository-scoped access checks and optional tenant-specific keys. Integrity metadata detects report replacement or corruption.

### Software Supply Chain

Dependencies and base images are pinned and continuously scanned. Builds produce provenance and SBOMs. Images are signed and admitted by digest. CI identities use federation, protected environments, mandatory review, and separation between build and production deployment authority.

## 8.4 Key Threats and Architectural Responses

| Threat | Architectural response |
|---|---|
| SSRF through repository or submodule URL | Provider allowlist, canonical IDs, redirect validation, controlled egress |
| Command injection through ref or filename | Strict validation, no shell, server-owned commands |
| Repository exploits scanner | Patched tools, restricted Job, resource bounds, sandbox option |
| Secret disclosed in output | Detection-time redaction, evidence references, output scanning |
| Cross-tenant report access | Resource authorization at service and storage boundary, opaque IDs, denial tests |
| Prompt injection in source | Treat code as data, fixed system policy, minimized context, schema validation |
| Malicious or hallucinated AI result | Advisory labeling, provenance, deterministic source preserved |
| Denial of service by large repository | Size/file/depth limits, quotas, deadlines, queue controls |
| Credential theft from Job | Short lifetime, no API token, narrow egress, memory/volume cleanup |
| Persistence after scan | Ephemeral volume, explicit cleanup, Pod deletion, Job TTL, alerts |

The detailed threat model is maintained separately and must be updated whenever the execution model, data retention, provider set, or tenancy model changes.

## 8.5 Secure Logging and Audit

Application logs are structured and pass through redaction before export. They contain opaque review IDs and operational state, not repository contents, tokens, clone URLs with credentials, secret matches, prompts, or model responses. Security audit events record actor, action, resource, authorization decision, policy version, timestamp, correlation ID, and outcome.

Audit storage is append-only or tamper-evident, access-controlled, and retained according to policy. Monitoring distinguishes operational logs from security audit records.

## 8.6 Vulnerability and Incident Management

Platform images, dependencies, IaC, and manifests are scanned in CI and periodically in deployed environments. Critical exposure has an owned remediation SLA. Incident response supports credential revocation, Job termination, report-access suspension, evidence preservation without source-code overcollection, and notification under applicable policy.

## 8.7 Security Verification

Production approval requires automated authorization tests, container and dependency scanning, IaC policy tests, network-policy verification, secret-leak testing, cleanup fault injection, backup restoration, and independent threat-model review. Controls are revalidated after material architecture changes.

Next: [AI Analysis Engine](09-ai-analysis-engine.md).
