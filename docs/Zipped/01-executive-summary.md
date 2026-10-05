# Executive Summary

**Project:** AI Scan Repos

**Document Version:** 1.0

**Status:** Draft

---

# Vision

AI Scan Repos is an AI-powered DevSecOps Repository Analysis Platform designed to improve repository security reviews by combining deterministic analysis tools with AI-assisted reasoning.

Rather than replacing traditional security scanners, the platform orchestrates multiple security and quality analysis engines, enriches their findings with contextual AI reasoning, and generates a unified actionable report.

The platform is designed around five primary principles:

- Security
- Privacy
- Modularity
- Scalability
- Extensibility

Customer source code is processed only inside isolated ephemeral execution environments and is never permanently stored.

---

# Problem Statement

Modern software development relies on numerous independent security and quality analysis tools.

Organizations frequently execute:

- Static Application Security Testing (SAST)
- Secret Detection
- Dependency Vulnerability Scanning
- Container Image Scanning
- Code Quality Analysis

Although each tool provides valuable information, their findings are often isolated, inconsistent and difficult to prioritize.

Security engineers and developers spend considerable effort correlating reports before meaningful remediation work can begin.

---

# Proposed Solution

AI Scan Repos consolidates repository analysis into a single workflow.

The platform performs deterministic analysis across the entire repository using specialized security scanners.

The resulting findings are then enriched using AI-assisted reasoning to provide:

- Risk prioritization
- Root cause analysis
- Remediation recommendations
- Architectural observations
- Executive summaries

AI complements deterministic scanners instead of replacing them.

---

# Report Model

The platform provides two report levels.

## Summary Report

Available for:

- Public repositories
- Anonymous users
- Unverified users

Includes:

- Overall security score
- Vulnerability score
- Code quality score
- High-level findings

No sensitive implementation details are exposed.

---

## Deep Report

Available only after successful repository ownership verification.

Includes:

- File-level findings
- Code snippets
- Detailed explanations
- Remediation guidance
- Risk prioritization

---

# Repository Trust Model

Repository ownership is never inferred from uploaded source code.

Instead, ownership is verified using supported repository providers such as:

- GitHub
- GitLab

Future versions will support additional providers through an adapter-based architecture.

---

# Core Architecture Principles

- Full repository deterministic scanning
- AI as an augmentation layer
- Ephemeral Kubernetes Jobs
- No source-code persistence
- Plugin architecture
- AI provider independence
- Cloud agnostic deployment
- Trust-based authorization

---

# High-Level Workflow

1. User submits a repository URL or Pull Request.
2. Authentication and repository verification are performed (when requesting a Deep Report).
3. The Review Orchestrator creates an ephemeral Kubernetes Job.
4. The repository is cloned into a temporary workspace.
5. Deterministic scanners analyze the entire repository.
6. Relevant findings are sent to the AI Analysis Engine.
7. A unified report is generated.
8. Findings and metadata are stored.
9. The temporary workspace is deleted with the code repo entirely.

---

# Business Value

AI Scan Repos enables organizations to:

- Consolidate repository analysis
- Reduce remediation time
- Improve developer productivity
- Preserve source-code privacy
- Deploy the platform on-premises or in the cloud
- Integrate multiple AI providers without changing the platform architecture

---

# Future Vision

Future releases may include:

- AI Router
- Infrastructure-as-Code analysis
- SBOM generation
- License compliance
- GitHub App
- GitLab App
- Azure DevOps integration
- Enterprise policy engine