# PostgreSQL Cluster (Zalando Operator)

This directory contains the configuration for the Zalando PostgreSQL operator and the PostgreSQL cluster deployed in Kubernetes using FluxCD.

## Directory Structure

```
postgresql/
├── operator/              # Zalando PostgreSQL operator
│   ├── helm-source.yaml   # HelmRepository for postgres-operator
│   ├── helm-release.yaml  # HelmRelease to install the operator
│   ├── configmap-zalando.yaml  # Operator configuration
│   └── kustomization.yaml # Kustomization for operator
├── cluster/               # PostgreSQL cluster definition
│   ├── cluster-zalando.yaml  # PostgreSQL CR manifest
│   └── kustomization.yaml # Kustomization for cluster
├── utils/                 # Utility scripts
│   ├── Dockerfile.spilo  # Custom Spilo image (for ARM64)
│   ├── README.md
│   └── backup/
│       ├── Dockerfile
│       └── dump.sh
└── README.md             # This file
```

## Deployment

### Deploy the operator first (via FluxCD):
```bash
kubectl apply -k kubernetes/apps/tools/postgresql/operator/
```

### Then deploy the cluster:
```bash
kubectl apply -k kubernetes/apps/tools/postgresql/cluster/
```

Or deploy everything together (FluxCD does this automatically):
```bash
kubectl apply -k kubernetes/apps/tools/postgresql/
```

## Common Operations

### Check PostgreSQL pod status:
```bash
kubectl get pods -n tools -l app=postgres
```

### Get PostgreSQL credentials:
```bash
# Get the secret containing passwords
kubectl get secret -n tools postgres-cluster.credentials -o yaml

# Decode a specific password
kubectl get secret -n tools postgres-cluster.credentials -o jsonpath='{.data.authelia_user}' | base64 -d
```

### Connect to PostgreSQL:
```bash
# Get pod name
POD=$(kubectl get pods -n tools -l app=postgres -o jsonpath='{.items[0].metadata.name}')

# Connect as superuser
kubectl exec -n tools $POD -- psql -U postgres

# Connect to a specific database
kubectl exec -n tools $POD -- psql -U postgres -d authelia
```

### List all databases:
```bash
kubectl exec -n tools postgres-cluster -- psql -U postgres -l
```

### List tables in a database:
```bash
kubectl exec -n tools postgres-cluster -- psql -U postgres -d authelia -c "\dt"
```

## Restoring a Backup

### Download backup from S3:
```bash
aws s3 cp s3://your-bucket/path/to/backup.sql.gz .
gunzip backup.sql.gz
```

### Copy backup to PostgreSQL pod:
```bash
kubectl cp backup.sql.gz tools/postgres-cluster:/tmp/backup.sql.gz
kubectl exec -n tools postgres-cluster -- gunzip /tmp/backup.sql.gz
```

### Drop and recreate the database (recommended to avoid duplicate table errors):
```bash
# Drop the existing database
kubectl exec -n tools postgres-cluster -- psql -U postgres -c "DROP DATABASE IF EXISTS authelia;"

# Recreate the database
kubectl exec -n tools postgres-cluster -- psql -U postgres -c "CREATE DATABASE authelia;"

# Restore the backup
kubectl exec -n tools postgres-cluster -- psql -U postgres -d authelia -f /tmp/backup.sql
```

### Alternative: Restore both databases at once:
```bash
kubectl exec -n tools postgres-cluster -- psql -U postgres -f /tmp/backup.sql
```

This will create both `authelia` and `homebox` databases when your dump includes both.

### Restart the application after restore:
```bash
kubectl rollout restart deployment -n <namespace> <app-name>
```

## Troubleshooting

### "acid.zalan.do/v1: the server could not find the requested resource"

This error means the PostgreSQL operator is not installed. Apply the operator first:
```bash
kubectl apply -k kubernetes/apps/tools/postgresql/operator/
```

Wait for the operator pod to be running:
```bash
kubectl get pods -n tools -l app=postgres-operator
```

### "relation already exists" errors during restore

This happens because the database was already initialized with default tables before the restore. Drop and recreate the database before restoring:
```bash
kubectl exec -n tools postgres-cluster-0 -- psql -U postgres -c "DROP DATABASE IF EXISTS authelia;"
kubectl exec -n tools postgres-cluster-0 -- psql -U postgres -c "CREATE DATABASE authelia;"
kubectl exec -n tools postgres-cluster-0 -- psql -U postgres -d authelia -f /tmp/backup.sql
```

### Password issues after restore

If the application can't connect due to password mismatch, update the password:
```bash
kubectl exec -n tools postgres-cluster -- psql -U postgres -c "ALTER USER authelia_user WITH PASSWORD 'your_new_password';"
```

Or get the password from the Kubernetes secret:
```bash
kubectl get secret -n tools postgres-cluster.credentials -o jsonpath='{.data.authelia_user}' | base64 -d
```

## Backup Configuration

The PostgreSQL cluster is configured with logical backups (see `cluster-zalando.yaml`):

```yaml
enableLogicalBackup: true
logicalBackupRetention: "1 week"
logicalBackupSchedule: "0 2 * * *"  # Daily at 2:00 AM
```

Backups are stored in the S3 bucket configured in the operator's ConfigMap.

## Updating the Cluster

To modify the PostgreSQL cluster configuration, edit `cluster/cluster-zalando.yaml` and apply changes:
```bash
kubectl apply -k kubernetes/apps/tools/postgresql/cluster/
```

## Useful Commands

```bash
# Check cluster status
kubectl get postgresql -n tools

# Describe the PostgreSQL cluster
kubectl describe postgresql -n tools postgres-cluster

# Check operator logs
kubectl logs -n tools -l app=postgres-operator

# Check backup job status
kubectl get jobs -n tools -l application=postgres-cluster
kubectl get cronjobs -n tools -l application=postgres-cluster
```
