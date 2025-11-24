# Domain Namespace Template

This directory contains a reusable template for creating isolated, multi-tenant data domain namespaces in the Open Data Platform (ODP).

## Purpose

The domain-template provides a standardized approach to provisioning Kubernetes namespaces for data teams with:
- **Resource isolation** via ResourceQuotas
- **Network isolation** via NetworkPolicies (deny-all default with explicit allow-list)
- **Role-based access control** integrated with enterprise LDAP/AD
- **Platform configuration** for Spark and Airflow workloads
- **Queue management** for Volcano scheduler integration

## Template Structure

The template consists of 7 YAML manifests using the `{{DOMAIN}}` placeholder:

```
domain-template/
├── namespace.yaml          # Namespace with labels
├── resource-quota.yaml     # CPU, memory, PVC, pod limits
├── network-policy.yaml     # Network isolation policy
├── rbac.yaml              # 3 Roles + RoleBindings (admin, developer, viewer)
├── spark-config.yaml      # Spark defaults ConfigMap
├── airflow-config.yaml    # Airflow configuration ConfigMap
└── volcano-queue.yaml     # Volcano Queue CRD for gang scheduling
```

## How to Use

### 1. Create a New Domain Namespace

**Option A: Manual Instantiation**

1. Copy the `domain-template` directory to a new `domain-{NAME}` directory:
   ```bash
   cp -r namespaces/domain-template namespaces/domain-{NAME}
   ```

2. Replace all `{{DOMAIN}}` placeholders with your domain name:
   ```bash
   cd namespaces/domain-{NAME}
   find . -type f -name "*.yaml" -exec sed -i 's/{{DOMAIN}}/{NAME}/g' {} +
   ```

   Example: For "finance" domain:
   ```bash
   find . -type f -name "*.yaml" -exec sed -i 's/{{DOMAIN}}/finance/g' {} +
   ```

3. Commit to Git and push:
   ```bash
   git add namespaces/domain-{NAME}
   git commit -m "Add {NAME} domain namespace"
   git push origin main
   ```

4. ArgoCD will automatically sync and create the namespace with all resources.

**Option B: Scripted Instantiation**

Create a helper script (recommended):
```bash
#!/bin/bash
DOMAIN=$1
if [ -z "$DOMAIN" ]; then
  echo "Usage: $0 <domain-name>"
  exit 1
fi

cp -r namespaces/domain-template namespaces/domain-$DOMAIN
find namespaces/domain-$DOMAIN -type f -name "*.yaml" -exec sed -i "s/{{DOMAIN}}/$DOMAIN/g" {} +
echo "Domain namespace '$DOMAIN' created. Review and commit to Git."
```

### 2. Verify Deployment

After ArgoCD syncs (typically 1-3 minutes), verify resources:

```bash
# Check namespace exists
kubectl get namespace domain-{NAME}

# Check resource quota
kubectl get resourcequota -n domain-{NAME}

# Check network policy
kubectl get networkpolicy -n domain-{NAME}

# Check RBAC
kubectl get role,rolebinding -n domain-{NAME}

# Check ConfigMaps
kubectl get configmap -n domain-{NAME}

# Check Volcano Queue
kubectl get queue {NAME}-queue
```

## Resource Specifications

### Namespace
- Name: `domain-{{DOMAIN}}`
- Labels:
  - `app.kubernetes.io/part-of: odp-platform`
  - `odp.platform/tenant: {{DOMAIN}}`
  - `odp.platform/isolation: strict`

### ResourceQuota
- **Requests**: 200 CPU cores, 800Gi RAM
- **Limits**: 400 CPU cores (2x burst), 1600Gi RAM (2x burst)
- **PVCs**: Max 50 PersistentVolumeClaims
- **Pods**: Max 500 pods

### NetworkPolicy
**Default**: Deny all ingress and egress

**Allowed Egress**:
- DNS (kube-system namespace, UDP/53)
- S3/Cloudian (10.0.0.0/8, TCP/443)
- Splunk HEC (10.100.0.0/16, TCP/8088)

**Allowed Ingress**:
- Same namespace only

### RBAC Roles

**1. Admin Role** (`{{DOMAIN}}-admin`)
- Full namespace access (`*/*`)
- Bound to AD/LDAP group: `{{DOMAIN}}-admins`

