# PostgreSQL Migration: Bitnami to CloudNativePG

## Overview

This document outlines the migration from Bitnami PostgreSQL Helm chart to CloudNativePG (CNPG) operator. CNPG provides better Kubernetes-native PostgreSQL management with improved HA capabilities and monitoring.

## Migration Strategy

### Phase 1: Preparation (Current State)
- ✅ CNPG operator and cluster configuration created
- ✅ Basic authentication and secrets configured (superuser only)
- ✅ Basic database initialization scripts (no automatic user/database creation)
- ✅ Backup strategy available via separate backup system

### Phase 2: Deployment
1. Deploy CNPG operator
2. Deploy new CNPG cluster alongside existing Bitnami instance
3. Test connectivity and functionality

### Phase 3: Manual Database Setup
1. Create backup of current Bitnami PostgreSQL data
2. Restore data to CNPG cluster
3. **Manually create application users and databases** (as requested)
4. Validate data integrity and user setup

### Phase 4: Switchover
1. Update applications to use new CNPG service
2. Remove old Bitnami deployment
3. Clean up unused resources

## Architecture Changes

### Before (Bitnami)
- Helm-based deployment
- Bitnami PostgreSQL image
- Standard Kubernetes LoadBalancer service
- Manual pg_hba configuration
- External secret integration

### After (CNPG)
- Kubernetes operator pattern
- Official PostgreSQL images
- Native PostgreSQL LoadBalancer service
- Built-in monitoring and alerting
- **Manual user/database creation** (as requested)
- Simplified initialization (no automatic Authentik setup)

## Important Notes

### Manual Database Setup
As requested, the CNPG deployment **does not** automatically create:
- Application users (like `authentik`)
- Application databases
- User permissions and roles

**You will need to manually create these** after the cluster is deployed:
```sql
-- Example: Create Authentik user and database manually
CREATE USER authentik WITH PASSWORD 'your-password';
CREATE DATABASE authentik OWNER authentik;
GRANT ALL PRIVILEGES ON DATABASE authentik TO authentik;
ALTER USER authentik CREATEDB;
```

## Deployment Order

### Step 1: Deploy CNPG Operator (First)
```bash
# Deploy the operator first - this creates the CRDs
kubectl apply -f helm-source-cnpg.yaml
kubectl apply -f helm-release-cnpg.yaml

# Wait for CRDs to be ready
kubectl wait --for=condition=available --timeout=300s deployment/cnpg -n tools
```

### Step 2: Deploy Secrets (Second)
```bash
# Deploy the modified secret-postgres.yaml
kubectl apply -f ../../../secrets/secret-postgres.yaml
```

### Step 3: Deploy PostgreSQL Cluster (Last)
```bash
# Deploy the cluster after operator and secrets are ready
kubectl apply -f cluster-cnpg.yaml
kubectl apply -f service-cnpg.yaml
```

## Files Structure

### Core CNPG Files
- `cluster-cnpg.yaml` - Main PostgreSQL cluster definition
- `helm-source-cnpg.yaml` - CNPG Helm repository
- `helm-release-cnpg.yaml` - CNPG operator deployment
- `service-cnpg.yaml` - LoadBalancer service (preserves IP)

### Authentication & Secrets
- `../../../secrets/secret-postgres.yaml` - Modified for CNPG compatibility

## Configuration Mapping

| Component | Bitnami | CNPG |
|-----------|---------|------|
| Image | `bitnami/postgresql:latest` | `ghcr.io/cloudnative-pg/postgresql:16.0` |
| Storage | 20Gi NFS | 20Gi NFS (same) |
| LoadBalancer IP | `10.42.20.70` | `10.42.20.70` (preserved) |
| Authentication | External Secrets | External Secrets (reutilizado) |
| Init Scripts | ConfigMap scripts | ❌ Eliminado (manual) |
| Monitoring | External | Built-in PodMonitor |

## Migration Steps

### 1. Pre-Migration Backup
```bash
# Create final backup of Bitnami PostgreSQL
kubectl exec -it postgresql-0 -- bash -c "pg_dumpall -U postgres > /tmp/final_backup.sql"
```

### 2. Deploy CNPG System
```bash
# Deploy in correct order
kubectl apply -k synergia-k8s/bootstrap/kubernetes/apps/tools/postgresql/app/
```

### 3. Wait for Resources
```bash
# Wait for operator
kubectl wait --for=condition=available --timeout=300s deployment/cnpg -n tools

# Wait for cluster
kubectl wait --for=condition=ready --timeout=600s cluster/postgresql-cnpg -n tools
```

### 4. Manual Database Setup
```bash
# Connect to new cluster
kubectl exec -it postgresql-cnpg-1 -- psql -U postgres

# Create your application users and databases manually
# CREATE USER authentik WITH PASSWORD 'your-password';
# CREATE DATABASE authentik OWNER authentik;
```

### 5. Data Migration
```bash
# Restore data to CNPG cluster when ready
kubectl exec -it postgresql-cnpg-1 -- psql -U postgres < /tmp/final_backup.sql
```

### 6. Validation
- Verify all databases exist
- Check data integrity
- Validate application connectivity
- Confirm backup functionality

## Troubleshooting

### Common Issues

1. **LoadBalancer IP not assigned**
   - Check MetalLB configuration
   - Verify IP is available

2. **Secrets not populated**
   - Check external-secrets operator
   - Verify Bitwarden credentials

3. **Cluster not starting**
   - Check PVC creation
   - Verify storage class availability

4. **CRD not found error**
   - Deploy CNPG operator first
   - Wait for CRDs to be installed

### Useful Commands

```bash
# Check deployment order
kubectl get events -n tools --sort-by=.metadata.creationTimestamp

# Check cluster status
kubectl get cluster postgresql-cnpg -n tools

# View cluster events
kubectl describe cluster postgresql-cnpg -n tools

# Check pod logs
kubectl logs -l postgresql=postgresql-cnpg -n tools

# Check secret creation
kubectl get secret postgresql-cnpg-secret -n tools -o yaml
```

## Rollback Plan

If issues occur, rollback to Bitnami:

1. **Keep Bitnami deployment** (in other app directories)
2. **Update applications** back to old service
3. **Remove CNPG resources** if needed

## Monitoring Integration

CNPG provides built-in monitoring:

- **PodMonitor** for Prometheus integration
- **Cluster status** via `kubectl get cluster`
- **Events** via `kubectl get events`

## Post-Migration Tasks

1. **Update documentation** with new service details
2. **Remove old Bitnami resources** after successful migration
3. **Configure monitoring alerts** for CNPG cluster
4. **Set up regular backup verification**

## Benefits of Migration

- **Better HA capabilities** - Native Kubernetes operator patterns
- **Official PostgreSQL images** - No vendor lock-in
- **Enhanced monitoring** - Built-in Prometheus integration
- **Improved resource management** - More efficient resource utilization
- **Simplified configuration** - Less files to maintain

## Support

For issues with CNPG:
- Check [CloudNativePG documentation](https://cloudnative-pg.io/)
- Review [GitHub issues](https://github.com/cloudnative-pg/cloudnative-pg/issues)
- Monitor cluster status with `kubectl describe cluster postgresql-cnpg -n tools`