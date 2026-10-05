# Threat Model

**Project:** AI Scan Repos  
**Document Version:** 1.0  
**Status:** Draft  
**Method:** STRIDE-informed data-flow analysis

---

## 1. Purpose

This threat model identifies plausible attacks against AI Scan Repos and the controls required to reduce their likelihood or impact. It covers the web/API edge, identity, repository-provider integrations, Kubernetes control plane, ephemeral scanner Jobs, data stores, AI providers, telemetry, and software supply chain.

The model must be revisited when the platform adds a provider, executes repository code, changes tenancy, changes retention, introduces autonomous AI tools, or moves a trust boundary.

## 2. Assets

Primary assets are:

- Customer repository source and intellectual property.
- Repository clone credentials and provider application credentials.
- User sessions, identity tokens, and workload identities.
- Deep Reports, detected-secret evidence, and remediation details.
- Authorization evidence and tenant policy.
- Review integrity, scanner provenance, and audit history.
- Kubernetes, database, storage, CI/CD, and encryption-key authority.
- Platform availability and tenant resource fairness.

## 3. Adversaries

| Adversary | Capability and objective |
|---|---|
| Anonymous attacker | Enumerate repositories/reports, exploit API inputs, consume capacity |
| Malicious authenticated user | Access another tenant, submit hostile repositories, abuse valid quotas |
| Malicious repository contributor | Craft files, names, history, submodules, or prompts to exploit scanners and AI |
| Compromised provider account/app | Return misleading authorization or issue excessive access |
| Compromised dependency/image | Execute within backend, scanner, or CI trust zone |
| Privileged insider | Misuse administrative, database, storage, or observability access |
| External provider | Retain or misuse data sent for identity, repository, or AI operations |

## 4. Trust Boundaries

1. Public clients to edge and API.
2. Edge to application control plane.
3. Control plane to identity and repository providers.
4. Control plane to Kubernetes API and durable data services.
5. Control plane to the untrusted scanner Job.
6. Repository provider to scanner workspace.
7. Scanner to AI and report-storage providers.
8. All workloads to telemetry and audit systems.
9. CI/CD to artifact registry and production deployment.

## 5. Threat Analysis

### 5.1 Identity and Authorization

| Threat | STRIDE | Controls | Validation |
|---|---|---|---|
| Stolen or forged identity token | Spoofing | OIDC signature/issuer/audience/expiry validation, PKCE, secure session, MFA for privilege | Invalid token suite, session fixation and replay tests |
| User claims repository ownership | Spoofing/Elevation | Canonical provider API verification; ignore Git metadata and client claims | Cross-account and fabricated repository tests |
| IDOR exposes another review/report | Information disclosure | Resource authorization on every access, opaque IDs, tenant scoping, non-disclosing denial | Automated cross-user/tenant matrix |
| Administrator reads source by default | Elevation/Disclosure | Separation of platform role and repository permission, audited break-glass | Privileged-access tests and audit review |
| Cached access remains after revocation | Elevation | Short evidence TTL, provider events, revalidation for Deep access | Membership revocation integration test |

### 5.2 Repository Acquisition and Processing

| Threat | STRIDE | Controls | Validation |
|---|---|---|---|
| Repository URL triggers SSRF | Spoofing/Disclosure | Provider allowlist, canonical ID, IP-range denial, redirect validation, egress gateway | Private/metadata/DNS-rebinding test corpus |
| Ref or filename causes command injection | Tampering/Elevation | Strict validation, no shell, fixed argument arrays | Metacharacter and Unicode filename tests |
| Git hook or build runs attacker code | Elevation | Hooks disabled; no builds/installers/repository executables | Malicious hook/build fixtures |
| Scanner parser is exploited | Elevation | Patched pinned tools, non-root Job, seccomp, no token, network/resource bounds | Scanner fuzzing, runtime-policy exercise |
| Archive/path escapes workspace | Tampering | Safe Git client behavior, canonical relative paths, allowed-root checks | Path traversal and symlink fixtures |
| Repository exhausts CPU/storage/time | Denial of service | Size/file/depth limits, quota, resource limits, deadlines, concurrency control | Oversized repository load tests |

### 5.3 Credentials and Secrets

| Threat | STRIDE | Controls | Validation |
|---|---|---|---|
| Clone token appears in logs or process list | Disclosure | Secret mount/workload identity, URL redaction, no argument token, structured errors | Log and process inspection tests |
| Compromised Job reuses credential elsewhere | Spoofing/Elevation | Review/repository-scoped read-only short-lived grant, narrow egress, revocation | Scope and expiry tests |
| Detected secret is reproduced in report/AI | Disclosure | Redact at normalization, evidence references, prompt/report scanning | Seeded canary-secret tests |
| Kubernetes secret readable by API broadly | Elevation/Disclosure | External secret manager, workload identity, narrow RBAC | RBAC `can-i` and policy tests |

### 5.4 AI-Specific Threats

| Threat | STRIDE | Controls | Validation |
|---|---|---|---|
| Source prompt injection changes model behavior | Tampering | Code delimited as data, server-owned prompt, no tools, minimized context | Adversarial repository evaluation set |
| Hosted provider retains confidential data | Disclosure | AI disabled by default, provider/region policy, no-training/no-retention terms | Configuration and contract review |
| AI fabricates or suppresses a finding | Tampering/Repudiation | Deterministic finding preserved, advisory fields separated, provenance | Schema and comparison tests |
| Model response reflects secret or unsafe link | Disclosure/Tampering | Response schema, redaction, link policy, output-size bounds | Malicious-provider fixtures |
| AI cost/latency exhaustion | Denial of service | Token/context budgets, concurrency limits, timeout, circuit breaker | Load and forced-timeout tests |

