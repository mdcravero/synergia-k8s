# Synergia K8s - Home Lab con Flux CD

Este repositorio contiene la configuración de un cluster Kubernetes usando Talos Linux en Raspberry Pi, gestionado con Flux CD.

## Arquitectura

### Orden de Despliegue

Flux CD despliega las aplicaciones en el siguiente orden usando dependencias:

1. **MetalLB Install** - Instala MetalLB y sus CRDs
2. **Network** - Configuración de MetalLB + Traefik
3. **Storage** - NFS Provisioner
4. **Tools** - PostgreSQL + Redis (dependencias de Authentik)
5. **Security** - External Secrets + Bitwarden + Authentik
6. **Apps** - Resto de aplicaciones

### Componentes Principales

- **MetalLB**: Load balancer para bare metal (rango IP: 192.168.1.50-192.168.1.90)
- **Traefik**: Ingress controller y reverse proxy
- **NFS Provisioner**: Storage dinámico usando NFS (servidor: 192.168.1.10)
- **Authentik**: Sistema de autenticación y autorización
- **External Secrets**: Gestión de secretos desde Bitwarden

## Estructura del Proyecto

```
bootstrap/kubernetes/apps/
├── flux-system/          # Configuración de Flux CD
├── network/              # MetalLB + Traefik
│   ├── metallb/
│   │   ├── install/      # Instalación de MetalLB
│   │   └── app/          # Configuración de MetalLB
│   └── traefik/app/      # Traefik ingress controller
├── storage/              # NFS provisioner
├── tools/                # PostgreSQL + Redis
├── security/             # Authentik + External Secrets
└── declarations/         # Aplicaciones principales
```

## Configuraciones Importantes

### MetalLB
- **Namespace**: `metallb-system` con privilegios elevados
- **FRR deshabilitado**: Solo usa modo L2
- **Pool de IPs**: 192.168.1.50-192.168.1.90

### Namespaces
Cada aplicación debe tener su archivo `namespace.yaml` como primer recurso en `kustomization.yaml` para evitar errores de "namespace not found".

### Dependencias Críticas
- **PostgreSQL/Redis** → **Authentik**: Bases de datos requeridas
- **External Secrets** → **Authentik**: Secretos necesarios para configuración
- **MetalLB** → **Traefik**: LoadBalancer para servicios

## Solución de Problemas Comunes

### Error: "namespace not found"
**Solución**: Agregar `namespace.yaml` como primer recurso en `kustomization.yaml`

### Error: "metallb.io/v1beta1: resource not found"
**Solución**: Separar instalación de MetalLB de su configuración usando dependencias de Flux

### Error: "violates PodSecurity baseline"
**Solución**: Configurar namespace con `pod-security.kubernetes.io/enforce: privileged`

### Error: "FRR startup probe failed"
**Solución**: Deshabilitar FRR en MetalLB usando:
```yaml
values:
  frrk8s:
    enabled: false
  speaker:
    frr:
      enabled: false
```

## Comandos Útiles

```bash
# Ver estado de Flux
flux get kustomizations

# Ver logs de reconciliación
kubectl logs -n flux-system deployment/kustomize-controller

# Forzar reconciliación
flux reconcile kustomization apps-network

# Ver recursos de MetalLB
kubectl get pods -n metallb-system
kubectl get ipaddresspool -n metallb-system
```

## Configuración de Red

- **Cluster**: Talos Linux en Raspberry Pi
- **MetalLB Pool**: 192.168.1.50-192.168.1.90
- **NFS Server**: 192.168.1.10:/srv/nfs4/k8s
- **Repositorio Git**: https://github.com/mdcravero/synergia-k8s.git

## Notas de Seguridad

- Secretos gestionados con SOPS y External Secrets
- Namespaces con políticas de seguridad apropiadas
- Authentik como punto central de autenticación
- Bitwarden como backend de secretos