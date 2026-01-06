# RustDesk Server Helm Chart

[![Artifact Hub](https://img.shields.io/endpoint?url=https://artifacthub.io/badge/repository/rustdesk-helm-charts)](https://artifacthub.io/packages/search?repo=rustdesk-helm-charts)

A Helm chart for deploying RustDesk self-hosted server on Kubernetes with arm64 support.

## Overview

This chart deploys both required RustDesk components:
- **hbbs**: ID registration and NAT traversal service
- **hbbr**: Relay service for connections

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- arm64 node support (for Raspberry Pi or ARM-based clusters)
- LoadBalancer service support (e.g., MetalLB)

## Port Requirements

The following ports are exposed and required:

### hbbs Ports
- **21115** (TCP): NAT type test
- **21116** (TCP/UDP): ID registration and heartbeat service / TCP hole punching
- **21118** (TCP): Web client support (optional)
- **21114** (TCP): Web console (Pro version only)

### hbbr Ports
- **21117** (TCP): Relay service
- **21119** (TCP): Web client support (optional)

## Installation

### Using FluxCD

This chart is designed to be deployed via FluxCD. The HelmRelease manifest is located at:
```
infrastructure/base/netops/rustdesk-helm-manifest.yaml
```

Update the GitRepository URL in the manifest to point to your repository, then apply:

```bash
kubectl apply -f infrastructure/base/netops/rustdesk-helm-manifest.yaml
```

### Using Helm directly

```bash
helm install rustdesk-server ./charts/rustdesk-server \
  --namespace netops-systems \
  --create-namespace
```

## Configuration

### Key Values

| Parameter | Description | Default |
|-----------|-------------|---------|
| `image.repository` | RustDesk server image | `rustdesk/rustdesk-server` |
| `image.tag` | Image tag | `latest` |
| `persistence.enabled` | Enable persistent storage | `true` |
| `persistence.size` | Storage size | `1Gi` |
| `loadBalancer.enabled` | Use LoadBalancer service | `true` |
| `env.ALWAYS_USE_RELAY` | Force relay mode | `""` (not set) |

### ARM64 Support

The chart includes `nodeSelector` configuration for arm64 nodes:

```yaml
hbbs:
  nodeSelector:
    kubernetes.io/arch: arm64

hbbr:
  nodeSelector:
    kubernetes.io/arch: arm64
```

### MetalLB Configuration

If using MetalLB for LoadBalancer services, add these annotations:

```yaml
loadBalancer:
  enabled: true
  annotations:
    metallb.universe.tf/allow-shared-ip: "rustdesk"
    metallb.universe.tf/loadBalancerIPs: "192.168.1.100"
```

## Usage

After deployment:

1. Get the LoadBalancer IP:
```bash
kubectl get svc -n netops-systems rustdesk-server-hbbs
kubectl get svc -n netops-systems rustdesk-server-hbbr
```

2. Configure RustDesk clients:
   - ID Server: `<LoadBalancer-IP>:21116`
   - Relay Server: `<LoadBalancer-IP>:21117`

3. Retrieve the encryption key:

**Option A: Using the Key Extraction Job (Recommended)**
```bash
kubectl logs -n netops-systems job/rustdesk-server-extract-keys
```

**Option B: Manual Extraction**
```bash
kubectl exec -n netops-systems deployment/rustdesk-server-hbbs -- cat /root/id_ed25519.pub
```

4. **Store the key in Bitwarden Secrets Manager** (Recommended for production)

See [BITWARDEN_INTEGRATION.md](./BITWARDEN_INTEGRATION.md) for detailed instructions on:
- Storing encryption keys in Bitwarden
- Automatic secret synchronization
- Secure key distribution to clients

## Web Client Support

To enable web client access:

1. Enable ingress in values:
```yaml
ingress:
  enabled: true
  className: traefik
  hosts:
    - host: rustdesk.yourdomain.com
      paths:
        - path: /
          pathType: Prefix
          service: hbbs
          port: 21118
```

2. Ensure DNS points to your ingress controller

## Troubleshooting

### Check pod status
```bash
kubectl get pods -n netops-systems -l app.kubernetes.io/name=rustdesk-server
```

### View logs
```bash
kubectl logs -n netops-systems deployment/rustdesk-server-hbbs
kubectl logs -n netops-systems deployment/rustdesk-server-hbbr
```

### Verify services
```bash
kubectl get svc -n netops-systems
```

## References

- [RustDesk Documentation](https://rustdesk.com/docs/)
- [RustDesk Docker Setup](https://rustdesk.com/docs/en/self-host/install/)
