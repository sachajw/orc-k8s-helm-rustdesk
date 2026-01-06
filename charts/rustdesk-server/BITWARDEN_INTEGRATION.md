# RustDesk with Bitwarden Secrets Manager Integration

This guide explains how to integrate RustDesk server encryption keys with Bitwarden Secrets Manager for secure key storage and distribution.

## Overview

RustDesk generates encryption keys (`id_ed25519` and `id_ed25519.pub`) on first startup. These keys are critical for securing client connections. Instead of manually distributing these keys, we store them in Bitwarden Secrets Manager and sync them to Kubernetes.

## Prerequisites

1. Bitwarden Secrets Manager organization with an active project
2. Bitwarden machine account with access to the project
3. Bitwarden operator deployed in your cluster (at `infrastructure/base/devsecops`)
4. Machine account access token

## Setup Process

### Step 1: Create Bitwarden Auth Secret

Create the authentication secret in the `netops-systems` namespace:

```bash
# Replace with your actual Bitwarden machine account token
kubectl create secret generic bw-auth-token \
  -n netops-systems \
  --from-literal=token="YOUR_BITWARDEN_MACHINE_ACCOUNT_TOKEN"
```

**Alternative: Use SOPS for GitOps**

Create `infrastructure/base/netops/rustdesk-bw-token.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: bw-auth-token
  namespace: netops-systems
type: Opaque
stringData:
  token: "YOUR_BITWARDEN_MACHINE_ACCOUNT_TOKEN"
```

Then encrypt it:

```bash
# Using age (if configured)
sops --age=YOUR_AGE_PUBLIC_KEY \
  --encrypt --encrypted-regex '^(data|stringData)$' \
  --in-place infrastructure/base/netops/rustdesk-bw-token.yaml
```

### Step 2: Initial RustDesk Deployment (Without Bitwarden)

Deploy RustDesk first to generate the encryption keys:

```yaml
# In rustdesk-helm-manifest.yaml
values:
  bitwarden:
    enabled: false  # Disable initially
  
  keyExtraction:
    enabled: true   # Enable key extraction job
```

Apply the manifest:

```bash
kubectl apply -f infrastructure/base/netops/rustdesk-helm-manifest.yaml
```

### Step 3: Extract the Encryption Keys

Wait for the pods to start, then extract the keys:

**Option A: Using the Key Extraction Job**

The Helm chart includes a post-install Job that automatically displays the keys:

```bash
kubectl logs -n netops-systems job/rustdesk-server-extract-keys
```

**Option B: Manual Extraction**

```bash
kubectl exec -n netops-systems deployment/rustdesk-server-hbbs -- cat /root/id_ed25519.pub
```

Save the output - this is your public encryption key.

### Step 4: Store Keys in Bitwarden Secrets Manager

1. Log into your Bitwarden organization web interface
2. Navigate to **Secrets Manager**
3. Select your project (e.g., "pi3s-alpha-infra")
4. Create a new secret:
   - **Name**: `rustdesk-public-key`
   - **Value**: Paste the public key from Step 3
   - **Notes**: "RustDesk server public encryption key - distribute to clients"
5. **Copy the Secret ID** (UUID format) that Bitwarden generates

**Optional: Store Private Key** (for backup/recovery purposes)

```bash
kubectl exec -n netops-systems deployment/rustdesk-server-hbbs -- cat /root/id_ed25519
```

Create another secret in Bitwarden:
- **Name**: `rustdesk-private-key`
- **Value**: Paste the private key
- **Notes**: "RustDesk server private key - DO NOT SHARE"

⚠️ **Security Note**: The private key should remain on the server. Only store it in Bitwarden for backup/disaster recovery purposes.

### Step 5: Update Helm Values with Bitwarden Integration

Update `infrastructure/base/netops/rustdesk-helm-manifest.yaml`:

```yaml
values:
  # Enable Bitwarden integration
  bitwarden:
    enabled: true
    organizationId: "YOUR_BITWARDEN_ORG_ID"
    authTokenSecret: "bw-auth-token"
    authTokenKey: "token"
    secretMappings:
      - bwSecretId: "UUID-FROM-BITWARDEN-FOR-PUBLIC-KEY"
        secretKeyName: PUBLIC_KEY
      # Optional: Include private key for backup
      # - bwSecretId: "UUID-FROM-BITWARDEN-FOR-PRIVATE-KEY"
      #   secretKeyName: PRIVATE_KEY
  
  # Disable key extraction job now that keys are in Bitwarden
  keyExtraction:
    enabled: false
```

Apply the changes:

