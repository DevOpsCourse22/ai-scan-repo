# 15. Risks and Trade-offs

**Document Version:** 1.1

## 15.1 Managing Risk

Each important risk has an owner, action, test, and review date. If the team accepts a remaining risk, it records that decision.

## 15.2 Principal Risks

| ID | Risk | Likelihood | Impact | Architectural treatment | Residual concern |
|---|---|:---:|:---:|---|---|
| R-01 | Malicious repository exploits Semgrep or the scanner code | Medium | Critical | Patched image, non-root Pod, no Kubernetes token, no inbound traffic, resource and network limits | A scanner vulnerability is still possible |
| R-02 | Source, secret, or model token reaches logs or reports | Medium | Critical | Redaction tests, bounded logs, no raw Semgrep output, no request bodies in traces | Redaction may miss an unknown secret format |
| R-03 | Incorrect authorization exposes a Deep Report | Low | Critical | Provider verification, server-side resource checks, fail-closed behavior, denial tests, audit | Provider identity mapping errors require monitoring |
| R-04 | Large repositories exhaust capacity | High | High | Size/file limits, quotas, deadlines, queue fairness, autoscaling | Strict limits may reject legitimate monorepos |
| R-05 | Repository provider outage blocks reviews | Medium | Medium | Timeouts, retryable state, clear failure result | Ownership checks must not succeed during an outage |
| R-06 | Model returns an incorrect explanation | High | Medium | Keep Semgrep result unchanged and label model text separately | A user must still review the advice |
| R-07 | Cleanup fails and source persists temporarily | Low | High | Explicit cleanup, ephemeral volume, Pod deletion, automatic Job removal, stale-resource alert | Node/storage failure can delay physical deletion |
| R-08 | Semgrep image is compromised | Low | Critical | Pinned digest, image scan, signature check, admission policy | A trusted upstream may still be compromised |
| R-09 | One tenant affects another tenant's data or capacity | Medium | High | Authorization checks, quotas, network policy, separate scanner namespace, tenant-scoped storage | Some tenants may require a separate cluster |
| R-10 | Semgrep false positives reduce trust | High | Medium | Record rule ID and version, support reviewed suppression, track false-positive feedback | Rules require tuning |
| R-11 | Control plane cannot create or watch Jobs in the scanner namespace | Medium | High | Cross-namespace RBAC tests, alerts, reconciliation, retryable state | Reviews wait until access or the Kubernetes API recovers |
| R-12 | Pod name or duration statistics are incomplete | Medium | Low | Watch Pods by run-name label and store every observed Pod with Kubernetes timestamps | A cluster outage can delay observation |
| R-13 | AI enrichment capacity is bound to a single non-redundant local GPU | High | Medium | Documented as an accepted POC constraint; AI enrichment stays optional and asynchronous and never blocks the deterministic report | Production use would require hosted or multi-GPU capacity for real concurrency |

## 15.3 Architectural Trade-offs

### Full Scans versus Incremental Scans

A full repository scan covers more code and can be repeated against the same commit, but it takes longer. The initial review uses a full Semgrep scan. A future changed-files scan must not replace scheduled full scans without an explicit decision.

### Kubernetes Jobs versus a Persistent Worker Pool

One Job per run gives each scan its own Pod, limits, labels, timestamps, and temporary storage. Starting a Pod adds delay and creates more Kubernetes objects. A permanent worker pool would start faster but would reuse processes and storage between reviews, so it is not used.

### One Backend Service versus Several Services

The backend runs as one service to avoid extra network calls and deployment steps. A module becomes a separate service only when load, security, or independent release needs show a clear benefit.

### Model Access

The model connects to the backend API with a short-lived token. It does not connect to Kubernetes, the scanner, the database, or object storage. Changing the model does not change the scanner Job or its permissions.

### Public Summaries versus Privacy

A public summary can reveal security information even without source code. Publish it only after confirming that the repository is public and the applicable policy allows publication. Private repository summaries are never public.

### Immediate Deletion versus Auditability

Delete the checked-out source when the scanner ends. Keep findings, reports, run records, and audit events only for their defined retention periods. Audit requirements do not justify keeping repository source.

### Local AI on a Single GPU versus Hosted, Scaled AI

Running AI enrichment locally on one team member's GPU keeps source code off third-party infrastructure and costs nothing, which fits a course POC well. It also means AI enrichment cannot run concurrently across multiple reviews — a real production deployment would use hosted or multi-GPU capacity instead. The team accepts the single-GPU limitation for this project and documents it explicitly (see [Scope Decisions Log](../SCOPE-DECISIONS.md)) rather than pretending the platform is horizontally scaled for AI the way it is for deterministic scanning.

## 15.4 Known Limitations

- Semgrep cannot identify every vulnerability or prove that code is safe.
- The initial platform does not safely execute builds or dynamic tests.
- NetworkPolicy alone cannot provide portable hostname-level egress control.
- Ownership verification depends on provider availability and correct application installation.
- Repository size and scan deadlines may exclude very large monorepos until partitioning is designed.
- Model explanations may be incomplete, outdated, or wrong.
- Different cloud environments may behave differently and have different costs.
- AI enrichment runs on a single local GPU and cannot process multiple reviews concurrently; this is acceptable for a POC and is not a production capacity model.

## 15.5 Decision and Review Process

Review risks after incidents and before adding a repository provider, executing repository code, changing tenant isolation, or changing data retention. Record decisions that change security or data handling and update the related tests.

Next: [Future Architecture](16-future-architecture.md).
