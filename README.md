# Red Chart

> Modern Helm chart for deploying web applications with Istio service mesh,
> cert-manager, and Traefik integration. Supports both legacy Istio
> Gateway/VirtualService and Kubernetes Gateway API routing patterns.

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
-   🕸️ **Dual Routing Modes**: Supports both legacy Istio Gateway/VirtualService
    and Kubernetes Gateway API with HTTPRoute
-   🚦 **Traefik Integration**: L4/SNI passthrough routing for HTTPS traffic
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
# Basic installation (legacy routing mode)
helm install my-app red-charts/red-chart \
  --set domain.apex=yourdomain.com \
  --set domain.subdomain=app

# Install with Gateway API routing mode
helm install my-app red-charts/red-chart \
  --set domain.apex=yourdomain.com \
  --set domain.subdomain=app \
  --set routingMode=gatewayapi

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
-   **Istio 1.20+** (with ingress gateway for legacy mode, or Gateway API
    support for gatewayapi mode)
-   **cert-manager 1.12+**
-   **Traefik 2.0+** (as ingress controller)
-   **ClusterIssuer configured** (default: `letsencrypt-prod`)
-   **Gateway API CRDs** (required for `routingMode: gatewayapi`)

## Quick Start Examples

### Deploy with Legacy Routing (Default)

```bash
helm install my-app red-charts/red-chart \
  --set domain.apex=example.com \
  --set domain.subdomain=app
```

This uses the default Istio Gateway + VirtualService routing with an explicit
cert-manager Certificate resource.

### Deploy with Gateway API Routing

```bash
helm install my-app red-charts/red-chart \
  --set domain.apex=example.com \
  --set domain.subdomain=app \
  --set routingMode=gatewayapi \
  --set gateway.name=my-app-gateway \
  --set certificate.enabled=false
```

This uses the Kubernetes Gateway API with cert-manager annotation-based
certificate provisioning on the Gateway resource.

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

This chart supports two routing modes. Both share Traefik at the edge for L4/SNI
passthrough but differ at the service mesh layer.

### Legacy Mode (`routingMode: legacy`)

Uses Istio's own Gateway and VirtualService CRDs. This is the default mode.

```
Client → Traefik (L4/SNI) → Istio Gateway (TLS term) → VirtualService → Service → Pod
```

**Resources created:** Istio Gateway, VirtualService, cert-manager Certificate,
Traefik IngressRouteTCP (targets `istio-ingressgateway`)

### Gateway API Mode (`routingMode: gatewayapi`)

Uses the standard Kubernetes Gateway API with Istio as the implementation.

```
Client → Traefik (L4/SNI) → K8s Gateway API Gateway (TLS term) → HTTPRoute → Service → Pod
```

**Resources created:** Gateway (`gateway.networking.k8s.io/v1`), HTTPRoute,
HTTPRoute (HTTP→HTTPS redirect), ReferenceGrant, Traefik IngressRouteTCP
(targets `<gateway-name>-istio`)

Certificate is provisioned via the `cert-manager.io/cluster-issuer` Gateway
annotation by default, or as an explicit Certificate resource if
`certificate.enabled=true`.

### Detailed Traffic Flow

```
┌─────────────────────────────────────────────────────┐
│                     Internet                         │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
            ┌─────────────────┐
            │     Traefik     │  (Layer 4: TCP/SNI Passthrough)
            │   Edge Proxy    │
            └────────┬────────┘
                     │
        ┌────────────┴────────────┐
        │                         │
        ▼ (legacy)                ▼ (gatewayapi)
┌───────────────┐        ┌────────────────┐
│ Istio Gateway │        │ K8s Gateway    │
│ (istio CRD)   │        │ (gateway API)  │
│ TLS Term      │        │ TLS Term       │
└───────┬───────┘        └───────┬────────┘
        │                        │
        ▼                        ▼
┌───────────────┐        ┌────────────────┐
│VirtualService │        │   HTTPRoute    │
│ (HTTP routing)│        │ (HTTP routing) │
└───────┬───────┘        └───────┬────────┘
        │                        │
        └────────────┬───────────┘
                     │
                     ▼
            ┌─────────────────┐
            │  K8s Service    │  (Load Balancing)
            └────────┬────────┘
                     │
                     ▼
            ┌─────────────────┐
            │   Your App      │  (Application Pods)
            │    (Pods)       │
            └─────────────────┘
```

## Configuration

### Core Parameters

| Parameter          | Description                    | Default                       |
| ------------------ | ------------------------------ | ----------------------------- |
| `app.name`         | Application name               | `red-app`                     |
| `app.version`      | Application version            | `1.0.0`                       |
| `routingMode`      | Routing pattern to use         | `legacy`                      |
| `namespace.name`   | Kubernetes namespace           | `red-app`                     |
| `namespace.create` | Create namespace if not exists | `true`                        |
| `domain.apex`      | Apex domain name               | `rag-space.com`               |
| `domain.subdomain` | Application subdomain          | `red-app`                     |
| `image.repository` | Container image                | `nginxinc/nginx-unprivileged` |
| `image.tag`        | Image tag                      | `alpine3.21`                  |
| `image.pullPolicy` | Image pull policy              | `IfNotPresent`                |
| `imagePullSecrets` | Image pull secrets             | `[]`                          |
| `replicaCount`     | Number of replicas             | `1`                           |