**2. Developer Role** (`{{DOMAIN}}-developer`)
- Deploy pipelines and workloads
- Manage: Pods, Services, ConfigMaps, Secrets, PVCs, Deployments, StatefulSets, Jobs, CronJobs, SparkApplications, Volcano Jobs
- Bound to AD/LDAP group: `{{DOMAIN}}-developers`

**3. Viewer Role** (`{{DOMAIN}}-viewer`)
- Read-only access to all resources
- Bound to AD/LDAP group: `{{DOMAIN}}-viewers`

### Spark Configuration
- Executor: 2 instances, 4 cores, 8Gi memory
- Driver: 2 cores, 4Gi memory
- Scheduler: Volcano
- Logging: S3 (`s3a://{{DOMAIN}}-spark-logs/`)
- S3 Endpoint: `https://s3.cloudian.internal`

### Airflow Configuration
- Executor: KubernetesExecutor
- Worker Image: `apache/airflow:2.7.3-python3.10`
- Logging: S3 (`s3://{{DOMAIN}}-airflow-logs`)
- Webserver: `https://airflow-{{DOMAIN}}.odp.internal`

### Volcano Queue
- Weight: 1 (fair-share)
- Capacity: 200 CPU, 800Gi RAM (matches ResourceQuota)
- Reclaimable: `true` (can borrow from other queues)

## Customization Guidelines

### When to Customize
After instantiating a domain from the template, you may need to adjust:
- **Resource Quota**: If domain has different capacity requirements
- **Network Policy**: Add egress rules for domain-specific external services
- **Spark/Airflow Config**: Tune executor sizes or add domain-specific settings

### When NOT to Customize
Do not modify:
- Namespace labels (used for platform automation)
- RBAC role names/structure (unless coordinating with platform team)
- Volcano Queue weight (managed centrally for fair-share)

## GitOps Integration

This template is managed by ArgoCD Application `namespaces`:
- **Repo**: `https://github.com/mokhahmed92/odp-platform-gitops`
- **Path**: `namespaces/`
- **Sync Policy**: Auto-sync with self-heal and prune
- **Recurse**: `true` (watches all subdirectories)

Any changes committed to `namespaces/domain-*` directories will automatically sync to the cluster.

## Troubleshooting

### Namespace not created after commit
1. Check ArgoCD Application status:
   ```bash
   kubectl get application namespaces -n argocd
   ```
2. Check for sync errors:
   ```bash
   kubectl describe application namespaces -n argocd
   ```
3. Manually trigger sync (if auto-sync delayed):
   ```bash
   kubectl patch application namespaces -n argocd --type merge \
     -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"HEAD"}}}'
   ```

### ResourceQuota errors
If pods fail to schedule with "exceeded quota" errors:
1. Check current usage:
   ```bash
   kubectl describe resourcequota -n domain-{NAME}
   ```
2. Either:
   - Scale down existing workloads
   - Request quota increase (modify `resource-quota.yaml` and commit)

### NetworkPolicy blocking traffic
If pods cannot reach required services:
1. Check NetworkPolicy:
   ```bash
   kubectl get networkpolicy -n domain-{NAME} -o yaml
   ```
2. Add egress rule for the destination CIDR/port
3. Commit and push changes

### RBAC permission denied
If users report "forbidden" errors:
1. Verify user is in correct AD/LDAP group (`{DOMAIN}-admins`, `{DOMAIN}-developers`, or `{DOMAIN}-viewers`)
2. Check RoleBinding:
   ```bash
   kubectl describe rolebinding -n domain-{NAME}
   ```
3. If group name is different, update `rbac.yaml` and commit

## Examples

Two example instantiations are provided:
- **domain-finance**: Finance data team namespace
- **domain-marketing**: Marketing data team namespace

These serve as references for correct placeholder replacement.

## Related Documentation

- **Epic 2 Tech Spec**: `/docs/sprint-artifacts/tech-spec-epic-2.md`
- **Architecture**: `/docs/architecture.md`
- **ArgoCD Applications**: `/infra-apps/`
- **Volcano Scheduler**: Epic 3 (future work)

## Support

For issues with the template or domain provisioning:
1. Check ArgoCD Application status
2. Review GitOps repository commit history
3. Contact platform team for quota or policy exceptions
