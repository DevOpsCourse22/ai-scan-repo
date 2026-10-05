# Code Review: https://github.com/Aviel-Amitay/kubernetes-exercises

- **Mode:** brief  
- **Model:** google/gemma-4-e4b  
- **Commit:** `ac0fec4e77a8`  
- **Generated:** 2026-08-20T15:47:11Z  
- **Files reviewed:** 34 (failed: 0, skipped: 0)

> ⚠️ **3 file(s) were not reviewed** because their type is not in the review set: `.md` ×3. Pass `extensions` to include them.

## Verdict: 5/10

This repository provides a comprehensive set of Kubernetes manifests and associated microservices for deploying multi-tier applications. While the overall architecture is sound for demonstrating K8s concepts, the codebase lacks critical operational best practices required for production use. Major deficiencies include missing resource requests/limits, reliance on hardcoded secrets, and general inconsistencies in deployment patterns (e.g., using Pods instead of Deployments).

## Key issues

- 🟠 **high** `class-porject-1/k8s/02-storage/pvc/05-mongo-pvc.yaml` (line 10) — PVC requests (2Gi) should be consistent with the actual storage needs of the service.
- 🟠 **high** `class-porject-1/k8s/01-initial-config/02-secrets.yaml` (line 9) — Secrets contain hardcoded credentials; use a dedicated secret management solution (e.g., Vault) in production.
- 🟠 **high** `class-porject-1/k8s/03-deployment/01-db/07-mongo-deployment.yaml` — Missing resource requests and limits for container stability.
- 🟠 **high** `class-porject-1/k8s/03-deployment/02-redis/14-redis-daemonset.yaml` — Missing resource requests and limits for container stability.
- 🟡 **medium** `1-pods-labs/lab2/webapp-pod.yaml` (line 18) — The environment variable name 'producation' is misspelled and should be corrected for maintainability.
- 🟡 **medium** `class-porject-1/apps/backend/server.js` (line 10) — The fallback MONGO_URI uses 'localhost', which is incorrect for a Kubernetes cluster environment.
- 🟡 **medium** `class-porject-1/k8s/03-deployment/03-backend-app/09-backend-deployment.yaml` — Missing resource requests and limits for container stability.
- 🟡 **medium** `class-porject-1/k8s/16-job-backup.yaml` (line 20) — Backup script uses hardcoded paths and lacks error handling for file operations.
- 🔵 **low** `2-ReplicaSets/lab1/mongo-rs.yml` (line 9) — Using ReplicaSet with replicas: 1 is redundant; consider using a single Pod definition if high availability isn't required.
- 🔵 **low** `class-porject-1/k8s/17-cronjob-backup.yaml` — CronJob definition should use a Job template for better resource isolation.

## Files

### `1-pods-labs/lab2/webapp-pod.yaml` — 7/10
Consider using Deployments instead of raw Pods for better lifecycle management.
- 🟡 The environment variable name 'producation' is misspelled and should be corrected for maintainability. (line 18)

### `2-ReplicaSets/lab1/mongo-rs.yml` — 8/10
For a single instance database setup, consider using StatefulSet for stable network identity.
- 🔵 Using ReplicaSet with replicas: 1 is redundant; consider using a single Pod definition if high availability isn't required. (line 9)

### `class-porject-1/apps/backend/server.js` — 7/10
The fallback URI in server.js should reference service names, not 'localhost'.
- 🟡 The fallback MONGO_URI uses 'localhost', which is incorrect for a Kubernetes cluster environment. (line 10)

### `class-porject-1/k8s/01-initial-config/02-secrets.yaml` — 3/10
Hardcoded secrets are a major security risk; use proper secret injection mechanisms.
- 🟠 Secrets contain hardcoded credentials; use a dedicated secret management solution (e.g., Vault) in production. (line 9)

### `class-porject-1/k8s/02-storage/pvc/05-mongo-pvc.yaml` — —
- 🟠 PVC requests (2Gi) should be consistent with the actual storage needs of the service. (line 10)

### `class-porject-1/k8s/03-deployment/01-db/07-mongo-deployment.yaml` — 7/10
Add resource limits and requests to the container spec.
- 🟠 Missing resource requests and limits for container stability.

### `class-porject-1/k8s/03-deployment/02-redis/14-redis-daemonset.yaml` — 7/10
Add resource limits and requests to the container spec.
- 🟠 Missing resource requests and limits for container stability.

### `class-porject-1/k8s/03-deployment/03-backend-app/09-backend-deployment.yaml` — 7/10
Add resource limits and requests to the container spec.
- 🟡 Missing resource requests and limits for container stability.

### `class-porject-1/k8s/16-job-backup.yaml` — 8/10
The backup script is functional but brittle; consider using dedicated tools or better volume management.
- 🟡 Backup script uses hardcoded paths and lacks error handling for file operations. (line 20)

### `class-porject-1/k8s/17-cronjob-backup.yaml` — 8/10
Ensure the CronJob's jobTemplate is robust and handles potential failures gracefully.
- 🔵 CronJob definition should use a Job template for better resource isolation.

*24 further file(s) were reviewed and found unremarkable (or trivial).*

*Reviewed in 2 batch(es) covering 34 files.*
