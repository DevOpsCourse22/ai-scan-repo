# 14. Technology Stack

## 14.1 Selection Principles

Technology choices favor open standards, operational maturity, security support, cloud portability, replaceable interfaces, and team maintainability. Product business logic must not depend directly on a specific cloud service or AI vendor.

Version numbers and exact managed-service products belong in deployment baselines and dependency lock files; this HLD identifies architectural technologies and selection rationale.

## 14.2 Application Stack

| Layer | Preferred technology | Rationale |
|---|---|---|
| Backend API | Python and FastAPI | Typed API contracts, asynchronous integrations, strong ecosystem |
| Validation | Pydantic | Explicit request, configuration, and internal schema validation |
| Persistence | SQLAlchemy and PostgreSQL | Portable transactional model, mature migration and indexing support |
| Database migration | Alembic | Versioned, reviewable schema evolution |
| Frontend | React with Vite | Component ecosystem, fast builds, static deployment model |
| API description | OpenAPI | Standard client generation and contract testing |
| Authentication | OpenID Connect and OAuth 2.0 | Standards-based identity and delegated provider access |

The browser communicates with the backend through a versioned REST API. Backend modules are initially deployed together as a modular service; extraction into separate network services requires measured scaling, ownership, or isolation need.

## 14.3 Analysis Stack

| Capability | Initial technology | Architectural role |
|---|---|---|
| SAST | Semgrep | Detect insecure code patterns using policy-controlled rules |
| Secret detection | Gitleaks | Detect committed credentials and secret patterns with redacted evidence |
| Dependency/IaC/container analysis | Trivy | Analyze supported manifests, dependencies, misconfiguration, and artifacts |
| Code quality | Pluggable quality adapter | Produce maintainability and quality findings |
| AI enrichment | Provider adapters | Connect approved Ollama, LM Studio, or hosted AI endpoints |

Tools run from pinned, signed scanner images. Adapter contracts prevent tool-specific output and exit behavior from leaking into core orchestration.

## 14.4 Platform Stack

| Concern | Technology | Notes |
|---|---|---|
| Container runtime | OCI-compatible containers | Minimal, non-root, signed images |
| Scheduling | Kubernetes | Deployments for services; Jobs for repository reviews |
| Packaging | Helm | Environment-configurable application release |
| Infrastructure | Terraform | Reviewable cloud/on-premises infrastructure modules |
| Ingress | Kubernetes Ingress or Gateway API implementation | TLS, routing, edge policy |
| Secrets | External secret manager with workload identity | Avoid static secrets and committed values |
| Report storage | S3-compatible object storage | Encryption, lifecycle, integrity, portable interface |
| Coordination | PostgreSQL-backed queue initially; Redis if justified | Avoid adding infrastructure before a measured need |

## 14.5 Observability Stack

OpenTelemetry provides vendor-neutral instrumentation. Prometheus-compatible metrics support alerting; Grafana-compatible dashboards visualize service and review health. Structured logs can be collected by Loki or an enterprise log platform. Trace storage is selected by the deployment environment behind OpenTelemetry exporters.

Audit storage has stronger integrity and access requirements than general logs and may use a dedicated database table and immutable archival destination.

## 14.6 CI/CD and Supply Chain

GitHub Actions is the preferred source pipeline, with equivalent Jenkins support where organizational requirements demand it. CI performs formatting, unit and contract tests, dependency and secret scanning, SAST, IaC policy validation, container build, SBOM creation, vulnerability scanning, signing, and provenance generation.

Argo CD is the preferred GitOps deployment controller. Promotion changes signed image digests in an environment repository after approval. CI does not hold unrestricted production cluster credentials.

## 14.7 Portability Rules

- Cloud SDKs remain inside infrastructure or provider adapters.
- Storage uses a platform interface supporting S3-compatible semantics.
- Identity uses OIDC claims mapped to internal subjects and roles.
- AI requests use the common AI provider contract.
- Repository behavior uses canonical provider adapters.
- Telemetry uses OpenTelemetry and standard metric formats.
- Kubernetes manifests avoid provider-only fields in the base chart; overlays add environment integrations.

## 14.8 Technology Governance

New technologies require an owner, security review, lifecycle status, upgrade path, observability plan, backup/restore consideration, and ADR when they introduce a durable architectural commitment. Dependencies without active support or a clear license are not admitted.

Next: [Risks and Trade-offs](15-risks-tradeoffs.md).