### Routing Mode

| Value        | Description                                                              |
| ------------ | ------------------------------------------------------------------------ |
| `legacy`     | Istio Gateway + VirtualService (default, targets `istio-ingressgateway`) |
| `gatewayapi` | K8s Gateway API + HTTPRoute (targets `<gateway-name>-istio`)             |

### Content Configuration

| Parameter      | Description                       | Default              |
| -------------- | --------------------------------- | -------------------- |
| `content.html` | Static HTML content via ConfigMap | Default welcome page |

**Note:** If `content.html` is not set (or set to empty string), the chart will
use the content from your container image. This is ideal for deploying
containerized applications.

### Gateway & Certificates

| Parameter              | Description                                | Default            |
| ---------------------- | ------------------------------------------ | ------------------ |
| `gateway.name`         | Gateway resource name                      | `red-app-gateway`  |
| `gateway.namespace`    | Gateway namespace                          | `istio-system`     |
| `gateway.className`    | GatewayClass name (gatewayapi mode only)   | `istio`            |
| `gateway.annotations`  | Gateway annotations (gatewayapi mode only) | see values.yaml    |
| `certificate.enabled`  | Create explicit Certificate resource       | `true`             |
| `certificate.issuer`   | ClusterIssuer name                         | `letsencrypt-prod` |
| `certificate.duration` | Certificate validity                       | `2160h` (90 days)  |
| `tcpRoute.enabled`     | Enable Traefik TCP route                   | `true`             |

#### Certificate Behavior by Mode

| Mode         | `certificate.enabled` | Behavior                                                        |
| ------------ | --------------------- | --------------------------------------------------------------- |
| `legacy`     | `true` (required)     | Creates an explicit cert-manager Certificate resource           |
| `gatewayapi` | `true`                | Creates an explicit Certificate; no annotation on Gateway       |
| `gatewayapi` | `false`               | No Certificate resource; Gateway annotation provisions the cert |

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

## Example Values Files

### Legacy Mode (Istio Gateway + VirtualService)

```yaml
# values-legacy.yaml
routingMode: legacy

app:
    name: red-app

namespace:
    name: red-app
    create: true

domain:
    apex: rag-space.com
    subdomain: red-app

gateway:
    name: red-app-gateway
    namespace: istio-system

certificate:
    enabled: true
    issuer: letsencrypt-prod
```

### Gateway API Mode (K8s Gateway + HTTPRoute)

```yaml
# values-gatewayapi.yaml
routingMode: gatewayapi

app:
    name: red-test

namespace:
    name: red-test
    create: true

domain:
    apex: rag-space.com
    subdomain: red-test

gateway:
    name: red-test-gateway
    namespace: istio-system
    className: istio
    annotations:
        cert-manager.io/cluster-issuer: letsencrypt-prod
        networking.istio.io/service-type: ClusterIP

certificate:
    enabled: false
```

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
# Development (Gateway API)
helm install my-app-dev red-charts/red-chart \
  -f values-dev.yaml \
  --set routingMode=gatewayapi \
  --namespace dev \
  --create-namespace

# Staging (Gateway API)
helm install my-app-staging red-charts/red-chart \
  -f values-staging.yaml \
  --set routingMode=gatewayapi \
  --namespace staging \
  --create-namespace

# Production (Legacy — proven stable)
helm install my-app-prod red-charts/red-chart \
  -f values-production.yaml \
  --set routingMode=legacy \
  --namespace production \
  --create-namespace
```

### Migrating from Legacy to Gateway API

To migrate an existing deployment from legacy to Gateway API mode:

1. Deploy a test instance with `routingMode: gatewayapi` on a separate subdomain
   to validate the Gateway API flow works in your cluster.
2. Once validated, update your production values to `routingMode: gatewayapi`.
3. Run `helm upgrade` — this will replace the Istio Gateway and VirtualService
   with the Gateway API Gateway, HTTPRoute, and ReferenceGrant resources.
4. The Traefik IngressRouteTCP will automatically update its target from
   `istio-ingressgateway` to `<gateway-name>-istio`.

```bash
helm upgrade my-app red-charts/red-chart \
  -f values.yaml \
  --set routingMode=gatewayapi \
  --set certificate.enabled=false
```

**Note:** During the upgrade there will be a brief period where the old routing
resources are removed and the new ones are created. Plan for a short window of
downtime or perform the migration during a maintenance window.

### Upgrading

```bash
# Update repository
helm repo update

# Check for new versions
helm search repo red-charts/red-chart --versions

# Upgrade to latest version
helm upgrade my-app red-charts/red-chart

