# RustDesk Server Quick Start Guide

This guide will get RustDesk server running on your Kubernetes cluster with Bitwarden Secrets Manager integration.

## Prerequisites

- Kubernetes cluster with arm64 nodes
- FluxCD installed and configured
- MetalLB or another LoadBalancer solution
- Bitwarden Secrets Manager organization (for production)

## Quick Deployment (Development/Testing)

For quick testing without Bitwarden:

1. **Update the GitRepository URL**

Edit `infrastructure/base/netops/rustdesk-helm-manifest.yaml` line 9:
```yaml
url: https://github.com/YOUR_USERNAME/YOUR_REPO.git
```

2. **Apply the manifest**
```bash
kubectl apply -f infrastructure/base/netops/rustdesk-helm-manifest.yaml
```

3. **Wait for deployment**
```bash
kubectl get pods -n netops-systems -w
```

4. **Get the LoadBalancer IP**
```bash
kubectl get svc -n netops-systems rustdesk-server-hbbs
```

5. **Extract the public key**
```bash
kubectl logs -n netops-systems job/rustdesk-server-extract-keys
```

6. **Configure clients**
- ID Server: `<LoadBalancer-IP>:21116`
- Relay Server: `<LoadBalancer-IP>:21117`
- Key: Use the public key from step 5

## Production Deployment (with Bitwarden)

For secure production deployment with secrets management:

### Phase 1: Initial Deployment

1. **Create Bitwarden auth token secret**

Create the token secret (use SOPS for GitOps):

```bash
kubectl create secret generic bw-auth-token \
  -n netops-systems \
  --from-literal=token="YOUR_BITWARDEN_MACHINE_ACCOUNT_TOKEN"
```

Or with SOPS (recommended):

```yaml
# infrastructure/base/netops/rustdesk-bw-token.yaml
apiVersion: v1
kind: Secret
metadata:
  name: bw-auth-token
  namespace: netops-systems
type: Opaque
stringData:
  token: "YOUR_BITWARDEN_MACHINE_ACCOUNT_TOKEN"
```

Encrypt it:
```bash
sops --age=YOUR_AGE_PUBLIC_KEY \
  --encrypt --encrypted-regex '^(data|stringData)$' \
  --in-place infrastructure/base/netops/rustdesk-bw-token.yaml
```

2. **Initial deployment with key extraction enabled**

The default configuration in `rustdesk-helm-manifest.yaml` has:
- `bitwarden.enabled: false`
- `keyExtraction.enabled: true`

Apply:
```bash
kubectl apply -f infrastructure/base/netops/rustdesk-helm-manifest.yaml
```

3. **Wait for pods to be ready**
```bash
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=rustdesk-server \
  -n netops-systems \
  --timeout=300s
```

4. **Extract encryption keys**
```bash
# View the extraction job output
kubectl logs -n netops-systems job/rustdesk-server-extract-keys

# Or extract manually
kubectl exec -n netops-systems deployment/rustdesk-server-hbbs -- cat /root/id_ed25519.pub
```

Save this key - you'll need it for the next step.

### Phase 2: Bitwarden Integration

5. **Store keys in Bitwarden Secrets Manager**

- Log into Bitwarden web portal
- Navigate to Secrets Manager
- Select your project (e.g., "pi3s-alpha-infra")
- Create a secret:
  - Name: `rustdesk-public-key`
  - Value: Paste the public key from step 4
- **Copy the Secret ID** (UUID format)

6. **Update Helm values with Bitwarden config**

Edit `infrastructure/base/netops/rustdesk-helm-manifest.yaml`:

```yaml
values:
  bitwarden:
    enabled: true  # Enable Bitwarden sync
    organizationId: "YOUR_BITWARDEN_ORG_ID"
    authTokenSecret: "bw-auth-token"
    authTokenKey: "token"
    secretMappings:
      - bwSecretId: "UUID_FROM_BITWARDEN"
        secretKeyName: PUBLIC_KEY
  
  keyExtraction:
    enabled: false  # Disable now that keys are in Bitwarden
```

