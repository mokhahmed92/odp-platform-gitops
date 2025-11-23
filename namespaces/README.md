# Multi-Tenant Namespace Definitions

**Managed by:** ArgoCD Application `namespaces` ([bootstrap/namespaces-app.yaml](../bootstrap/namespaces-app.yaml))
**Resource Scope:** Cluster-wide (Namespace, ClusterRole, ClusterRoleBinding)
**Epic:** 2 - Multi-Tenant Platform

## Purpose

This directory contains namespace definitions for multi-tenant data domains in the ODP (Open Data Platform). Each data domain (e.g., Finance, Marketing, HR) gets an isolated namespace with dedicated resource quotas, network policies, and RBAC configuration.

## Multi-Tenancy Model

**Tenant Isolation Strategy:**
- **Namespace Isolation:** Each data domain has a dedicated Kubernetes namespace
- **Resource Quotas:** CPU/memory limits per namespace prevent resource exhaustion
- **Network Policies:** Inter-namespace communication controlled via NetworkPolicy resources
- **RBAC:** Role-based access control ensures teams only access their namespace
- **Storage Quotas:** PersistentVolume claims limited per namespace

## Directory Structure

```
namespaces/
├── README.md                          # This file
├── domain-finance/                    # Finance data domain (Epic 2.1-2.4)
│   ├── namespace.yaml                # Namespace resource
│   ├── resourcequota.yaml            # CPU/memory quotas
│   ├── networkpolicy.yaml            # Network isolation
│   ├── rbac.yaml                     # Roles, RoleBindings for finance team
│   └── argocd-project.yaml           # ArgoCD AppProject for self-service
├── domain-marketing/                  # Marketing data domain
│   ├── namespace.yaml
│   ├── resourcequota.yaml
│   ├── networkpolicy.yaml
│   ├── rbac.yaml
│   └── argocd-project.yaml
├── domain-hr/                         # HR data domain
│   ├── namespace.yaml
│   ├── resourcequota.yaml
│   ├── networkpolicy.yaml
│   ├── rbac.yaml
│   └── argocd-project.yaml
└── shared/                            # Shared resources (ClusterRoles, etc.)
    └── cluster-rbac.yaml
```

## Namespace Template

Each data domain follows a standard template (created in Epic 2):

### 1. Namespace Resource

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: odp-domain-finance
  labels:
    app.kubernetes.io/part-of: odp-platform
    odp.platform/tenant: finance
    odp.platform/isolation: strict
```

### 2. ResourceQuota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: finance-quota
  namespace: odp-domain-finance
spec:
  hard:
    requests.cpu: "50"       # 50 CPU cores
    requests.memory: "200Gi" # 200GB RAM
    limits.cpu: "100"        # 100 CPU cores max
    limits.memory: "400Gi"   # 400GB RAM max
    persistentvolumeclaims: "20"  # Max 20 PVCs
```

### 3. NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: finance-isolation
  namespace: odp-domain-finance
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Allow from odp-platform namespace (Spark Operator, Airflow)
    - from:
      - namespaceSelector:
          matchLabels:
            app.kubernetes.io/name: odp-platform
  egress:
    # Allow to S3 storage (Cloudian)
    # Allow to DNS
    # Deny all other egress
```

### 4. RBAC Configuration

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: finance-developer
  namespace: odp-domain-finance
rules:
  - apiGroups: ["", "apps", "batch"]
    resources: ["pods", "services", "deployments", "jobs", "configmaps", "secrets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: finance-team-binding
  namespace: odp-domain-finance
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: finance-developer
subjects:
  - kind: Group
    name: finance-team  # SSO group from Enterprise IdP (Story 2.4)
    apiGroup: rbac.authorization.k8s.io
```

### 5. ArgoCD AppProject (Self-Service Deployment)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: finance-project
  namespace: argocd
spec:
  description: Finance team self-service deployments
  sourceRepos:
    - 'https://github.com/finance-team/data-pipelines'
  destinations:
    - namespace: odp-domain-finance
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'
```

## Deployment Workflow

### GitOps Auto-Sync

1. Platform engineer creates new domain directory (e.g., `domain-sales/`)
2. Adds namespace.yaml, resourcequota.yaml, networkpolicy.yaml, rbac.yaml
3. Commits to Git and creates PR
4. PR reviewed by platform team (CODEOWNERS: @platform-team)
5. PR merged to `main` branch
6. ArgoCD `namespaces` Application detects change within 3 minutes
7. Namespace and associated resources created in cluster

**Note:** Namespace creation happens in **sync wave 1** (before platform-core and observability), ensuring foundation is ready for platform components.

### Manual Deployment (Initial Setup)

```bash
# Apply namespaces Application to cluster (one-time bootstrap)
kubectl apply -f bootstrap/namespaces-app.yaml

# Verify Application created
kubectl get application namespaces -n argocd

# Monitor sync status
kubectl get application namespaces -n argocd -w

