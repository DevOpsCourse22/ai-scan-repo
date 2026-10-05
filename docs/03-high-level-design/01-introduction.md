# 1. Introduction

## 1.1 Purpose

This High-Level Design (HLD) defines the target architecture for **AI Scan Repos**, an AI-powered DevSecOps repository analysis platform. It describes the major system components, their responsibilities, their interactions, and the architectural constraints that guide implementation.

The HLD translates the business and functional requirements into a technology-independent solution architecture. It establishes a shared understanding for engineering, security, platform, and business stakeholders before detailed component design and implementation begin.

This document intentionally focuses on architectural structure and behavior. Class design, database columns, API payload schemas, Kubernetes manifest details, and implementation algorithms belong in the Low-Level Design (LLD) and component specifications.

---

## 1.2 Background

Modern repository assessments commonly depend on several independent tools for static application security testing, secret detection, dependency analysis, container scanning, and code-quality inspection. These tools produce findings in different formats and often provide limited context for prioritization and remediation.

AI Scan Repos addresses this fragmentation by coordinating deterministic scanners in a single review workflow, normalizing their findings, and optionally enriching those findings with AI-assisted explanations. Deterministic scanners remain the authoritative source of security findings; AI serves only as an augmentation layer for prioritization, root-cause analysis, and remediation guidance.

Repository source code is considered sensitive and untrusted. It is processed inside an isolated, ephemeral execution environment and is never retained as a permanent platform asset.

---

## 1.3 Scope

This HLD covers the architecture required to:

- Accept a Git repository URL or pull-request review request.
- Authenticate users and authorize access to protected platform capabilities.
- Verify repository access through supported repository-provider APIs.
- Schedule each repository review in an isolated Kubernetes pod.
- Clone the selected repository revision into an ephemeral workspace e.g. /tmp/.
- Scan the complete selected revision with deterministic analysis tools.
- Normalize findings into a provider-independent internal model.
- Build controlled AI context from findings and relevant source fragments.
- Produce Summary and Deep Reports according to the authorization model.
- Store report data, review metadata, and security audit events.
- Expose review capabilities through APIs and a web interface.
- Provide metrics, logs, traces, health checks, and operational alerts.
- Support deployment through Kubernetes, Helm, and Terraform.

The initial architecture includes GitHub as the primary repository provider and supports Semgrep, Gitleaks, Trivy, and a code-quality scanner. The provider and plugin interfaces are designed to accommodate additional repository, scanner, and AI integrations.

---

## 1.4 Out of Scope

The following concerns are outside the initial HLD scope:

- Automatically modifying, committing, or merging customer code.
- Executing repository build scripts or arbitrary repository-supplied binaries.
- Using AI output as the sole source of a vulnerability decision.
- Persistently storing cloned repositories or complete source archives.
- Inferring repository ownership from Git metadata, uploaded files, or naming conventions.
- Defining detailed cloud-provider resource implementations.
- Defining every database table field, API schema, or user-interface component.
- Implementing the proposed future AI router or multi-agent analysis model.

These capabilities require separate design reviews and explicit security approval before they can enter the platform scope.

---

## 1.5 Intended Audience

This document is intended for:

| Audience | Primary interest |
|---|---|
| Software Architects | System boundaries, component responsibilities, and design decisions |
| Backend Engineers | Control-plane services, APIs, persistence, and provider interfaces |
| Security Engineers | Trust boundaries, authorization, data handling, and threat controls |
| DevSecOps Engineers | Scanner orchestration, CI/CD, supply-chain security, and policy enforcement |
| Platform Engineers | Kubernetes execution, scaling, isolation, networking, and observability |
| Frontend Engineers | User workflows, report access, and API dependencies |
| Product and Engineering Leaders | Business alignment, delivery risks, limitations, and future evolution |

---

## 1.6 Architecture Drivers

The architecture is shaped by the following primary drivers:

1. **Source-code confidentiality** — repository contents must not be permanently stored and must not be disclosed to unauthorized users or providers.
2. **Deterministic coverage** — authoritative analysis must scan the full selected repository revision using deterministic tools.
3. **Strong authorization** — Deep Reports require repository access verification through the repository provider.
4. **Workload isolation** — every review must execute in a bounded, isolated, ephemeral environment.
5. **Provider independence** — repository, scanner, AI, and storage integrations must be replaceable through stable interfaces.
6. **Horizontal scalability** — independent reviews must scale without coupling scanner capacity to the API or user interface.
7. **Auditability** — security-sensitive actions and authorization decisions must be attributable and traceable.
8. **Cloud portability** — the platform must support cloud and on-premises Kubernetes deployments without embedding provider-specific behavior in core business logic.
9. **Operational resilience** — partial scanner failures, provider outages, timeouts, and cleanup failures must be visible and recoverable.

