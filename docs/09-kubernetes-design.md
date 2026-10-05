# Kubernetes Design

**Project:** AI Scan Repos  
**Document Version:** 1.0  
**Status:** Draft

---

## 1. Purpose

This document defines the Kubernetes resources, workload isolation, RBAC, networking, scheduling, scaling, reconciliation, and cleanup model for AI Scan Repos. The central design decision is one isolated Kubernetes Job per repository review.

## 2. Cluster and Namespace Model

Long-running control-plane services and ephemeral scanners run in a dedicated application namespace enforcing the restricted Pod Security profile. Production environments may split them into `control-plane` and `scanner` namespaces to provide stronger quotas, policies, node placement, and operational ownership.

Managed PostgreSQL, object storage, identity, and secret services remain outside the scanner namespace. Scanner Jobs must never have direct database access.

## 3. Workloads

| Component | Resource | Replicas/lifecycle |
|---|---|---|
| Frontend | Deployment, Service | Two or more production replicas |
| Backend API | Deployment, Service | Horizontally scaled, stateless |
| Review reconciler | Deployment | Leader-elected or safely partitioned |
| Repository scanner | Job | One per review, no inbound Service |
| Metrics/log collectors | Platform-managed agents | Cluster topology dependent |

Long-running services use readiness/startup/liveness probes with different purposes. Scanner Jobs use phase reporting, a maximum active deadline, and terminal status rather than liveness restarts.

## 4. Scanner Job Template

Job fields are owned by the platform. User input may select only policy-approved review options and cannot provide image, command, namespace, environment variables, volumes, service account, or node placement.

Required settings include:

```yaml
spec:
  backoffLimit: 0
  activeDeadlineSeconds: 1800
  ttlSecondsAfterFinished: 600
  template:
    spec:
      restartPolicy: Never
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: scanner
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          resources:
            requests: {cpu: "500m", memory: "512Mi"}
            limits: {cpu: "2", memory: "2Gi"}
```

Production images are pinned by digest. The workspace and `/tmp` are separate, size-limited `emptyDir` volumes. No source persistent volume is permitted.

## 5. Service Accounts and RBAC

The API/reconciler service account receives a namespaced Role allowing only required Job create/get/list/watch/delete and bounded Pod status/log reads. It cannot create arbitrary RBAC, privileged Pods, persistent volumes, or cluster-scoped resources.

The scanner and frontend service accounts have token automount disabled and no Kubernetes permissions. Deployment automation uses a separate identity. CI build identity does not receive runtime or cluster administration access.

RBAC is verified with automated `kubectl auth can-i` or equivalent policy tests for allowed and denied operations.

## 6. Network Policy

The namespace starts with default-deny ingress and egress. Explicit paths are:

- Ingress controller to frontend and API.
- API to identity provider, PostgreSQL, object storage, queue, Kubernetes API, DNS, and telemetry.
- Scanner to DNS, configured source provider, approved vulnerability databases, report upload, optional AI provider, and telemetry.

Scanner-to-scanner, scanner-to-database, public-to-scanner, and unrestricted internet access remain denied. Domain-aware controls require an egress proxy/gateway or CNI feature because base NetworkPolicy does not provide portable hostname rules.

## 7. Scheduling and Isolation

Scanner workloads should use a dedicated autoscaled node group with taints and tolerations. Node affinity keeps scanners away from database, ingress, and platform-management nodes. Higher-risk environments should use sandboxed runtimes, dedicated nodes per trust class, or separate clusters.

Long-running services use topology spread and anti-affinity across zones. Priority classes must prevent scanner bursts from evicting essential control-plane or observability workloads.

## 8. Resource Governance

ResourceQuota limits aggregate CPU, memory, ephemeral storage, Pod count, and Job count. LimitRange provides safe defaults and maxima. Application-level quotas add tenant/user concurrency, daily review, repository-size, and scan-profile controls.

The scheduler admits work only when capacity policy allows it. Pending queue age is observable. Cluster autoscaling has explicit minimum/maximum bounds to prevent both starvation and unbounded cost.

## 9. Configuration and Secrets

ConfigMaps contain validated non-secret environment configuration. Helm values define immutable release settings and policy references. Secrets come from an external secret manager through workload identity or a controlled secret synchronization mechanism.

Clone grants are short-lived and review-scoped. They must not appear in Job labels, annotations, arguments, logs, or status. Prefer projected in-memory or short-lived file mounts with restrictive permissions.

## 10. Reconciliation

The database is authoritative for product review state; Kubernetes is authoritative for workload observation. The reconciler handles:

- Queued reviews without Jobs.
- Jobs whose creation outcome was uncertain.
- Running reviews with missing/failed Pods.
- Successful Jobs without finalized reports.
- Completed reviews with leftover Jobs.
- Orphaned Jobs without durable reviews.
- Cancellation, timeout, and stale termination.

Reconciliation is idempotent and uses review state versions. Job success is insufficient unless the report digest and cleanup outcome are recorded.

## 11. Availability and Updates

API and frontend Deployments use rolling updates, readiness gates, disruption budgets, and topology spread. Database migrations are backward compatible. Helm deployments use atomic rollback and immutable image promotion.

Scanner Jobs continue on the image digest with which they started. New releases do not mutate active Jobs. Emergency image revocation can cancel affected Jobs and require a new review.

## 12. Admission and Runtime Policy

Admission policy enforces approved registries, signed digests, non-root execution, no privilege escalation, dropped capabilities, resource limits, approved volume types, and forbidden host access. Runtime monitoring detects unexpected processes, network destinations, writable paths, and privilege activity.

Runtime detection supplements rather than replaces preventive controls.

## 13. Cleanup

Cleanup uses four layers:

1. Worker `finally` cleanup removes checkout and temporary artifacts.
2. Pod termination destroys Job-local ephemeral volumes.
3. Job TTL removes completed Kubernetes objects.
4. A stale-resource controller detects and alerts on leaked Jobs or volumes.

Cleanup is tested under process crash, Pod eviction, node loss, timeout, cancellation, report-upload failure, and control-plane outage.

## 14. Validation Checklist

- Manifests pass schema, policy, and security-context validation.
- RBAC allows only documented operations.
- Network reachability matches the allowlist.
- Jobs cannot reach metadata services, database, or other Pods.
- Quota and deadlines terminate abusive workloads.
- Unsigned or privileged images are rejected.
- Source volumes disappear after every terminal path.