# View created namespaces
kubectl get namespaces -l app.kubernetes.io/part-of=odp-platform
```

## Testing

### Verify Namespace Creation

```bash
# List all ODP namespaces
kubectl get namespaces -l app.kubernetes.io/part-of=odp-platform

# Check specific namespace
kubectl get namespace odp-domain-finance -o yaml

# Verify resource quota
kubectl describe resourcequota -n odp-domain-finance

# Verify network policy
kubectl get networkpolicy -n odp-domain-finance
```

### Test Resource Quota Enforcement

```bash
# Try to exceed CPU quota (should fail)
kubectl run quota-test -n odp-domain-finance \
  --image=nginx \
  --requests='cpu=60' \
  --limits='cpu=60'

# Expected: Error from quota enforcement
```

### Test Network Isolation

```bash
# Create test pod in finance namespace
kubectl run test-pod -n odp-domain-finance --image=busybox -- sleep 3600

# Try to access pod from marketing namespace (should fail)
kubectl run curl-test -n odp-domain-marketing --image=curlimages/curl -- \
  curl http://test-pod.odp-domain-finance.svc.cluster.local
```

### Test RBAC

```bash
# Impersonate finance team member
kubectl auth can-i create pods -n odp-domain-finance --as=user@finance-team

# Try to create pods in marketing namespace (should fail)
kubectl auth can-i create pods -n odp-domain-marketing --as=user@finance-team
```

## Rollback Procedure

Use standard GitOps rollback workflow (Story 1.5):

```bash
# Identify problematic commit
cd /c/Users/mokhtar/odp-platform-gitops
git log --oneline namespaces/ -10

# Revert commit
git revert <commit-hash>

# Push revert
git push origin main

# Monitor ArgoCD sync (only namespaces Application syncs)
kubectl get application namespaces -n argocd -w
```

**Warning:** Reverting namespace deletion will **not** restore data. Always backup PersistentVolumes before namespace changes.

## Adding a New Data Domain

To onboard a new data domain (e.g., "Sales"):

1. **Create domain directory:**
   ```bash
   mkdir -p namespaces/domain-sales
   ```

2. **Copy template files:**
   ```bash
   cp namespaces/domain-finance/namespace.yaml namespaces/domain-sales/
   cp namespaces/domain-finance/resourcequota.yaml namespaces/domain-sales/
   cp namespaces/domain-finance/networkpolicy.yaml namespaces/domain-sales/
   cp namespaces/domain-finance/rbac.yaml namespaces/domain-sales/
   cp namespaces/domain-finance/argocd-project.yaml namespaces/domain-sales/
   ```

3. **Update resource names:**
   - Replace "finance" with "sales" in all YAML files
   - Update namespace labels, quota values, RBAC groups

4. **Commit and create PR:**
   ```bash
   git add namespaces/domain-sales/
   git commit -m "feat(namespaces): Add sales data domain

   - Create odp-domain-sales namespace
   - Configure resource quotas (50 CPU, 200Gi RAM)
   - Set up network isolation
   - Configure RBAC for sales-team SSO group"

   git push origin feature/add-sales-domain
   ```

5. **Follow PR approval workflow** (Story 1.4)

6. **ArgoCD auto-syncs** new namespace after PR merge

## Troubleshooting

### Namespace Stuck in Terminating State

```bash
# Check for finalizers blocking deletion
kubectl get namespace odp-domain-finance -o yaml | grep finalizers

# Remove finalizers (if safe)
kubectl patch namespace odp-domain-finance -p '{"metadata":{"finalizers":[]}}' --type=merge

# Check for resources blocking deletion
kubectl api-resources --verbs=list --namespaced -o name | \
  xargs -n 1 kubectl get --show-kind --ignore-not-found -n odp-domain-finance
```

### Application Not Syncing

```bash
# Check Application status
kubectl describe application namespaces -n argocd

# Check ArgoCD logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller --tail=100 | grep namespaces

# Manually trigger sync
kubectl patch application namespaces -n argocd \
  --type merge \
  -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"normal"}}}'
```

### ResourceQuota Not Enforcing

```bash
# Verify ResourceQuota created
kubectl get resourcequota -n odp-domain-finance

# Check quota usage
kubectl describe resourcequota -n odp-domain-finance

# Verify pods have resource requests/limits
kubectl get pods -n odp-domain-finance -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].resources}{"\n"}{end}'
```

## References

- [Kubernetes Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [ArgoCD AppProjects](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/)
- [Rollback Deployment Runbook](../docs/runbooks/rollback-deployment.md)

## Related Epics and Stories

- **Epic 1:** GitOps Foundation (this Application created in Story 1.6)
- **Epic 2:** Multi-Tenant Platform (namespace templates created in Stories 2.1-2.5)
  - Story 2.1: Create Multi-Tenant Namespace Template
  - Story 2.2: Implement Resource Quotas per Namespace
  - Story 2.3: Implement Network Isolation with NetworkPolicies
  - Story 2.4: Implement RBAC with Enterprise SSO Integration
  - Story 2.5: Create ArgoCD Projects for Self-Service Deployments
