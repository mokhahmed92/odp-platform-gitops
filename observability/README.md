# Observability Components

**Managed by:** ArgoCD Application `observability` ([bootstrap/observability-app.yaml](../bootstrap/observability-app.yaml))
**Target Namespace:** `odp-observability`
**Epic:** 7 - Enterprise Observability

## Purpose

This directory contains monitoring, logging, and observability platform components for the ODP (Open Data Platform). All resources deploy to the isolated `odp-observability` namespace to separate observability infrastructure from platform workloads.

## Components

### Splunk Connect for Kubernetes (Epic 7.1)
- **Directory:** `splunk-connect/`
- **Purpose:** Log collection and forwarding to Splunk
- **Components:**
  - Splunk Connect Logging (DaemonSet for pod logs)
  - Splunk Connect Metrics (Deployment for cluster metrics)
  - Splunk Connect Objects (Deployment for Kubernetes events)
- **Configuration:** Helm chart with base + environment overlays
- **Target:** Splunk Enterprise/Cloud instance

### New Relic APM (Epic 7.3)
- **Directory:** `newrelic/`
- **Purpose:** Application Performance Monitoring and infrastructure metrics
- **Components:**
  - New Relic Infrastructure Agent (DaemonSet)
  - New Relic Kubernetes Integration
  - APM agents for Spark/Airflow monitoring
- **Configuration:** Helm chart with environment-specific API keys
- **Target:** New Relic One platform

## Directory Structure

```
observability/
├── README.md                          # This file
├── splunk-connect/                    # Splunk logging (Epic 7.1-7.2)
│   ├── Chart.yaml
│   ├── values-base.yaml              # Common Splunk configuration
│   ├── values-dev.yaml               # Dev environment (minimal forwarding)
│   ├── values-staging.yaml           # Staging environment
│   ├── values-prod.yaml              # Production (full logging, retention)
│   └── templates/
│       ├── logging-daemonset.yaml
│       ├── metrics-deployment.yaml
│       └── objects-deployment.yaml
└── newrelic/                          # New Relic APM (Epic 7.3)
    ├── Chart.yaml
    ├── values-base.yaml              # Common New Relic config
    ├── values-dev.yaml
    ├── values-staging.yaml
    ├── values-prod.yaml
    └── templates/
        ├── infrastructure-agent-daemonset.yaml
        └── kubernetes-integration-deployment.yaml
```

## Deployment Workflow

### GitOps Auto-Sync

1. Developer commits changes to `observability/` directory
2. Changes pushed to `main` branch (after PR approval)
3. ArgoCD detects changes within 3 minutes (polling)
4. `observability` Application automatically syncs changes to cluster
5. Helm charts rendered with appropriate values file (dev/staging/prod)
6. Resources deployed to `odp-observability` namespace

**Note:** Changes to `observability/` trigger **only** the observability Application. Platform-core and namespaces Applications remain unaffected (independent sync cycles).

### Manual Deployment (Initial Setup)

```bash
# Apply observability Application to cluster (one-time bootstrap)
kubectl apply -f bootstrap/observability-app.yaml

# Verify Application created
kubectl get application observability -n argocd

# Monitor sync status
kubectl get application observability -n argocd -w

# View Application in ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Navigate to https://localhost:8080 → Applications → observability
```

## Configuration Management

### Helm Values Pattern

Following the Helm base + environment overlay pattern (Story 1.3):

```bash
# Render Helm chart for specific environment
helm template splunk-connect observability/splunk-connect/ \
  -f observability/splunk-connect/values-base.yaml \
  -f observability/splunk-connect/values-prod.yaml \
  -n odp-observability

# Test Helm chart syntax
helm lint observability/splunk-connect/ \
  -f observability/splunk-connect/values-base.yaml \
  -f observability/splunk-connect/values-dev.yaml
```

### Secrets Management

Observability components require sensitive credentials:

- **Splunk HEC Token:** Stored in Kubernetes Secret `splunk-credentials`
- **New Relic License Key:** Stored in Kubernetes Secret `newrelic-credentials`

**Important:** Credentials **NOT** stored in Git. Use external secret management:
- Sealed Secrets (Epic 2.4)
- External Secrets Operator
- Manual `kubectl create secret` (initial setup)

Example:
```bash
# Create Splunk HEC token secret
kubectl create secret generic splunk-credentials \
  -n odp-observability \
  --from-literal=hec-token=YOUR-TOKEN-HERE

# Create New Relic license key secret
kubectl create secret generic newrelic-credentials \
  -n odp-observability \
  --from-literal=license-key=YOUR-LICENSE-KEY-HERE
```

## Testing

### Verify Logging Pipeline

```bash
# Check Splunk Connect pods running
kubectl get pods -n odp-observability -l app=splunk-connect

# View Splunk Connect logs
kubectl logs -n odp-observability -l app=splunk-connect-logging --tail=50

# Test log forwarding (create test pod with logs)
kubectl run test-logger --image=busybox -n odp-platform -- sh -c "while true; do echo 'Test log message'; sleep 5; done"

# Verify logs appear in Splunk (search: index=kubernetes source=test-logger)
```

### Verify APM Monitoring

```bash
# Check New Relic agent pods
kubectl get pods -n odp-observability -l app=newrelic-infrastructure

# View New Relic agent logs
kubectl logs -n odp-observability -l app=newrelic-infrastructure --tail=50

# Verify data in New Relic UI:
# Infrastructure → Kubernetes → Cluster Overview
```

## Rollback Procedure

Use standard GitOps rollback workflow (Story 1.5):

```bash
# Identify problematic commit
cd /c/Users/mokhtar/odp-platform-gitops
git log --oneline observability/ -10

# Revert commit
git revert <commit-hash>

# Push revert
git push origin main

# Monitor ArgoCD sync (only observability Application syncs)
kubectl get application observability -n argocd -w
```

## Troubleshooting

### Application Not Syncing

```bash
# Check Application status
kubectl describe application observability -n argocd

# Check ArgoCD Application Controller logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller --tail=100 | grep observability

# Manually trigger sync
kubectl patch application observability -n argocd \
  --type merge \
  -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"normal"}}}'
```

### Pods Not Starting

```bash
# Check pod status
kubectl get pods -n odp-observability

# Describe pod for events
kubectl describe pod <pod-name> -n odp-observability

# Check resource quotas
kubectl describe resourcequota -n odp-observability

# Verify secrets exist
kubectl get secrets -n odp-observability
```

## References

- [Splunk Connect for Kubernetes Docs](https://github.com/splunk/splunk-connect-for-kubernetes)
- [New Relic Kubernetes Integration Docs](https://docs.newrelic.com/docs/kubernetes-pixie/kubernetes-integration/get-started/introduction-kubernetes-integration/)
- [ArgoCD Application CRD](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#applications)
- [Helm Chart Standards](../docs/helm-chart-standards.md)
- [Rollback Deployment Runbook](../docs/runbooks/rollback-deployment.md)

## Related Epics and Stories

- **Epic 1:** GitOps Foundation (this Application created in Story 1.6)
- **Epic 7:** Enterprise Observability (components deployed in Stories 7.1-7.6)
  - Story 7.1: Deploy Splunk Connect for Kubernetes
  - Story 7.2: Configure Structured JSON Logging
  - Story 7.3: Deploy New Relic APM
  - Story 7.4: Create Splunk Dashboards
  - Story 7.5: Implement Real-Time Alerts
  - Story 7.6: Implement Security Audit Logging
