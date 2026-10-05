# Deployment Guide

**Project:** AI Scan Repos  
**Document Version:** 1.0  
**Status:** Draft

---

## 1. Purpose

This guide describes how to prepare, configure, deploy, verify, upgrade, roll back, and recover AI Scan Repos. The examples are environment-neutral; production values and secrets must come from approved infrastructure and secret-management processes.

## 2. Prerequisites

- Kubernetes cluster with supported Jobs, NetworkPolicy, restricted Pod Security, quotas, and autoscaling.
- Helm 3 and a compatible ingress or Gateway implementation.
- PostgreSQL with encrypted backup and point-in-time recovery.
- Encrypted S3-compatible report storage.
- External identity provider using OIDC.
- GitHub application for initial repository authorization and cloning.
- External secret manager and workload identity integration.
- Metrics, logs, traces, alerting, and audit destinations.
- Container registry supporting immutable digests and signed artifacts.
- DNS and TLS certificate management.

## 3. Environment Preparation

Create isolated development, staging, and production environments. Production should use private data services, controlled scanner egress, dedicated scanner capacity, and separate deployment credentials.

Provision infrastructure with reviewed Terraform plans. Remote state must be encrypted, locked, access-controlled, and free of plaintext secrets. Record database endpoints, object-store buckets, workload identities, cluster context, DNS names, and encryption-key references as outputs with appropriate sensitivity.

## 4. Provider Configuration

### 4.1 OIDC

Register the production callback/logout URLs. Configure exact issuer and audience, PKCE, supported claims, role mapping, session duration, and privileged authentication requirements. Store client secrets in the secret manager when a confidential client is used.

### 4.2 Repository Provider

Install the GitHub application with the minimum repository metadata/content read permissions required. Configure canonical API and clone endpoints, webhook verification secret, application identity, and token issuance. Never use a broad personal access token as the production integration.

### 4.3 AI Provider

Leave AI disabled unless data classification, region, retention, no-training terms, model, endpoint, and policy are approved. Configure the provider through secret references and an allowlisted egress destination.

## 5. Build and Artifact Promotion

CI must:

1. Run formatting, unit, contract, integration, and security tests.
2. Scan source, dependencies, secrets, IaC, and container images.
3. Build minimal backend, scanner, and frontend images.
4. Generate SBOM and provenance.
5. Sign immutable image digests.
6. Publish to the approved registry.
7. Promote digests through reviewed environment configuration.

Do not rebuild different binaries for staging and production. Promote the verified digest.

## 6. Configuration

Copy the environment values template and set:

- Image digests and pull policy.
- Hostname, TLS secret/reference, and trusted proxy settings.
- OIDC issuer, audience, and callback configuration.
- Repository provider application configuration.
- PostgreSQL and object-store secret references.
- Scanner resource profiles, deadline, repository limits, and concurrency quotas.
- Allowed egress destinations and AI policy.
- Retention periods, telemetry endpoints, and alert receivers.

No secret value belongs in a Helm values file or command history.

## 7. Pre-Deployment Validation

Run:

```bash
helm lint helm/ai-scan-repos
helm template ai-scan-repos helm/ai-scan-repos \
  --namespace ai-scan-repos \
  --values values-production.yaml > rendered.yaml
```

Validate rendered resources with Kubernetes schemas, organization policy, secret scanning, image-signature policy, RBAC review, and NetworkPolicy tests. Review the database migration plan and confirm backup/restore readiness.

## 8. Deployment Sequence

1. Provision or verify infrastructure dependencies.
2. Create namespace labels, quotas, limit ranges, and default-deny policies.
3. Configure workload identities and external secret references.
4. Apply backward-compatible database migrations.
5. Deploy backend and reconciler.
6. Deploy frontend and ingress.
7. Enable scanner Job admission after control-plane readiness.
8. Configure ServiceMonitors, dashboards, alerts, and audit export.
9. Run post-deployment verification before opening user traffic.

Example release command:

```bash
helm upgrade --install ai-scan-repos helm/ai-scan-repos \
  --namespace ai-scan-repos \
  --create-namespace \
  --values values-production.yaml \
  --atomic --wait --timeout 15m
```

## 9. Post-Deployment Verification

Verify:

- Liveness and readiness without exposing dependency secrets.
- OIDC sign-in, sign-out, expiry, and invalid-token denial.
- Cross-user and cross-tenant report denial.
- GitHub repository access verification and revocation.
- One canary public Summary review and one authorized Deep review.
- Scanner coverage, immutable commit SHA, report digest, and AI status.
- Job security context, RBAC, egress restrictions, deadline, and cleanup.
- Metrics, traces, audit events, dashboards, and alert delivery.
- Direct object-store access is denied without authorized short-lived access.

## 10. Upgrade Strategy

Use rolling updates for stateless services and immutable digests for scanner Jobs. Existing Jobs finish with their original image and policy version unless a security revocation requires cancellation.

Database upgrades follow expand/migrate/contract. Report and finding schemas are explicitly versioned so older artifacts remain readable or have a controlled migration path. Perform canary or staging validation before production promotion.

## 11. Rollback

Application rollback uses the prior Helm revision and image digests:

```bash
helm history ai-scan-repos --namespace ai-scan-repos
helm rollback ai-scan-repos <REVISION> --namespace ai-scan-repos --wait
```

Do not automatically reverse a destructive database migration. Prefer forward recovery or a tested migration-specific rollback. Confirm that the prior application version supports the current expanded schema.

After rollback, verify authentication, authorization, review admission, reconciliation, report retrieval, and cleanup.

## 12. Backup and Recovery

Back up PostgreSQL, report objects, policy/configuration repositories, encryption-key references, and audit archives. Repository working copies require no backup.

Recovery procedure:

1. Restore infrastructure and keys under approved access.
2. Restore PostgreSQL to the chosen recovery point.
3. Restore or reconnect report storage.
4. Verify object digests and tenant access.
5. Reconcile reviews that were active at the recovery point.
6. Reissue workload/provider credentials where compromise is possible.
7. Run security and canary verification before reopening traffic.

Recovery point and recovery time objectives must be defined and tested for production.

## 13. Operational Safety

- Use explicit Kubernetes context and environment in deployment commands.
- Require peer approval for production changes.
- Never copy production source, report evidence, or secrets into support tickets.
- Audit privileged access and emergency configuration.
- Pause review admission before high-risk maintenance while allowing safe status/report reads.

## 14. Uninstall and Decommission

An uninstall must not delete databases, report buckets, encryption keys, or audit archives implicitly. Decommissioning follows an approved data-retention plan: stop admission, finish/cancel Jobs, verify cleanup, export required audit evidence, delete report/data assets, revoke provider installations and credentials, remove DNS/TLS, and finally destroy infrastructure.
