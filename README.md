# Red Chart

> Modern Helm chart for deploying web applications with Istio service mesh,
> cert-manager, and Traefik integration.

[![Helm](https://img.shields.io/badge/Helm-v3.8%2B-blue)](https://helm.sh)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.24%2B-blue)](https://kubernetes.io)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

## Overview

Red Chart provides a production-ready Helm chart pattern for deploying web
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

## Installation

### Add the Helm Repository

```bash
helm repo add red-charts https://russellgilmore.github.io/red-chart
helm repo update
```

### Search for Available Charts

```bash
helm search repo red-charts
```

### Install the Chart

```bash
# Basic installation
helm install my-app red-charts/red-chart \
  --set domain.apex=yourdomain.com \
  --set domain.subdomain=app

# Install with custom values file
helm install my-app red-charts/red-chart -f values.yaml
```

### Pull the Chart Locally

```bash
# Download the chart package
helm pull red-charts/red-chart

# Download and extract
helm pull red-charts/red-chart --untar
```

## Prerequisites

Before using Red Chart, ensure your cluster has:

-   **Kubernetes 1.24+**
-   **Helm 3.8+**
-   **Istio 1.20+** (with ingress gateway)
-   **cert-manager 1.12+**
-   **Traefik 2.0+** (as ingress controller)
-   **ClusterIssuer configured** (default: `letsencrypt-prod`)

## Quick Start Examples

### Deploy a Simple Web App

```bash
helm install my-app red-charts/red-chart \
  --set domain.apex=example.com \
  --set domain.subdomain=app \
  --set image.repository=nginx \
  --set image.tag=alpine
```

### Deploy with Custom HTML Content

Create a `values.yaml`:

```yaml
domain:
    apex: example.com
    subdomain: myapp

content:
    html: |
        <!DOCTYPE html>
        <html>
        <head><title>My App</title></head>
        <body><h1>Hello World!</h1></body>
        </html>
```

Install:

```bash
helm install my-app red-charts/red-chart -f values.yaml
```

### Deploy a Container Image from Private Registry

```yaml
image:
    repository: registry.example.com/my-app
    tag: latest
    pullPolicy: Always

imagePullSecrets:
    - name: my-registry-secret

domain:
    apex: example.com
    subdomain: app
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
            │   Your App      │  (Pods)
            └─────────────────┘
```

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

For a complete list of configuration options, see [values.yaml](values.yaml).

## Advanced Usage

### Production Configuration

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

Deploy:

```bash
helm install my-app red-charts/red-chart -f values-production.yaml
```

### Multi-Environment Deployments

```bash
# Development
helm install my-app-dev red-charts/red-chart \
  -f values-dev.yaml \
  --namespace dev

# Staging
helm install my-app-staging red-charts/red-chart \
  -f values-staging.yaml \
  --namespace staging

# Production
helm install my-app-prod red-charts/red-chart \
  -f values-production.yaml \
  --namespace production
```

### Upgrading

```bash
# Update repository
helm repo update

# Check for new versions
helm search repo red-charts/red-chart --versions

# Upgrade to latest version
helm upgrade my-app red-charts/red-chart

# Upgrade to specific version
helm upgrade my-app red-charts/red-chart --version 0.1.2
```

### Uninstalling

```bash
helm uninstall my-app
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

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file
for details.

## Maintainers

-   Russell Gilmore - [Russell.Gilmore@pm.me](mailto:Russell.Gilmore@pm.me)

## Links

-   [Chart Repository](https://russellgilmore.github.io/red-chart)
-   [Source Code](https://github.com/RussellGilmore/red-chart)
-   [Issues](https://github.com/RussellGilmore/red-chart/issues)
