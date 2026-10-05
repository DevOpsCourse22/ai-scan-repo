# AI Scan Repos Documentation

This directory is the source of truth for product requirements, architecture, security, implementation design, operations, and roadmap decisions. Generated Word or PDF documents are artifacts and must not replace these Markdown sources.

## Core Documents

| No. | Document | Purpose | Status |
|---:|---|---|---|
| 01 | [Executive Summary](01-executive-summary.md) | Vision, problem, solution, report model, and business value | Draft |
| 02 | [Business Requirements](02-business-requirements.md) | Objectives, stakeholders, BR/FR requirements, and authorization model | Draft |
| 03 | [High-Level Design](03-high-level-design/README.md) | System architecture, trust boundaries, components, and deployment model | Draft |
| 04 | [Low-Level Design](04-low-level-design.md) | Implementable services, interfaces, state, worker, and failure design | Draft |
| 05 | [Security and Authorization Design](05-security-authorization-design.md) | Identity, repository authorization, data protection, and security controls | Draft |
| 06 | [Threat Model](06-threat-model.md) | Assets, adversaries, trust boundaries, threats, and validation | Draft |
| 07 | [API Specification](07-api-specification.md) | v1 resources, schemas, errors, idempotency, and API security | Draft |
| 08 | [Database Design](08-database-design.md) | PostgreSQL entities, tenant isolation, transactions, retention, and recovery | Draft |
| 09 | [Kubernetes Design](09-kubernetes-design.md) | Jobs, RBAC, NetworkPolicy, scheduling, reconciliation, and cleanup | Draft |
| 10 | [Deployment Guide](10-deployment-guide.md) | Prerequisites, configuration, deployment, verification, upgrade, and recovery | Draft |
| 11 | [Monitoring and Observability](11-monitoring-observability.md) | Metrics, logs, traces, audit, SLOs, alerts, and runbooks | Draft |
| 12 | [Roadmap](12-roadmap.md) | Delivery phases, exit criteria, workstreams, and success measures | Draft |
| 13 | [Future Enhancements](13-future-enhancements.md) | Candidate providers, scanners, AI, governance, and scale capabilities | Candidate backlog |
| 14 | [MCP Adaptation Plan](14-mcp-adaptation-plan.md) | How the existing `code-review-mcp` is split into platform components and the `mcp-bridge` | Proposed |
| 15 | [MCP Server Documentation](15-mcp-server-documentation.md) | Full disclosure of the current MCP server: architecture, all tools, settings, report schema, disaster recovery | Current |
| 16 | [Gemma 4 26B User Guide](16-gemma-user-guide.md) | Step-by-step guide to driving the MCP from LM Studio with Gemma 4 26B | Current |
| 17 | [Review API (Flask)](17-review-api.md) | HTTP control plane over the review pipeline: endpoints, prompt consolidation, private-repo tokens, restart/resume, cloud + on-prem providers, configuration, demo console | Current |
| 18 | [Team Setup Guide](18-team-setup-guide.md) | Running the project on any machine: install paths, choosing a cloud or on-prem LLM, first review, private repos, testing without an LLM, troubleshooting | Current |
| 19 | [Secret Exposure — Limits Analysis](19-secret-exposure-limits.md) | Why a credential committed inside a reviewed repository cannot be guaranteed to stay out of reports, logs and provider requests; what the implementation does today; achievable controls. **Bilingual: English + עברית** | Current |
| 20 | [Phase 3 Plan](20-phase-3-plan.md) | Platform build-out to the 14 October submission: K3s, CI/CD per repo, the six-repo split, Jenkins, secrets, and what to cut first. |
| 21 | [Writing the Kubernetes manifests](21-kubernetes-manifests-guide.md) | A guided exercise, not an answer key: what review-api and review-console need from a cluster, the traps met while verifying them, and how to prove each step. |

## Repository Layout

The implementation is split into three components, over one shared review
engine. Each has its own README:

| Folder | Component | Reference |
|---|---|---|
| `mcp-server/` | MCP server **and** the shared review pipeline | [15](15-mcp-server-documentation.md), [16](16-gemma-user-guide.md) |
| `backend/` | Flask HTTP API | [17](17-review-api.md) |
| `frontend/` | Demo console (single HTML file, no build) | [17 §10](17-review-api.md) |

Start with [18 — Team Setup Guide](18-team-setup-guide.md) to get it running.

## High-Level Design Chapters

The HLD is divided into focused chapters. Start with the [HLD index](03-high-level-design/README.md), which links to all 16 sections from introduction through future architecture.

## Architecture Decision Records

Durable architecture decisions live under [`adr/`](adr/). An ADR records context, decision, alternatives, consequences, and status. Accepted ADRs take precedence when a summary document has not yet been updated.

Planned decisions include:

- Ephemeral repository processing.
- Kubernetes Jobs for review execution.
- Full-repository deterministic scanning.
- AI augmentation rather than replacement.
- Summary versus Deep Report separation.
- Repository-provider ownership verification.
- Plugin/provider architecture.
- Future policy-driven AI routing.

## Diagrams

Editable and rendered architecture diagrams live under [`diagrams/`](diagrams/). Editable Draw.io sources are canonical; PNG files are generated views for documents and presentations.

## Documentation Conventions

- Use Markdown as the editable source.
- Use numbered top-level documents and HLD chapters for stable navigation.
- Use relative links so documentation works offline and in Git hosting.
- Store architecture diagrams in `docs/diagrams/`.
- Store architecture decisions in `docs/adr/`.
- Do not include credentials, repository source, customer identifiers, or detected secret values.
- Mark assumptions, risks, open decisions, document status, and version explicitly.
- Update related requirements, designs, threat model, ADRs, and tests together after material changes.

## Recommended Reading Order

1. Executive Summary.
2. Business Requirements.
3. High-Level Design.
4. Security Design and Threat Model.
5. Low-Level Design, API, and Database Design.
6. Kubernetes Design and Deployment Guide.
7. Monitoring, Roadmap, and Future Enhancements.

## Current Architecture Invariants

- Customer source code is processed only in an ephemeral review workspace.
- Deep Reports require repository-provider authorization.
- Deterministic scanners remain authoritative.
- AI is optional and advisory.
- Repository, scanner, AI, and infrastructure integrations remain replaceable.
- Kubernetes scanner workloads are isolated, bounded, and receive no ambient cluster authority.
