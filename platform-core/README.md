# Platform Core Components

**Managed by:** ArgoCD Application `platform-core`  
**Target Namespace:** `odp-platform`  
**Sync Policy:** Automated (prune: true, selfHeal: true)

## Overview

This directory contains Helm charts for core platform components that power the Open Data Platform (ODP). All components deploy automatically via ArgoCD when changes are committed to the `main` branch.

## Components

### Epic 3: Kubernetes-Native Batch Processing
- **spark-operator/** - Kubeflow Spark Operator (v2.3.0) for running Spark jobs on Kubernetes
- **volcano/** - Volcano scheduler for gang scheduling and resource management  

### Epic 4: Workflow Orchestration  
- **airflow/** - Apache Airflow with KubernetesExecutor for pipeline orchestration

## Directory Structure

Each component follows this structure:

```
platform-core/
├── <component-name>/
│   ├── Chart.yaml              # Helm chart metadata
│   ├── values-base.yaml        # Common default values
│   ├── values-dev.yaml         # Development environment overrides
│   ├── values-staging.yaml     # Staging environment overrides
│   ├── values-prod.yaml        # Production environment overrides
│   └── templates/              # Kubernetes manifests
│       ├── deployment.yaml
│       ├── service.yaml
│       └── ...
```

## Deployment Workflow

1. **Develop:** Create/modify Helm chart in feature branch
2. **Pull Request:** Open PR with changes, CI validates Helm charts
3. **Approve:** Require 1+ approval per branch protection rules
4. **Merge:** PR merged to `main` branch
5. **Auto-Sync:** ArgoCD detects change within 3 minutes
6. **Deploy:** ArgoCD syncs resources to `odp-platform` namespace
7. **Verify:** Check ArgoCD UI for deployment status

## ArgoCD Application Configuration

This directory is managed by the ArgoCD Application defined in:  
`bootstrap/platform-core-app.yaml`

**Key Settings:**
- **Auto-Sync:** Enabled (changes deploy automatically)
- **Prune:** Enabled (resources deleted from Git are removed from cluster)
- **SelfHeal:** Enabled (manual kubectl changes are reverted)
- **CreateNamespace:** Enabled (`odp-platform` namespace created if missing)

## Testing Changes Locally

Before committing, test Helm chart rendering:

```bash
# Validate Helm chart syntax
helm lint platform-core/spark-operator/

# Render templates with base values
helm template spark-operator platform-core/spark-operator/ \
  -f platform-core/spark-operator/values-base.yaml

# Render templates with environment-specific values  
helm template spark-operator platform-core/spark-operator/ \
  -f platform-core/spark-operator/values-base.yaml \
  -f platform-core/spark-operator/values-prod.yaml

# Dry-run apply to cluster
helm template spark-operator platform-core/spark-operator/ \
  -f platform-core/spark-operator/values-base.yaml | kubectl apply --dry-run=client -f -
```

## Rollback

To rollback a failed deployment:

```bash
# Identify problematic commit
git log --oneline platform-core/

# Revert the commit
git revert <commit-hash>

# Push revert
git push origin main

# ArgoCD will auto-sync the revert within 3 minutes
```

## Status

Current components:
- ⏳ **spark-operator** - Pending (Epic 3, Story 3.1)
- ⏳ **volcano** - Pending (Epic 3, Story 3.2)
- ⏳ **airflow** - Pending (Epic 4, Story 4.1)

## References

- **Tech Spec:** `docs/sprint-artifacts/tech-spec-epic-1.md`
- **Application Manifest:** `bootstrap/platform-core-app.yaml`
- **ArgoCD Docs:** https://argo-cd.readthedocs.io/