---

## 1.7 Architectural Principles

The following principles are mandatory across all components:

### Security by Design

Identity, authorization, least privilege, secure defaults, dependency integrity, network isolation, and auditability are architectural requirements rather than deployment add-ons.

### Privacy by Default

Repository source is cloned only into an ephemeral workspace associated with one review. Source code must not enter persistent logs, metrics, audit records, or AI prompts unless an approved policy explicitly permits the minimum necessary context.

### AI as an Augmentation Layer

AI explains and prioritizes deterministic findings. It does not replace security scanners, silently suppress findings, or make final authorization decisions.

### Interface-Driven Extensibility

Repository providers, AI providers, scanners, and report exporters implement common interfaces. Core workflows depend on those interfaces rather than on a specific vendor implementation.

### Stateless Control Services

API and orchestration services remain stateless wherever practical. Durable review state, reports, and audit events are stored in dedicated data services.

### Explicit Trust

Repository names, URLs, uploaded archives, Git history, and client-supplied metadata are untrusted. Trust is established through verified identity and live repository-provider authorization.

---

## 1.8 Assumptions and Constraints

The HLD assumes:

- The platform runs on a Kubernetes cluster that supports Jobs, resource quotas, network policies, and the restricted Pod Security profile.
- Scanner workloads are constrained by non-root execution, read-only root filesystems, no privilege escalation, runtime security profiles such as Seccomp/AppArmor, ephemeral storage, and a prohibition on directly executing uploaded untrusted code.
- An external identity provider supports standards-based authentication such as OpenID Connect.
- Repository providers expose APIs for verifying user or application access.
- Repository clone credentials can be short-lived and limited to read-only access.
- PostgreSQL is available for transactional metadata and audit records.
- Report objects can be encrypted and retained according to organizational policy.
- Approved AI providers expose a bounded request/response interface, or AI processing can be disabled.
- Platform operators control the scanner images, rules, and plugin allowlist.

Key constraints include:

- Source repositories may contain malicious files, excessive data, secrets, or prompt-injection content.
- Scanner execution time and resource consumption vary significantly by repository size and language mix.
- Network access from scanner workloads must be tightly controlled without preventing approved repository access and vulnerability-database updates.
- Summary Reports must not expose sensitive implementation details to anonymous or unverified users.

---

## 1.9 Requirements Alignment

The architecture directly supports the Business Requirements Document:

| Requirement area | HLD response |
|---|---|
| Unified repository analysis | Review orchestration and normalized finding model |
| Code-quality and security analysis | Pluggable deterministic scanner layer |
| AI-assisted reasoning | Policy-controlled AI Analysis Engine |
| Summary and Deep Reports | Tiered reporting and authorization controls |
| Ownership verification | Repository-provider permission API integration |
| Ephemeral processing | Isolated Kubernetes Job and temporary workspace |
| Provider independence | Adapter and plugin interfaces |
| Cloud-agnostic deployment | Kubernetes, Helm, and infrastructure modules |
| Scalability and reliability | Stateless control plane and horizontally scalable Jobs |
| Observability | Central metrics, structured logs, traces, and audit events |

Detailed traceability between individual BR, FR, security control, component, and verification test will be maintained as the design matures.

**POC scope note:** "pluggable"/"plugin"/"adapter interfaces" in this table describe the interface design goal. This project implements one concrete instance of each — GitHub, Semgrep/Gitleaks/Trivy, one local AI provider — not a dynamically loadable multi-provider system. See [Scope Decisions Log](../SCOPE-DECISIONS.md).

---

## 1.10 Document Conventions

The words **must**, **must not**, **required**, **shall**, and **shall not** indicate mandatory architectural constraints. The words **should** and **recommended** indicate a preferred design that may be changed only with documented justification.

Architecture decisions with long-term consequences are recorded as Architecture Decision Records under `docs/adr/`. Diagrams referenced by HLD chapters are maintained as editable sources under `docs/diagrams/`. When a chapter and an ADR differ, the accepted ADR takes precedence until the HLD is updated.

The next chapter, [Solution Overview](02-solution-overview.md), introduces the end-to-end platform architecture and the responsibilities of its major subsystems.