# Upgrade to specific version
helm upgrade my-app red-charts/red-chart --version 0.2.1

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
# Check certificate status (legacy mode or gatewayapi with certificate.enabled=true)
kubectl get certificate -n istio-system
kubectl describe certificate <release-name>-cert -n istio-system

# Check if annotation-based provisioning worked (gatewayapi mode)
kubectl get secret -n istio-system <release-name>-cert-tls

# Check cert-manager logs
kubectl logs -n cert-manager deployment/cert-manager

# Verify ClusterIssuer
kubectl get clusterissuer letsencrypt-prod
kubectl describe clusterissuer letsencrypt-prod
```

### Gateway Not Working (Legacy Mode)

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

### Gateway Not Working (Gateway API Mode)

```bash
# Check Gateway API Gateway
kubectl get gateway.gateway.networking.k8s.io -n istio-system
kubectl describe gateway.gateway.networking.k8s.io <gateway-name> -n istio-system

# Check the auto-created gateway service
kubectl get svc -n istio-system <gateway-name>-istio

# Check HTTPRoutes
kubectl get httproute -n <namespace>
kubectl describe httproute <route-name> -n <namespace>

# Check ReferenceGrant
kubectl get referencegrant -n istio-system

# Verify Gateway API CRDs are installed
kubectl get crd gateways.gateway.networking.k8s.io
kubectl get crd httproutes.gateway.networking.k8s.io
```

### Traefik Routing Issues

```bash
# Check IngressRouteTCP
kubectl get ingressroutetcp -n istio-system
kubectl describe ingressroutetcp <route-name> -n istio-system

# Verify the TCP route target service exists
# Legacy: istio-ingressgateway
# Gateway API: <gateway-name>-istio
kubectl get svc -n istio-system

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

## Resources Created by Mode

### Legacy Mode

| Resource        | API Group           | Namespace     | Purpose                  |
| --------------- | ------------------- | ------------- | ------------------------ |
| Namespace       | v1                  | —             | Application namespace    |
| Deployment      | apps/v1             | app namespace | Application pods         |
| Service         | v1                  | app namespace | ClusterIP load balancing |
| ConfigMap       | v1                  | app namespace | Static HTML content      |
| Gateway         | networking.istio.io | istio-system  | TLS termination          |
| VirtualService  | networking.istio.io | app namespace | HTTP routing             |
| Certificate     | cert-manager.io     | istio-system  | TLS certificate          |
| IngressRouteTCP | traefik.io          | istio-system  | L4/SNI passthrough       |

### Gateway API Mode

| Resource        | API Group                 | Namespace     | Purpose                    |
| --------------- | ------------------------- | ------------- | -------------------------- |
| Namespace       | v1                        | —             | Application namespace      |
| Deployment      | apps/v1                   | app namespace | Application pods           |
| Service         | v1                        | app namespace | ClusterIP load balancing   |
| ConfigMap       | v1                        | app namespace | Static HTML content        |
| Gateway         | gateway.networking.k8s.io | istio-system  | TLS termination            |
| HTTPRoute       | gateway.networking.k8s.io | app namespace | HTTP routing               |
| HTTPRoute       | gateway.networking.k8s.io | app namespace | HTTP→HTTPS redirect        |
| ReferenceGrant  | gateway.networking.k8s.io | istio-system  | Cross-namespace references |
| Certificate     | cert-manager.io           | istio-system  | TLS cert (if enabled)      |
| IngressRouteTCP | traefik.io                | istio-system  | L4/SNI passthrough         |

## Chart Versions

To see all available versions:

```bash
helm search repo red-charts/red-chart --versions
```

| Version | Changes                                         |
| ------- | ----------------------------------------------- |
| 0.2.1   | Dual routing mode support (legacy + gatewayapi) |
| 0.1.3   | Initial release with legacy Istio routing       |

## Development

### Local Development

```bash
# Clone the repository
git clone https://github.com/RussellGilmore/red-chart.git
cd red-chart

# Lint both routing modes
helm lint charts/red-chart -f charts/red-chart/ci/legacy-values.yaml
helm lint charts/red-chart -f charts/red-chart/ci/gatewayapi-values.yaml

# Dry-run template for legacy mode
helm template test charts/red-chart -f charts/red-chart/ci/legacy-values.yaml

# Dry-run template for Gateway API mode
helm template test charts/red-chart -f charts/red-chart/ci/gatewayapi-values.yaml

# Verify resource isolation between modes
helm template test charts/red-chart --set routingMode=legacy | grep "^kind:" | sort
helm template test charts/red-chart --set routingMode=gatewayapi | grep "^kind:" | sort
```

### CI Test Values

The `charts/red-chart/ci/` directory contains values files used for lint
testing:

-   `legacy-values.yaml` — validates legacy Istio routing mode
-   `gatewayapi-values.yaml` — validates Kubernetes Gateway API routing mode

### Making Changes

1. Make your changes to the chart
2. Bump the version in `Chart.yaml`
3. Lint and template both routing modes
4. Commit and push to `main` branch
5. GitHub Actions will automatically package and publish the new version

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
