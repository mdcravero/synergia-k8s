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
8. **Monitoring** - Grafana + Loki + Prometheus (requiere secrets)
9. **Authentik** - Sistema de autenticación (requiere secrets)
10. **Tools** - PostgreSQL + Redis + Homarr (Homarr espera a Authentik)
11. **Media** - Jellyfin, Radarr, Sonarr, etc. (usa Authentik para autenticación)

### Componentes Principales

- **MetalLB**: Load balancer para bare metal (rango IP: 10.42.20.50-10.42.20.90)
- **Traefik**: Ingress controller y reverse proxy con middlewares de Authentik
- **NFS Provisioner**: Storage dinámico usando NFS (servidor: 10.42.20.10)
- **Monitoring**: Grafana + Loki + Prometheus para observabilidad
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
├── monitoring/           # Grafana + Loki + Prometheus
├── secrets/              # Secretos específicos (SOPS)
├── tools/                # PostgreSQL + Redis
└── declarations/         # Aplicaciones de media (Jellyfin, *arr, etc.)
```

## Configuraciones Importantes

### MetalLB
- **Namespace**: `metallb-system` con privilegios elevados
- **FRR deshabilitado**: Solo usa modo L2
- **Pool de IPs**: 10.42.20.40-10.42.20.90

### Namespaces
- **Centralizados**: Todos los namespaces se crean en la primera etapa desde `apps/namespaces/`
- **Privilegios especiales**: `metallb-system` tiene `pod-security.kubernetes.io/enforce: privileged`
- **Sin duplicación**: Ya no es necesario crear `namespace.yaml` en cada aplicación

### Servicios y IPs
- **Traefik**: 10.42.20.50
- **Grafana**: 10.42.20.51 (monitoring)
- **Authentik**: 10.42.20.72
- **PostgreSQL**: 10.42.20.70
- **Redis**: 10.42.20.71
- **Jellyfin**: 10.42.20.60
- **Jellyseerr**: 10.42.20.61
- **Prowlarr**: 10.42.20.62
- **Radarr**: 10.42.20.63
- **Sonarr**: 10.42.20.64
- **Bazarr**: 10.42.20.67
- **Transmission Movies**: 10.42.20.69
- **Transmission Shows**: 10.42.20.68

### Dependencias Críticas
- **External Secrets** → **Monitoring**: Grafana requiere secretos para datasources
- **External Secrets** → **Bitwarden**: Sistema de gestión de secretos
- **Bitwarden** → **Secrets Config**: Backend de secretos disponible
- **Secrets Config** → **Tools**: Secretos específicos para PostgreSQL/Redis
- **PostgreSQL/Redis** → **Authentik**: Bases de datos requeridas
- **Authentik** → **Media**: Aplicaciones usan middlewares de autenticación
- **MetalLB** → **Traefik**: LoadBalancer para servicios
- **Namespaces** → **Todo**: Evita errores de "namespace not found"

## Comandos Útiles

```bash
# Ver estado de Flux
flux get kustomizations

# Ver logs de reconciliación
kubectl logs -n flux-system deployment/kustomize-controller

# Forzar reconciliación
flux reconcile kustomization apps-monitoring
flux reconcile kustomization apps

# Ver recursos de MetalLB
kubectl get pods -n metallb-system
kubectl get ipaddresspool -n metallb-system

# Ver servicios con IPs asignadas
kubectl get svc -A | grep LoadBalancer

# Acceder a Grafana (usuario/admin, contraseña de secret)
kubectl port-forward -n monitoring svc/grafana 3000:80
```

## Notas de Seguridad

- Secretos gestionados con SOPS y External Secrets
- Namespaces con políticas de seguridad apropiadas
- Authentik como punto central de autenticación
- Bitwarden como backend de secretos