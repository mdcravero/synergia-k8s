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

## Configuration Mapping

| Component | Bitnami | CNPG |
|-----------|---------|------|
| Image | `bitnami/postgresql:latest` | `ghcr.io/cloudnative-pg/postgresql:16.0` |
| Storage | 20Gi NFS | 20Gi NFS (same) |
| LoadBalancer IP | `10.42.20.70` | `10.42.20.70` (preserved) |
| Authentication | External Secrets | External Secrets + Kubernetes Secrets |
| Init Scripts | ConfigMap scripts | ConfigMap scripts (same) |
| Monitoring | External | Built-in PodMonitor |

## Files Created

### Core CNPG Files
- `cluster-cnpg.yaml` - Main PostgreSQL cluster definition
- `helm-source-cnpg.yaml` - CNPG Helm repository
- `helm-release-cnpg.yaml` - CNPG operator deployment
- `service-cnpg.yaml` - LoadBalancer service (preserves IP)

### Authentication & Secrets
- `secret-cnpg.yaml` - Database user secrets
- External secrets integration maintained

### Initialization
- `configmap-cnpg-init.yaml` - Database initialization scripts
- Authentik database creation preserved

### Backup Strategy
- `backup-cnpg-mega.yaml` - Mega.nz backup configuration
- Scheduled backups with retention policy

## Deployment Order

1. **Deploy CNPG Operator**
   ```bash
   kubectl apply -k .
   ```

2. **Wait for CRDs to be ready**
   ```bash
   kubectl wait --for=condition=available --timeout=300s deployment/cnpg -n tools
   ```

3. **Deploy PostgreSQL Cluster**
   ```bash
   kubectl apply -f cluster-cnpg.yaml
   ```

4. **Verify cluster status**
   ```bash
   kubectl get cluster postgresql-cnpg -n tools -o yaml
   ```

## Migration Steps

### 1. Pre-Migration Backup
```bash
# Create final backup of Bitnami PostgreSQL
kubectl exec -it postgresql-0 -- bash -c "pg_dumpall -U postgres > /tmp/final_backup.sql"
```

### 2. Deploy CNPG Cluster
```bash
# Deploy new CNPG cluster (already configured)
kubectl apply -f cluster-cnpg.yaml
```

### 3. Data Migration
```bash
# Restore data to CNPG cluster when ready
kubectl exec -it postgresql-cnpg-1 -- bash -c "psql -U postgres < /tmp/final_backup.sql"
```

### 4. Validation
- Verify all databases exist
- Check data integrity
- Validate application connectivity
- Confirm backup functionality

## Rollback Plan

If issues occur, rollback to Bitnami:

1. **Keep Bitnami deployment** (commented out in kustomization.yaml)
2. **Update applications** back to old service
3. **Remove CNPG resources** if needed

## Monitoring Integration

CNPG provides built-in monitoring:

- **PodMonitor** for Prometheus integration
- **Cluster status** via `kubectl get cluster`
- **Backup status** via `kubectl get scheduledbackup`

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

### Useful Commands

```bash
# Check cluster status
kubectl get cluster postgresql-cnpg -n tools

# View cluster events
kubectl describe cluster postgresql-cnpg -n tools

# Check pod logs
kubectl logs -l postgresql=postgresql-cnpg -n tools

# View backup status
kubectl get scheduledbackup -n tools
```

## Post-Migration Tasks

1. **Update documentation** with new service details
2. **Remove old Bitnami resources** after successful migration
3. **Configure monitoring alerts** for CNPG cluster
4. **Set up regular backup verification**

## Benefits of Migration

- **Better HA capabilities** - Native Kubernetes operator patterns
- **Official PostgreSQL images** - No vendor lock-in
- **Enhanced monitoring** - Built-in Prometheus integration
- **Improved backup/restore** - Native Kubernetes backup strategies
- **Better resource management** - More efficient resource utilization

## Support

For issues with CNPG:
- Check [CloudNativePG documentation](https://cloudnative-pg.io/)
- Review [GitHub issues](https://github.com/cloudnative-pg/cloudnative-pg/issues)
- Monitor cluster status with `kubectl describe cluster postgresql-cnpg -n tools`