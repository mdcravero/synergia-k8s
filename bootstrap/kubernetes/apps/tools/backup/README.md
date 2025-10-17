# Mega.nz Backup System

A reusable, Kubernetes-native backup solution using rclone and Mega.nz cloud storage, designed for the Synergia-K8s homelab environment.

## Overview

This backup system provides:
- **Mega.nz integration** via rclone
- **Scheduled backups** using Kubernetes CronJobs
- **Multiple backup types**: PostgreSQL, filesystem, configurations
- **Reusable templates** for different projects
- **External secret integration** with Bitwarden
- **Monitoring and alerting** capabilities

## Architecture

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Kubernetes    │    │     Rclone       │    │    Mega.nz      │
│   CronJobs      │───▶│   Sidecar/Pod    │───▶│   Cloud Storage │
│                 │    │                  │    │                 │
│ - Scheduled     │    │ - Authentication │    │ - Unlimited     │
│ - Event-driven  │    │ - Data transfer  │    │ - Encrypted     │
│ - Manual        │    │ - Compression    │    │ - Versioned     │
└─────────────────┘    └──────────────────┘    └─────────────────┘
```

## Quick Start

### 1. Prerequisites

- Mega.nz account with API access
- Bitwarden secret store configured
- External Secrets Operator installed

### 2. Setup Mega.nz Credentials

Add your Mega.nz credentials to Bitwarden under `mega-backup-credentials`:

```yaml
mega-backup-credentials:
  username: "your-mega-email@example.com"
  password: "your-mega-password"
```
Is neccesary to obfuscate a password with rclone
You can use a docker image:
```bash
docker run --rm rclone/rclone obscure "mypassword"
```

### 3. Deploy Backup System

```bash
# Deploy the backup infrastructure
kubectl apply -k synergia-k8s/bootstrap/kubernetes/apps/tools/backup/app/

# Verify deployment
kubectl get pods -n backup-system
kubectl get cronjobs -n backup-system
```

### 4. Create Your First Backup

Copy and customize an example:

```bash
# Copy PostgreSQL backup example
cp app/examples-postgresql.yaml app/my-postgres-backup.yaml

# Edit the configuration for your database
vim app/my-postgres-backup.yaml

# Apply the backup
kubectl apply -f app/my-postgres-backup.yaml
```

## Backup Types

### PostgreSQL Backups

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: my-postgres-backup
  namespace: backup-system
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: postgres-backup
            image: postgres:16
            command: ["/bin/bash", "-c"]
            args:
            - |
              pg_dump -h $POSTGRES_HOST -U $POSTGRES_USER -d $DATABASE_NAME -f /tmp/backup.sql
              rclone copy /tmp/backup.sql mega:/backups/postgres/$(date +%Y-%m-%d)/
            env:
            - name: POSTGRES_HOST
              value: "my-postgres-service.namespace.svc.cluster.local"
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: my-postgres-secret
                  key: username
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: my-postgres-secret
                  key: password
            - name: DATABASE_NAME
              value: "mydatabase"
```

### Filesystem Backups

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: my-filesystem-backup
  namespace: backup-system
spec:
  schedule: "0 3 * * 0"  # Weekly on Sunday
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: filesystem-backup
            image: rclone/rclone:latest
            command: ["/bin/sh", "-c"]
            args:
            - |
              rclone sync /source/data mega:/backups/files/$(date +%Y-%m-%d) \
                --include="**/*.mp4" \
                --include="**/*.jpg" \
                --exclude="**/*"
            volumeMounts:
            - name: source-data
              mountPath: /source/data
          volumes:
          - name: source-data
            persistentVolumeClaim:
              claimName: my-data-pvc
```

## Configuration Examples

### Project-Specific Backups

#### Media Server (Jellyfin/Plex)
```yaml
# Daily media backup (selective)
schedule: "0 4 * * *"
backup_path: "mega:/backups/media/jellyfin"
include_patterns: ["**/*.mp4", "**/*.mkv", "**/*.avi"]
exclude_patterns: ["**/.git/**", "**/tmp/**"]
```

#### Database Applications
```yaml
# Hourly database backup
schedule: "0 * * * *"
backup_path: "mega:/backups/databases/authentik"
database_name: "authentik"
retention_days: 30
```

#### Configuration Backups
```yaml
# Daily config backup
schedule: "0 6 * * *"
backup_path: "mega:/backups/config/kubernetes"
include_resources: ["configmaps", "secrets", "deployments"]
namespaces: ["media", "security", "tools"]
```

## Advanced Features

### Custom Rclone Configuration

For advanced Mega.nz configurations:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: custom-rclone-config
  namespace: backup-system
type: Opaque
data:
  rclone.conf: |
    [mega-custom]
    type = mega
    user = your-username
    pass = your-password
    hard_delete = true
    debug_https = true
```

