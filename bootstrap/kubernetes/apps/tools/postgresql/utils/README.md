# Spilo PostgreSQL Custom Image

## ¿Qué hace esta imagen?

Imagen personalizada de Spilo 17 con permisos ajustados (1000:1000) para compatibilidad con el cluster Kubernetes.

### Problemas que resuelve:
- ✅ Permisos UID/GID 1000:1000 para usuarios no-root
- ✅ Compatibilidad con ARM64 
- ✅ Incluye herramientas de backup (pg_dumpall, psql, pigz)
- ✅ Soporte para backups a AWS S3

## Comandos de Build y Deploy

### Construir la imagen
```bash
docker build -f Dockerfile.spilo -t mdcravero/spilo-17:4.0-p3-uid1000-arm64-v2 .
```

### Subir al registry
```bash
docker push mdcravero/spilo-17:4.0-p3-uid1000-arm64-v2
```

### Usar en el cluster
Actualizar `cluster-zalando.yaml`:
```yaml
spec:
  dockerImage: mdcravero/spilo-17:4.0-p3-uid1000-arm64-v2
  spiloRunAsUser: 1000
  spiloRunAsGroup: 1000
  spiloFSGroup: 1000
```

## Variables de Backup S3
```yaml
env:
  - name: AWS_ACCESS_KEY_ID
    valueFrom:
      secretKeyRef:
        name: secret-zalando-s3-bkp
        key: access-key-id
  - name: AWS_SECRET_ACCESS_KEY
    valueFrom:
      secretKeyRef:
        name: secret-zalando-s3-bkp
        key: secret-access-key
  - name: WAL_S3_BUCKET
    valueFrom:
      secretKeyRef:
        name: secret-zalando-s3-bkp
        key: bucket-name
```

## Habilitar Backups Lógicos
```yaml
enableLogicalBackup: true
logicalBackupRetention: "1 week"
logicalBackupSchedule: "0 2 * * *"