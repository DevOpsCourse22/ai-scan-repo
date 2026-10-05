# Business Requirements Document (BRD)

**Project:** AI Scan Repos

**Document Version:** 1.0

**Status:** Draft

**Owner:** AI Scan Repos Team

---

# 1. Purpose

This document defines the business requirements for **AI Scan Repos**, an AI-powered DevSecOps Repository Analysis Platform.

The purpose of this document is to establish the business objectives, functional requirements, non-functional requirements, constraints, assumptions, and success criteria that guide the solution architecture and future implementation.

---

# 2. Scope

AI Scan Repos provides automated repository analysis by combining deterministic security scanners with AI-assisted reasoning.

The platform enables organizations to evaluate:

- Security vulnerabilities
- Exposed secrets
- Dependency risks
- Code quality

The platform supports multiple report levels while ensuring customer source code is never permanently stored.

---

# 3. Business Problem

Modern software development depends on numerous independent analysis tools.

Examples include:

- Static Application Security Testing (SAST)
- Secret Detection
- Dependency Vulnerability Scanning
- Container/IaC Configuration Scanning
- Code Quality Analysis
- AI Code Review

Although each tool provides valuable information, their findings are typically isolated and difficult to prioritize.

Organizations often spend significant effort correlating reports before developers can begin remediation.

Current AI-based solutions frequently lack deterministic validation, while traditional scanners lack contextual explanations and remediation guidance.

---

# 4. Business Objectives

The platform aims to:

- Consolidate repository analysis into a single workflow.
- Reduce repository review time.
- Improve remediation quality.
- Increase developer productivity.
- Reduce security risk.
- Protect customer intellectual property.
- Support enterprise deployment models.
- Support multiple AI providers.
- Remain cloud agnostic.

---

# 5. Stakeholders

| Stakeholder | Responsibility |
|-------------|----------------|
| Developers | Submit repositories and consume reports |
| Security Engineers | Review findings and manage security posture |
| DevOps Engineers | Deploy and maintain the platform |
| Platform Administrators | Configure infrastructure and integrations |
| Engineering Managers | Monitor repository health |
| Executive Management | Track organizational security posture |

---

# 6. Business Requirements

- BR-001  
    The platform shall provide automated repository analysis.

- BR-002  
    The platform shall consolidate security findings into a unified report.

- BR-003  
    The platform shall support code-quality analysis.

- BR-004  
    The platform shall support AI-assisted reasoning.
- BR-005  
    The platform shall provide two report levels.
    - Summary Report
    - Deep Report

- BR-006  
    Deep Reports shall only be available after successful repository ownership verification.

- BR-007  
    Repository ownership shall be verified using repository-provider APIs.

- BR-008  
    The platform shall never infer repository ownership from uploaded source code, Git metadata, or repository naming conventions.

- BR-009  
    The platform shall execute repository analysis inside isolated ephemeral execution environments.

- BR-010  
    Repository contents shall be deleted automatically after analysis.

- BR-011  
    The platform shall support multiple repository providers.  
    
    Initial providers:
    - GitHub  
    
    Future providers:
    - GitLab
    - Azure DevOps
    - Bitbucket

- BR-012  
    The platform shall define a provider abstraction layer for AI integration. This project implements one working local provider against that interface; supporting additional providers is a future enhancement, not a current requirement.

---

# 7. Functional Requirements

## Repository Management

- FR-001  
    Accept Git repository URLs.

- FR-002  
    Support Pull Request analysis.

- FR-003  
    Clone repositories into temporary workspaces.

---

## Repository Verification

- FR-004  
    Authenticate users through supported identity providers.

- FR-005  
    Verify repository ownership before granting Deep Reports.

- FR-006  
    Validate repository permissions through provider APIs.

---

## Analysis Engine

- FR-007  
    Perform deterministic analysis across the entire repository.

    Supported scanners include:

    - Semgrep
    - Gitleaks
    - Trivy

    The architecture uses a scanner interface designed to support additional scanners in the future; this project implements exactly these three.

- FR-008  
    Normalize scanner findings into a common internal format.

- FR-009  
    Generate AI context from deterministic findings.

- FR-010  
    Produce AI-assisted analysis.