```bash
kubectl apply -f infrastructure/base/netops/rustdesk-helm-manifest.yaml
```

### Step 6: Verify Bitwarden Secret Sync

Check that the BitwardenSecret resource is working:

```bash
# Check BitwardenSecret status
kubectl get bitwardensecret -n netops-systems

# Check the synced Kubernetes secret
kubectl get secret -n netops-systems rustdesk-server-keys

# View the secret (base64 encoded)
kubectl get secret -n netops-systems rustdesk-server-keys -o yaml
```

## Using the Public Key with Clients

To configure RustDesk clients:

1. **Get the public key from Bitwarden** or from the synced Kubernetes secret:

```bash
kubectl get secret -n netops-systems rustdesk-server-keys \
  -o jsonpath='{.data.PUBLIC_KEY}' | base64 -d
```

2. **Configure RustDesk client**:
   - ID Server: `<LoadBalancer-IP>:21116`
   - Relay Server: `<LoadBalancer-IP>:21117`
   - Key: Paste the public key

## Key Rotation

To rotate encryption keys:

1. Scale down RustDesk deployments:
```bash
kubectl scale deployment -n netops-systems rustdesk-server-hbbs --replicas=0
kubectl scale deployment -n netops-systems rustdesk-server-hbbr --replicas=0
```

2. Delete the PVC to remove old keys:
```bash
kubectl delete pvc -n netops-systems rustdesk-server-data
```

3. Scale back up (new keys will be generated):
```bash
kubectl scale deployment -n netops-systems rustdesk-server-hbbs --replicas=1
kubectl scale deployment -n netops-systems rustdesk-server-hbbr --replicas=1
```

4. Extract new keys and update Bitwarden (repeat Steps 3-4)

5. Distribute new public key to all clients

## Troubleshooting

### BitwardenSecret Not Syncing

```bash
# Check BitwardenSecret status
kubectl describe bitwardensecret -n netops-systems rustdesk-server-secrets

# Check Bitwarden operator logs
kubectl logs -n devsecops-systems deployment/bitwarden-sm-operator-controller-manager
```

### Public Key Not Available

If the key extraction job shows "key not found":

1. Wait for hbbs to fully start (it generates keys on first run)
2. Check hbbs logs:
```bash
kubectl logs -n netops-systems deployment/rustdesk-server-hbbs
```
3. Re-run the job:
```bash
kubectl delete job -n netops-systems rustdesk-server-extract-keys
# Helm will recreate it on next upgrade
```

### Wrong Secret IDs

Verify your Bitwarden secret IDs:

```bash
# Using Bitwarden CLI (if installed)
bws secret list --project-id YOUR_PROJECT_ID

# Or check the web UI
```

## Best Practices

1. **Backup Keys**: Always store both public and private keys in Bitwarden for disaster recovery
2. **Access Control**: Limit Bitwarden project access to operations team only
3. **Token Security**: Protect the machine account token - rotate it regularly
4. **Client Distribution**: Use a secure method to distribute the public key to end users
5. **Monitoring**: Monitor the BitwardenSecret resource for sync failures
6. **Documentation**: Document which clients are using which key version

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ Bitwarden Secrets Manager                                   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Project: pi3s-alpha-infra                            │   │
│  │  ├─ rustdesk-public-key  (UUID: abc-123)            │   │
│  │  └─ rustdesk-private-key (UUID: def-456)            │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              ↓
                    (Bitwarden Operator)
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ Kubernetes - netops-systems namespace                       │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ BitwardenSecret: rustdesk-server-secrets             │  │
│  │  - Maps Bitwarden UUIDs to friendly names            │  │
│  └──────────────────────────────────────────────────────┘  │
│                         ↓                                   │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Secret: rustdesk-server-keys                         │  │
│  │  data:                                               │  │
│  │    PUBLIC_KEY: <base64-encoded>                      │  │
│  │    PRIVATE_KEY: <base64-encoded>                     │  │
│  └──────────────────────────────────────────────────────┘  │
│                         ↓                                   │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Deployments: hbbs, hbbr                              │  │
│  │  - Mount /root from PVC (has generated keys)         │  │
│  │  - Reference PUBLIC_KEY for verification (optional)  │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## References

- [Bitwarden Secrets Manager Documentation](https://bitwarden.com/help/secrets-manager-overview/)
- [RustDesk Self-Hosted Documentation](https://rustdesk.com/docs/en/self-host/install/)
- [Bitwarden Kubernetes Operator](https://bitwarden.com/help/secrets-manager-kubernetes-operator/)
