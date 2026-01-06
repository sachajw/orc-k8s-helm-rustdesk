# Artifact Hub API Integration (Optional)

This document explains how to integrate Artifact Hub API keys for advanced features. **Note: This is optional and not required for basic chart publishing.**

## When You Need API Keys

The Artifact Hub API keys are useful for:
- Programmatically managing repository metadata
- Triggering manual syncs
- Claiming repository ownership
- Advanced automation workflows

For standard Helm chart publishing via GitHub Pages, **you don't need these keys**.

## Your API Keys (Stored in Bitwarden)

You have two secrets stored in Bitwarden under project "pi3s":

- **artifacthub-api-key-id**: `a89edd2d-6081-418d-b7a6-b3ca00e58288`
- **artifacthub-api-key-secret**: `9a735108-8e93-4368-a73d-b3ca00e5f1d9`

## Adding Keys to GitHub Repository (If Needed)

If you want to use API features in GitHub Actions:

### Step 1: Add as GitHub Secrets

1. Go to your repository on GitHub
2. Navigate to **Settings** → **Secrets and variables** → **Actions**
3. Click **New repository secret**
4. Add each secret:

   **First Secret:**
   - Name: `ARTIFACTHUB_API_KEY_ID`
   - Value: Retrieve from Bitwarden: `a89edd2d-6081-418d-b7a6-b3ca00e58288`

   **Second Secret:**
   - Name: `ARTIFACTHUB_API_KEY_SECRET`
   - Value: Retrieve from Bitwarden: `9a735108-8e93-4368-a73d-b3ca00e5f1d9`

### Step 2: Use in GitHub Actions (Optional Enhancement)

If you want to trigger repository syncs after releases, add this job to `.github/workflows/release.yml`:

```yaml
  notify-artifacthub:
    needs: release
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Notify Artifact Hub
        run: |
          curl -X POST https://artifacthub.io/api/v1/repositories/sync \
            -H "Content-Type: application/json" \
            -H "X-API-Key-ID: ${{ secrets.ARTIFACTHUB_API_KEY_ID }}" \
            -H "X-API-Key-Secret: ${{ secrets.ARTIFACTHUB_API_KEY_SECRET }}" \
            -d '{"repositoryID": "YOUR_REPO_ID_FROM_ARTIFACTHUB"}'
```

**Note:** You'll need to replace `YOUR_REPO_ID_FROM_ARTIFACTHUB` with the actual repository ID after adding the repo to Artifact Hub.

## API Usage Examples

### Trigger Manual Repository Sync

```bash
curl -X POST https://artifacthub.io/api/v1/repositories/sync \
  -H "Content-Type: application/json" \
  -H "X-API-Key-ID: YOUR_KEY_ID" \
  -H "X-API-Key-Secret: YOUR_KEY_SECRET" \
  -d '{"repositoryID": "YOUR_REPO_ID"}'
```

### Get Repository Information

```bash
curl -X GET https://artifacthub.io/api/v1/repositories/YOUR_REPO_ID \
  -H "X-API-Key-ID: YOUR_KEY_ID" \
  -H "X-API-Key-Secret: YOUR_KEY_SECRET"
```

### Update Repository Metadata

```bash
curl -X PUT https://artifacthub.io/api/v1/repositories/YOUR_REPO_ID \
  -H "Content-Type: application/json" \
  -H "X-API-Key-ID: YOUR_KEY_ID" \
  -H "X-API-Key-Secret: YOUR_KEY_SECRET" \
  -d '{
    "displayName": "RustDesk Helm Charts",
    "description": "Official Helm charts for RustDesk server"
  }'
```

## Security Best Practices

1. **Never commit API keys to Git**
2. **Store in Bitwarden** (✅ You've already done this)
3. **Use GitHub Secrets** for CI/CD workflows
4. **Rotate keys periodically**
5. **Use minimum required permissions**

## Retrieving Keys from Bitwarden

### Using Bitwarden CLI

```bash
# Install Bitwarden CLI
brew install bitwarden-cli

# Login
bw login

# Get API Key ID
bw get item artifacthub-api-key-id

# Get API Key Secret
bw get item artifacthub-api-key-secret
```

### Using Bitwarden Secrets Manager (Kubernetes)

Since you have Bitwarden operator in your cluster, you can create a BitwardenSecret resource:

```yaml
apiVersion: k8s.bitwarden.com/v1
kind: BitwardenSecret
metadata:
  name: artifacthub-api-keys
  namespace: netops-systems
spec:
  organizationId: "YOUR_BITWARDEN_ORG_ID"
  secretName: artifacthub-credentials
  authToken:
    secretName: bw-auth-token
    secretKey: token
  map:
    - bwSecretId: "a89edd2d-6081-418d-b7a6-b3ca00e58288"
      secretKeyName: API_KEY_ID
    - bwSecretId: "9a735108-8e93-4368-a73d-b3ca00e5f1d9"
      secretKeyName: API_KEY_SECRET
```

Then use in a Job or CronJob for automation.

## Standard Workflow (No API Keys Needed)

For most users, the standard workflow is simpler and doesn't require API keys:

1. **Push charts to GitHub** → GitHub Actions runs
2. **GitHub Actions** → Publishes to GitHub Pages
3. **Artifact Hub** → Automatically discovers and indexes (every ~30 min)
4. **Users** → Install via `helm repo add`

This is the **recommended approach** and requires no API keys.

## Conclusion

- ✅ Your API keys are safely stored in Bitwarden
- ✅ You can add them to GitHub Secrets if needed
- ✅ For standard chart publishing, you don't need them
- ✅ Use them only for advanced API automation

Proceed with the standard GitHub Pages workflow unless you have specific automation needs!