---

## Reports

- FR-011  
    Generate Summary Reports.

- FR-012  
    Generate Deep Reports.

- FR-013  
    Generate JSON output.

- FR-014  
    Generate Markdown reports.

---

## Dashboard

- FR-015  
    Display repository history.

- FR-016  
    Display risk scores.

- FR-017  
    Display review history.

---

## API

- FR-018  
    Expose REST APIs.

---

# 8. Authorization Model

| User | Summary Report | Deep Report |
|------|:--------------:|:-----------:|
| Anonymous | ✅ | ❌ |
| Authenticated | ✅ | ❌ |
| Verified Repository Owner | ✅ | ✅ |
| Enterprise Integration | ✅ | ✅ |

---

# 9. Non-Functional Requirements

## Performance

- Support concurrent repository analysis.
- Support horizontal scaling through Kubernetes.

---

## Scalability

- Stateless services.
- Kubernetes-native deployment.
- Independent service scaling.

---

## Security

- No persistent repository storage.
- Ephemeral execution.
- OAuth authentication.
- Repository ownership verification.

---

## Privacy

Customer source code must never persist after analysis.

---

## Reliability

The platform should recover automatically from service failures.

---

## Maintainability

The platform shall use a plugin-style scanner interface for maintainability. This project implements a fixed set of three scanners (Semgrep, Gitleaks, Trivy) against that interface, not a dynamically loadable plugin system.

---

## Extensibility

New repository providers, AI providers, and scanners should be added without modifying core business logic.

---

## Observability

The platform shall expose:

- Metrics
- Logs
- Health endpoints
- Traces

---

# 10. Assumptions

- Git provider APIs are available.
- Kubernetes is available.
- AI providers are available.
- Network connectivity exists between all components.

---

# 11. Constraints

- Source code cannot be permanently stored.
- Repository ownership must be externally verified.
- Deep Reports require successful authorization.
- The platform must remain cloud agnostic.

---

# 12. Success Criteria

The platform is considered successful when:

- Repository analysis completes successfully.
- Unified reports are generated.
- Temporary repository data is deleted.
- Unauthorized users cannot access Deep Reports.
- Organizations can deploy the platform on-premises or in cloud environments.

---

# 13. MVP Scope

Delivered in phases per `12-roadmap.md` (Phase 1 → Phase 2), not all at once.

## MVP Phase 1 (local vertical slice, no Kubernetes)

- GitHub OAuth authentication — used for both sign-in and Deep Report ownership verification directly (not the full generic multi-provider OIDC abstraction, which is Phase 3)
- GitHub support
- Semgrep, Gitleaks, Trivy
- AI-assisted reasoning (single local provider)
- Summary Reports
- Deep Reports
- REST API
- Dashboard — **owner not yet assigned (TBD).** Sahar is willing to take it (AI-assisted) but Aviel may also want it; the team has not decided.

## MVP Phase 2 (adds Kubernetes)

- Kubernetes Jobs (one per review), replacing the Phase 1 local worker

---

# 14. Out of Scope (MVP)

The following capabilities are intentionally excluded from the initial release.

- AI Router
- Multi-agent AI
- GitLab
- Azure DevOps
- Bitbucket
- Infrastructure-as-Code Analysis
- Kubernetes Manifest Analysis
- Terraform Analysis
- License Compliance
- SBOM Generation
- IDE Extensions
- CI/CD Native Plugins

---

# 15. Risks

| Risk | Mitigation |
|------|------------|
| Repository ownership spoofing | OAuth + provider permission verification |
| AI hallucinations | AI only augments deterministic findings |
| Sensitive source code exposure | Ephemeral repository processing |
| Scanner inconsistency | Internal finding normalization |
| Vendor lock-in | Provider abstraction layer |

---

# 16. Business Value

AI Scan Repos provides measurable value by:

- Reducing repository review time.
- Improving remediation quality.
- Consolidating multiple security tools.
- Preserving source-code privacy.
- Supporting enterprise deployment models.
- Enabling future extensibility through a modular architecture.

---

# 17. References

- Executive Summary
- High-Level Design (HLD)
- Architecture Decision Records (ADRs)
- Security & Authorization Design
