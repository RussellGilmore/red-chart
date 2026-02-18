# Red Chart

> Modern Helm chart for deploying web applications with Istio service mesh,
> cert-manager, and Traefik integration.

[![Helm](https://img.shields.io/badge/Helm-v3.8%2B-blue)](https://helm.sh)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.24%2B-blue)](https://kubernetes.io)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Release Charts](https://github.com/RussellGilmore/red-chart/actions/workflows/release-charts.yaml/badge.svg)](https://github.com/RussellGilmore/red-chart/actions/workflows/release-charts.yaml)

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
-   📦 **Container Ready**: Supports both static HTML via ConfigMap and
    containerized applications
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

### Deploy a Simple Static Web App

```bash
helm install my-app red-charts/red-chart \
  --set domain.apex=example.com \
  --set domain.subdomain=app
```

This uses the default static HTML content from the chart.

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

### Deploy a Containerized Application

For applications built as container images (like React apps):

```yaml
app:
    name: my-react-app

image:
    repository: registry.example.com/my-app
    tag: v1.0.0
    pullPolicy: Always

domain:
    apex: example.com
    subdomain: app
# Don't set content.html - use the app from the container
```

### Deploy from Private Registry

Create image pull secret first:

```bash
kubectl create secret docker-registry my-registry-secret \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass \
  --namespace=my-namespace
```

Then in `values.yaml`:

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

This chart uses **Istio Gateway and VirtualService** for routing.

```
┌─────────────────────────────────────────────────────┐
│                     Internet                         │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
            ┌─────────────────┐
            │     Traefik     │  (Layer 4: TCP/SNI)
            │  Ingress / LB   │
            └────────┬────────┘
                     │
                     ▼
            ┌─────────────────┐
            │  Istio Gateway  │  (Layer 7: TLS Termination)
            │  (istio-system) │  (cert-manager certificate)
            └────────┬────────┘
                     │
                     ▼
            ┌─────────────────┐
            │ VirtualService  │  (HTTP Routing Rules)
            └────────┬────────┘
                     │
                     ▼
            ┌─────────────────┐
            │ K8s Service     │  (Load Balancing)
            └────────┬────────┘
                     │
                     ▼
            ┌─────────────────┐
            │   Your App      │  (Application Pods)
            │    (Pods)       │
            └─────────────────┘
```

**Traffic Flow:**

1. Client makes HTTPS request → Traefik (SNI passthrough on port 443)
2. Traefik → Istio Gateway (TLS termination using cert-manager certificate)
3. Istio Gateway → VirtualService (HTTP routing based on hostname/path)
4. VirtualService → Kubernetes Service (ClusterIP)
5. Service → Application Pods (your containerized app or static content)

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
| `image.pullPolicy` | Image pull policy              | `IfNotPresent`                |
| `imagePullSecrets` | Image pull secrets             | `[]`                          |
| `replicaCount`     | Number of replicas             | `1`                           |

### Content Configuration

| Parameter      | Description                       | Default              |
| -------------- | --------------------------------- | -------------------- |
| `content.html` | Static HTML content via ConfigMap | Default welcome page |

**Note:** If `content.html` is not set (or set to empty string), the chart will
use the content from your container image. This is ideal for deploying
containerized applications.

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
  --namespace dev \
  --create-namespace

# Staging
helm install my-app-staging red-charts/red-chart \
  -f values-staging.yaml \
  --namespace staging \
  --create-namespace

# Production
helm install my-app-prod red-charts/red-chart \
  -f values-production.yaml \
  --namespace production \
  --create-namespace
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

# Upgrade with new values
helm upgrade my-app red-charts/red-chart -f values.yaml
```

### Uninstalling

```bash
# Uninstall the release
helm uninstall my-app

# Uninstall and delete namespace
helm uninstall my-app --namespace my-namespace
kubectl delete namespace my-namespace
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

## Chart Versions

To see all available versions:

```bash
helm search repo red-charts/red-chart --versions
```

## Development

### Local Development

```bash
# Clone the repository
git clone https://github.com/RussellGilmore/red-chart.git
cd red-chart

# Test the chart
helm lint .

# Dry-run install
helm install test-release . --dry-run --debug

# Install locally
helm install test-release .
```

### Making Changes

1. Make your changes to the chart
2. Bump the version in `Chart.yaml`
3. Commit and push to `main` branch
4. GitHub Actions will automatically package and publish the new version

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add some amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file
for details.

## Maintainers

-   Russell Gilmore - [ragilmore775@gmail.com](mailto:ragilmore775@gmail.com)

## Links

-   [Chart Repository](https://russellgilmore.github.io/red-chart)
-   [Source Code](https://github.com/RussellGilmore/red-chart)
-   [Issues](https://github.com/RussellGilmore/red-chart/issues)
-   [Releases](https://github.com/RussellGilmore/red-chart/releases)