### Backup with Compression

```yaml
command: ["/bin/bash", "-c"]
args:
- |
  # Create compressed backup
  tar -czf /tmp/backup-$(date +%Y%m%d-%H%M%S).tar.gz /source/data
  rclone copy /tmp/backup-*.tar.gz mega:/backups/compressed/$(date +%Y-%m-%d)/
  # Cleanup old files
  rm /tmp/backup-*.tar.gz
```

### Multi-Stage Backups

```yaml
command: ["/bin/bash", "-c"]
args:
- |
  # Stage 1: Database dump
  pg_dump -h $DB_HOST -U $DB_USER -d $DB_NAME > /tmp/db.sql

  # Stage 2: File archive
  tar -czf /tmp/files.tar.gz /source/files

  # Stage 3: Upload both
  rclone copy /tmp/db.sql mega:/backups/db/$(date +%Y-%m-%d)/
  rclone copy /tmp/files.tar.gz mega:/backups/files/$(date +%Y-%m-%d)/

  # Cleanup
  rm /tmp/db.sql /tmp/files.tar.gz
```

## Monitoring and Alerting

### Check Backup Status

```bash
# List all backup jobs
kubectl get cronjobs -n backup-system

# Check job history
kubectl get jobs -n backup-system

# View logs
kubectl logs -l job-name=my-backup-job -n backup-system

# Check rclone pod logs
kubectl logs deployment/rclone-backup -n backup-system
```

### Prometheus Integration

The backup system exposes metrics:

```bash
# Port forward to access metrics
kubectl port-forward -n backup-system deployment/rclone-backup 8080:8080

# Query metrics
curl http://localhost:8080/metrics
```

## Troubleshooting

### Common Issues

#### Authentication Failures
```bash
# Check secret contents
kubectl get secret mega-credentials -n backup-system -o yaml

# Test rclone configuration
kubectl exec -it deployment/rclone-backup -n backup-system -- rclone ls mega:
```

#### Permission Errors
```bash
# Check pod security context
kubectl describe pod rclone-backup -n backup-system

# Verify volume mounts
kubectl exec -it deployment/rclone-backup -n backup-system -- mount
```

#### Network Issues
```bash
# Test Mega.nz connectivity
kubectl exec -it deployment/rclone-backup -n backup-system -- rclone ping mega:

# Check DNS resolution
kubectl exec -it deployment/rclone-backup -n backup-system -- nslookup api.mega.nz
```

### Debug Mode

Enable debug logging:

```yaml
env:
- name: RCLONE_DEBUG
  value: "true"
- name: RCLONE_VERBOSE
  value: "2"
```

## Security Considerations

### Secret Management
- All credentials stored in Kubernetes secrets
- External Secrets Operator integration
- Automatic secret rotation support

### Network Security
- Internal cluster communication only
- No external access to backup pods
- Encrypted data transfer to Mega.nz

### Access Control
- RBAC policies for backup operations
- Namespace isolation
- Limited service account permissions

## Performance Optimization

### Large Backup Jobs
```yaml
resources:
  requests:
    memory: "1Gi"
    cpu: "500m"
  limits:
    memory: "2Gi"
    cpu: "1000m"

# Increase timeout for large backups
activeDeadlineSeconds: 7200  # 2 hours
```

### Parallel Backups
```yaml
# Use rclone with multiple threads
command: ["rclone", "sync", "source", "mega:destination", "--transfers", "4"]
```

## Integration Examples

### With PostgreSQL CNPG
```yaml
# Reference the backup system from your app
env:
- name: BACKUP_COMMAND
  value: "kubectl create job postgres-backup-$(date +%s) -n backup-system --from=cronjob/daily-postgresql-backup"
```

### With Flux
```yaml
# Trigger backup on config changes
apiVersion: notification.toolkit.fluxcd.io/v1beta2
kind: Provider
metadata:
  name: backup-trigger
spec:
  type: generic
  address: "http://rclone-backup.backup-system/trigger"
```

## Contributing

### Adding New Backup Types

1. Create new job template in `app/backup-jobs.yaml`
2. Add example in `app/examples-*.yaml`
3. Update this documentation
4. Test thoroughly

### Custom Rclone Configurations

1. Add new secret in `secret-mega.yaml`
2. Update deployment environment variables
3. Document the configuration

## Support

For issues and questions:
- Check the troubleshooting section
- Review Kubernetes logs
- Test rclone commands manually
- Verify Mega.nz account settings

## License

Part of the Synergia-K8s homelab project.