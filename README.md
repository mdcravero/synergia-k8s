# Synergia K8s - Home Lab con Flux CD

Este repositorio contiene la configuración de un cluster Kubernetes usando Talos Linux en Raspberry Pi, gestionado con Flux CD.

## Arquitectura

### Orden de Despliegue

Flux CD despliega las aplicaciones en el siguiente orden usando dependencias:

1. **Namespaces** - Crea todos los namespaces necesarios
2. **MetalLB Install** - Instala MetalLB y sus CRDs
3. **Network** - Configuración de MetalLB + Traefik
4. **Storage** - NFS Provisioner
5. **External Secrets** - Sistema de gestión de secretos
6. **Bitwarden** - Backend de secretos
7. **Secrets Config** - Secretos específicos (PostgreSQL, Redis, etc.)
8. **Tools** - PostgreSQL + Redis (con secretos disponibles)
9. **Authentik** - Sistema de autenticación (usa PostgreSQL/Redis)
10. **Apps** - Resto de aplicaciones

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
├── namespaces/           # Todos los namespaces centralizados
├── network/              # MetalLB + Traefik
│   ├── metallb/
│   │   ├── install/      # Instalación de MetalLB
│   │   └── app/          # Configuración de MetalLB
│   └── traefik/app/      # Traefik ingress controller
├── storage/              # NFS provisioner
├── security/             # External Secrets + Bitwarden + Authentik
│   ├── external-secrets/ # Gestión de secretos
│   ├── bitwarden/        # Backend de secretos
│   └── authentik/        # Sistema de autenticación
├── secrets/              # Secretos específicos (SOPS)
├── tools/                # PostgreSQL + Redis
└── declarations/         # Aplicaciones principales
```

## Configuraciones Importantes

### MetalLB
- **Namespace**: `metallb-system` con privilegios elevados
- **FRR deshabilitado**: Solo usa modo L2
- **Pool de IPs**: 192.168.1.50-192.168.1.90

### Namespaces
- **Centralizados**: Todos los namespaces se crean en la primera etapa desde `apps/namespaces/`
- **Privilegios especiales**: `metallb-system` tiene `pod-security.kubernetes.io/enforce: privileged`
- **Sin duplicación**: Ya no es necesario crear `namespace.yaml` en cada aplicación

### Dependencias Críticas
- **External Secrets** → **Bitwarden**: Sistema de gestión de secretos
- **Bitwarden** → **Secrets Config**: Backend de secretos disponible
- **Secrets Config** → **Tools**: Secretos específicos para PostgreSQL/Redis
- **PostgreSQL/Redis** → **Authentik**: Bases de datos requeridas
- **MetalLB** → **Traefik**: LoadBalancer para servicios
- **Namespaces** → **Todo**: Evita errores de "namespace not found"

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

## Notas de Seguridad

- Secretos gestionados con SOPS y External Secrets
- Namespaces con políticas de seguridad apropiadas
- Authentik como punto central de autenticación
- Bitwarden como backend de secretos