7. **Apply the updated configuration**
```bash
kubectl apply -f infrastructure/base/netops/rustdesk-helm-manifest.yaml
```

8. **Verify Bitwarden sync**
```bash
# Check BitwardenSecret resource
kubectl get bitwardensecret -n netops-systems

# Check the synced secret
kubectl get secret -n netops-systems rustdesk-server-keys

# View the secret (to verify it matches)
kubectl get secret -n netops-systems rustdesk-server-keys \
  -o jsonpath='{.data.PUBLIC_KEY}' | base64 -d
```

### Phase 3: Client Configuration

9. **Get LoadBalancer IP**
```bash
kubectl get svc -n netops-systems rustdesk-server-hbbs -o wide
```

10. **Retrieve public key for distribution**
```bash
# From Bitwarden-synced secret
kubectl get secret -n netops-systems rustdesk-server-keys \
  -o jsonpath='{.data.PUBLIC_KEY}' | base64 -d
```

11. **Configure RustDesk clients**
- ID Server: `<LoadBalancer-IP>:21116`
- Relay Server: `<LoadBalancer-IP>:21117`
- Key: Use the public key from step 10

## Verification

### Check Deployment Status
```bash
# All resources
kubectl get all -n netops-systems -l app.kubernetes.io/name=rustdesk-server

# Logs
kubectl logs -n netops-systems deployment/rustdesk-server-hbbs
kubectl logs -n netops-systems deployment/rustdesk-server-hbbr

# Services and endpoints
kubectl get svc,endpoints -n netops-systems
```

### Test Connectivity
```bash
# From a client machine, test ports
nc -zv <LoadBalancer-IP> 21116
nc -zv <LoadBalancer-IP> 21117
```

## Customization

### Configure MetalLB IP Address

Edit `rustdesk-helm-manifest.yaml`:

```yaml
loadBalancer:
  enabled: true
  annotations:
    metallb.universe.tf/allow-shared-ip: "rustdesk"
    metallb.universe.tf/loadBalancerIPs: "192.168.1.100"
```

### Enable Relay Mode

Force all connections through relay:

```yaml
env:
  ALWAYS_USE_RELAY: "Y"
```

### Adjust Resource Limits

For more powerful nodes:

```yaml
hbbs:
  resources:
    limits:
      memory: 512Mi
      cpu: 500m
    requests:
      memory: 256Mi
      cpu: 200m
```

## Troubleshooting

### Pods not starting
```bash
kubectl describe pod -n netops-systems -l app.kubernetes.io/name=rustdesk-server
```

### LoadBalancer stuck in pending
```bash
# Check MetalLB
kubectl logs -n metallb-system deployment/controller

# Verify IP pool configuration
kubectl get ipaddresspool -n metallb-system
```

### Bitwarden secret not syncing
```bash
# Check BitwardenSecret status
kubectl describe bitwardensecret -n netops-systems rustdesk-server-secrets

# Check Bitwarden operator
kubectl logs -n devsecops-systems deployment/bitwarden-sm-operator-controller-manager
```

### Connection issues
1. Verify firewall rules allow ports 21115-21119
2. Check LoadBalancer external IP is reachable
3. Verify encryption key matches between server and client
4. Check server logs for errors

## Next Steps

- **Monitoring**: Set up Prometheus metrics (port 7472)
- **Ingress**: Enable web client support via Traefik
- **Backup**: Document key backup procedures
- **Documentation**: Share connection details with users
- **Testing**: Test from various network configurations

## Resources

- [Full README](./README.md)
- [Bitwarden Integration Guide](./BITWARDEN_INTEGRATION.md)
- [RustDesk Documentation](https://rustdesk.com/docs/)
- [Values Configuration](./values.yaml)
