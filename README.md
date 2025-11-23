# ODP Platform GitOps Repository

**Project:** Open Data Platform (ODP) - Kubernetes-Native Data Infrastructure
**Owner:** mokhahmed92
**Purpose:** GitOps-based infrastructure management using ArgoCD

## Overview

This repository contains Kubernetes infrastructure-as-code for the Open Data Platform, managed declaratively through ArgoCD GitOps workflows. All platform components deploy automatically from this repository to the Kubernetes cluster.

## Directory Structure

```
odp-platform-gitops/
├── bootstrap/               # ArgoCD self-management resources
│   ├── argocd-values.yaml  # ArgoCD Helm values (HA configuration)
│   └── *-app.yaml          # ArgoCD Application definitions
├── platform-core/           # Core platform components
│   ├── spark-operator/     # Kubeflow Spark Operator (Epic 3)
│   ├── volcano/            # Volcano scheduler (Epic 3)
│   └── airflow/            # Apache Airflow (Epic 4)
├── observability/           # Monitoring and logging
│   ├── splunk-connect/     # Splunk Connect for Kubernetes (Epic 7)
│   └── newrelic/           # New Relic APM (Epic 7)
└── namespaces/              # Multi-tenant namespace definitions
    ├── domain-finance/     # Finance team namespace (Epic 2)
    └── domain-marketing/   # Marketing team namespace (Epic 2)
```

## GitOps Workflow

1. **Change:** Create feature branch, modify infrastructure manifests
2. **Review:** Open pull request, require 1+ approval
3. **CI:** GitHub Actions validates Helm charts and YAML syntax
4. **Merge:** Approved PR merged to `main` branch
5. **Sync:** ArgoCD detects change within 3 minutes
6. **Deploy:** ArgoCD syncs resources to Kubernetes cluster automatically
7. **Verify:** Check ArgoCD UI for deployment status

## Rollback

To rollback a failed deployment:

```bash
# Identify problematic commit
git log --oneline -10

# Revert the commit
git revert <commit-hash>

# Push revert to main
git push origin main

# ArgoCD will detect and sync the revert within 3 minutes
```

##

 ArgoCD Applications

This repository is managed by the following ArgoCD Applications:

- **platform-core**: Manages `platform-core/` directory → `odp-platform` namespace
- **observability**: Manages `observability/` directory → `odp-observability` namespace
- **namespaces**: Manages `namespaces/` directory → cluster-scoped resources

## Getting Started

### Prerequisites

- Kubernetes cluster (v1.28+)
- `kubectl` with admin access
- `helm` (v3.x)
- ArgoCD installed (see Story 1.1 in Epic 1)

### Deploy Infrastructure

#### Option 1: App of Apps Pattern (Recommended)

Deploy all infrastructure Applications with a single command using the App of Apps pattern:

```bash
kubectl apply -f bootstrap/infra-apps.yaml
```

This creates the `infra-apps` Application which automatically manages all child Applications:
- **namespaces** (sync wave 1) - Multi-tenant namespace definitions
- **platform-core** (sync wave 2) - Core platform components
- **observability** (sync wave 3) - Monitoring and logging

Verify deployment:
```bash
kubectl get applications -n argocd
```

#### Option 2: Manual Application Deployment

Deploy Applications individually:

```bash
kubectl apply -f bootstrap/namespaces-app.yaml
kubectl apply -f bootstrap/platform-core-app.yaml
kubectl apply -f bootstrap/observability-app.yaml
```

### Trigger Manual Sync

All deployments happen automatically via ArgoCD. To manually trigger a sync:

```bash
# Sync specific Application
argocd app sync platform-core

# Or via kubectl
kubectl patch application platform-core -n argocd \
  --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"normal"}}}'
```

## Project Documentation

- **Epic Breakdown:** `odp-infra-prj/docs/epics.md`
- **Technical Specifications:** `odp-infra-prj/docs/sprint-artifacts/tech-spec-epic-*.md`
- **Architecture Decisions:** `odp-infra-prj/docs/architecture.md`
- **Product Requirements:** `odp-infra-prj/docs/PRD.md`

## Epic Progress

- [x] Epic 1: GitOps Foundation (ArgoCD, Helm patterns, PR workflows)
- [ ] Epic 2: Multi-Tenant Platform (Namespaces, RBAC, network isolation)
- [ ] Epic 3: Kubernetes-Native Batch Processing (Spark, Volcano)
- [ ] Epic 4: Workflow Orchestration (Airflow pipelines)
- [ ] Epic 5: Storage Architecture (Medallion zones: Bronze/Silver/Gold)
- [ ] Epic 6: Data Transformation (DBT with Spark Connect)
- [ ] Epic 7: Enterprise Observability (Splunk, New Relic)
- [ ] Epic 8: Hadoop Migration Support (Phased migration tooling)

## License

Proprietary - Internal use only

## Contact

For questions or support, contact: mokhahmed92
# Test branch protection