### 5.5 Data and Reporting

| Threat | STRIDE | Controls | Validation |
|---|---|---|---|
| Summary leaks private file details | Disclosure | Allowlisted Summary schema generated separately from Deep view | Snapshot and property tests with seeded sensitive data |
| Report object key is guessed | Disclosure | Opaque keys, server authorization, short signed URLs, storage identity policy | Direct object access tests |
| Report is replaced or mismatched | Tampering | Immutable key/version, digest, revision and schema provenance | Integrity-failure simulation |
| Audit event is changed or deleted | Tampering/Repudiation | Append-only/tamper-evident storage, restricted roles, archival | Audit integrity and role tests |
| Retained backup violates deletion expectation | Disclosure | Documented backup lifecycle, encryption, delayed-erasure policy | Retention and restore inspection |

### 5.6 Platform and Supply Chain

| Threat | STRIDE | Controls | Validation |
|---|---|---|---|
| API creates attacker-controlled Pod | Elevation | Server-owned typed Job template, namespaced RBAC, admission policy | Mutation and policy test suite |
| Scanner moves laterally | Elevation/Disclosure | No service token, default-deny network, dedicated identity/nodes | Network reachability tests |
| Malicious image enters production | Tampering | Provenance, SBOM, scan, signature, digest admission | Unsigned-image rejection test |
| CI credential deploys arbitrary production change | Elevation | Federated scoped identities, protected environments, GitOps approval | IAM and pipeline policy review |
| Cleanup controller fails | Disclosure | Worker cleanup, ephemeral volume, Pod removal, TTL, stale-resource alarm | Node/Job failure injection |

## 6. Abuse Cases

- An attacker submits thousands of small repositories to monopolize scanner capacity.
- A user tests private repository names to infer their existence through timing or errors.
- A repository contains enormous generated files and recursive submodule references.
- A secret scanner finds a credential that another scanner or AI provider then echoes.
- A malicious README instructs the model to ignore scanner findings and reveal its prompt.
- A compromised scanner writes a valid-looking report for the wrong commit.
- A support operator queries Deep Reports without repository permission.

Each abuse case must map to rate/quota, authorization, validation, isolation, provenance, and audit tests before production release.

## 7. Residual Risks

- Scanner zero-days can execute inside the Job despite containment controls.
- Static analysis cannot prove that a repository is safe or complete.
- Provider authorization may be temporarily stale within an approved cache window.
- Secret detection and redaction may not identify novel credential formats.
- Local AI avoids external transfer but remains vulnerable to malicious context and inaccurate output.
- Physical deletion from storage snapshots may lag logical deletion under documented policy.

Residual risks require named ownership and explicit acceptance at production readiness review.

## 8. Security Test Gates

Release gates include authorization matrix tests, SSRF tests, malicious Git fixtures, output canary secrets, prompt-injection evaluations, container/IaC policy, signed image admission, RBAC/network verification, dependency scanning, cleanup fault injection, and report integrity checks.

## 9. Malicious-Repository Controls: What Is Actually Built (2026-09-06)

Sections 5–8 describe the target. This section records what the current
build does, verified against the code rather than assumed, so the gap is
visible instead of implied.

### Structurally prevented: execution of repository content

The strongest control is that **repository content is never executed**.
No build, no dependency install, no test run. The only `subprocess` calls
in the ingest and scanner paths are our own `git`, our own process
cleanup, and the scanner binaries — all of which *parse* the repository
rather than run it. Clones are `--depth 1` and do not recurse submodules,
and `git clone` does not run remote-supplied hooks.

This is deliberate and matches the roadmap: repository build or dynamic
execution is a separate high-risk program requiring sandbox architecture
(Phase 7), not something to arrive at by accident.

### Mitigated but not proven: prompt injection

Repository content reaches the model wrapped in an explicit instruction
that it is untrusted data and that instructions inside it must not be
followed (`mcp-server/src/code_review_mcp/llm.py`). ADR-009 extends the
same reasoning to third-party findings.

This reduces the risk; it does not close it. LLM injection defences are
probabilistic. The realistic scenario is a repository owner hiding their
own vulnerabilities to obtain a clean report to show someone else. The
prompt-injection evaluations that would give this an acceptance threshold
are Phase 4/5 exit criteria and **have not been run**.

### Partially bounded: resource exhaustion

Limits that exist: `max_files` (500), chunked file content, a 300s clone
timeout and a 900s per-scanner timeout. Not verified: behaviour on a
single enormous file, on tens of thousands of small files, or on
pathological archive content. The abuse case in Section 6 is listed; the
corresponding test is not written.

### Open and currently unmitigated: report accumulation

Reports are written to shared storage with **no retention limit**, so
repeated review requests fill disk indefinitely. This is tracked as a
product gap (the UI already tells users their reports are not kept, which
is currently untrue) and is now also recorded here, because unbounded
growth driven by an untrusted caller is an availability threat, not only
an untidy directory. A delete path exists
(`DELETE /api/v1/reports/<name>`, exposed through the gateway with the
same ownership check as reading); automatic expiry does not.

### No path found: attacker-controlled write locations

Repository-supplied names never determine where anything is written.
Report filenames derive from a server-generated `review_id`, and findings
files are named by the caller of the scanner, not by scanned content.

## 10. Review Cadence

The threat model is reviewed at least each major release and after an incident, new provider, new scanner, new data flow, tenancy change, execution capability, or material infrastructure change. Findings feed the security backlog, ADRs, design documents, and verification plan.
