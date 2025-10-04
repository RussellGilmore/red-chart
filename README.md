# Red Charts

> Modern Helm charts for deploying web applications with Istio service mesh,
> cert-manager, and Traefik integration.

[![Helm](https://img.shields.io/badge/Helm-v3.8%2B-blue)](https://helm.sh)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.24%2B-blue)](https://kubernetes.io)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

## Overview

Red Charts provides a production-ready Helm chart pattern for deploying web
applications on Kubernetes with a modern service mesh architecture. The chart
handles all the complexity of integrating Istio, cert-manager, and Traefik,
allowing you to deploy secure, TLS-enabled applications with a single command.

### Key Features

-   🔒 **Automatic TLS**: Integration with cert-manager for Let's Encrypt
    certificates
-   🕸️ **Service Mesh Ready**: Native Istio support with Gateway and
    VirtualService
-   🚦 **Traefik Integration**: TCP routing for HTTPS traffic
-   🛡️ **Security First**: Pod Security Standards compliant, runs as non-root
-   📦 **Single Chart, Multiple Apps**: Deploy different applications using the
    same pattern
-   ⚡ **Production Ready**: Health probes, resource limits, and security
    contexts included

## Prerequisites

Before using Red Charts, ensure your cluster has:

-   **Kubernetes 1.24+**
-   **Helm 3.8+**
-   **Istio 1.20+** (with ingress gateway)
-   **cert-manager 1.12+**
-   **Traefik 2.0+** (as ingress controller)
-   **ClusterIssuer configured** (default: `letsencrypt-prod`)

## Quick Start

### Installation

```bash
# Clone the repository
git clone https://github.com/RussellGilmore/red-charts.git
cd red-charts

# Install with default values
helm install my-app charts/red-chart \
  --set domain.apex=yourdomain.com \
  --set domain.subdomain=app

# Install with custom values file
helm install red-app charts/red-chart -f examples/red-app-values.yaml
```

### Example Deployments

#### Deploy Red App

```bash
helm install red-app charts/red-chart \
  --values examples/red-app-values.yaml \
  --set domain.apex=yourdomain.com
```

#### Deploy Red Cards

```bash
helm install red-cards charts/red-chart \
  --values examples/red-cards-values.yaml \
  --set domain.apex=yourdomain.com
```

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                     Internet                         │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
            ┌─────────────────┐
            │     Traefik     │  (TCP Route with SNI)
            │  Ingress / LB   │
            └────────┬────────┘
                     │
                     ▼
            ┌─────────────────┐
            │  Istio Gateway  │  (TLS Termination)
            │  (istio-system) │
            └────────┬────────┘
                     │
                     ▼
            ┌─────────────────┐
            │ VirtualService  │  (HTTP Routing)
            └────────┬────────┘
                     │
                     ▼
            ┌─────────────────┐
            │ K8s Service     │
            └────────┬────────┘
                     │
                     ▼
            ┌─────────────────┐
            │   Nginx Pod     │  (Application)
            │  (ConfigMap)    │
            └─────────────────┘
```

### Component Flow

1. **Client Request** → Traefik (TCP Route with SNI matching)
2. **Traefik** → Istio Gateway (TLS termination using cert-manager certificate)
3. **Gateway** → VirtualService (HTTP routing rules)
4. **VirtualService** → Kubernetes Service (load balancing)
5. **Service** → Nginx Pods (serving content from ConfigMap)

## Configuration

### Core Parameters

| Parameter          | Description                    | Default                       |
| ------------------ | ------------------------------ | ----------------------------- |
| `app.name`         | Application name               | `red-app`                     |
| `app.version`      | Application version            | `1.0.0`                       |
| `namespace.name`   | Kubernetes namespace           | `red-app`                     |
| `namespace.create` | Create namespace if not exists | `true`                        |
| `domain.apex`      | Apex domain name               | `rag-space.com`               |
| `domain.subdomain` | Application subdomain          | `red-app`                     |
| `image.repository` | Container image                | `nginxinc/nginx-unprivileged` |
| `image.tag`        | Image tag                      | `alpine3.21`                  |
| `replicaCount`     | Number of replicas             | `1`                           |

### Gateway & Certificates

| Parameter              | Description          | Default            |
| ---------------------- | -------------------- | ------------------ |
| `gateway.name`         | Istio Gateway name   | `red-app-gateway`  |
| `gateway.namespace`    | Gateway namespace    | `istio-system`     |
| `certificate.enabled`  | Enable cert-manager  | `true`             |
| `certificate.issuer`   | ClusterIssuer name   | `letsencrypt-prod` |
| `certificate.duration` | Certificate validity | `2160h` (90 days)  |
| `tcpRoute.enabled`     | Enable Traefik route | `true`             |

### Resources & Security

| Parameter                            | Description          | Default |
| ------------------------------------ | -------------------- | ------- |
| `resources.requests.memory`          | Memory request       | `64Mi`  |
| `resources.requests.cpu`             | CPU request          | `100m`  |
| `resources.limits.memory`            | Memory limit         | `128Mi` |
| `resources.limits.cpu`               | CPU limit            | `200m`  |
| `containerSecurityContext.runAsUser` | Container UID        | `101`   |
| `podSecurityContext.fsGroup`         | Volume ownership GID | `101`   |

### Custom HTML Content

Customize your application content through `values.yaml`:

```yaml
content:
    html: |
        <!DOCTYPE html>
        <html>
        <head>
          <title>My Custom App</title>
        </head>
        <body>
          <h1>Welcome to {{ .Values.app.name }}!</h1>
          <p>Custom content here...</p>
        </body>
        </html>
```

## Advanced Usage

### Production Configuration

Create a production values file:

```yaml
# values-production.yaml
replicaCount: 3

resources:
    requests:
        memory: 128Mi
        cpu: 200m
    limits:
        memory: 256Mi
        cpu: 500m

affinity:
    podAntiAffinity:
        preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                  labelSelector:
                      matchExpressions:
                          - key: app.kubernetes.io/name
                            operator: In
                            values:
                                - red-chart
                  topologyKey: kubernetes.io/hostname
```

Deploy with:

```bash
helm install my-app charts/red-chart \
  -f values-production.yaml \
  --set domain.apex=production.com \
  --set domain.subdomain=app
```

### Multi-Environment Deployments

```bash
# Development
helm install red-app-dev charts/red-chart \
  -f values-dev.yaml \
  --namespace dev

# Staging
helm install red-app-staging charts/red-chart \
  -f values-staging.yaml \
  --namespace staging

# Production
helm install red-app-prod charts/red-chart \
  -f values-production.yaml \
  --namespace production
```

## Troubleshooting

### Certificate Not Issued

```bash
# Check certificate status
kubectl get certificate -n istio-system
kubectl describe certificate <release-name>-cert -n istio-system

# Check cert-manager logs
kubectl logs -n cert-manager deployment/cert-manager

# Verify ClusterIssuer
kubectl get clusterissuer letsencrypt-prod
kubectl describe clusterissuer letsencrypt-prod
```

### Gateway Not Working

```bash
# Check Istio gateway
kubectl get gateway -n istio-system
kubectl describe gateway <gateway-name> -n istio-system

# Verify Istio ingress gateway
kubectl get svc -n istio-system istio-ingressgateway
kubectl get pods -n istio-system -l istio=ingressgateway

# Check VirtualService
kubectl get virtualservice -n <namespace>
kubectl describe virtualservice <vs-name> -n <namespace>
```

### Traefik Routing Issues

```bash
# Check IngressRouteTCP
kubectl get ingressroutetcp -n istio-system
kubectl describe ingressroutetcp <route-name> -n istio-system

# Verify Traefik logs
kubectl logs -n traefik deployment/traefik
```

### Application Not Starting

```bash
# Check pod status
kubectl get pods -n <namespace>
kubectl describe pod <pod-name> -n <namespace>

# View pod logs
kubectl logs -n <namespace> <pod-name>

# Check events
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

## Security

### Security Features

-   ✅ **Non-root containers**: Runs as UID 101
-   ✅ **Read-only root filesystem**: Immutable container
-   ✅ **Dropped capabilities**: All Linux capabilities removed
-   ✅ **Seccomp profile**: RuntimeDefault enforced
-   ✅ **Resource limits**: Memory and CPU constrained
-   ✅ **Security contexts**: Pod and container level
-   ✅ **TLS encryption**: Automatic HTTPS with valid certificates
-   ✅ **Pod Security Standards**: Restricted policy compliant
