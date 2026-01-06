# Publishing RustDesk Helm Chart to Artifact Hub

This guide explains how to publish the RustDesk Helm chart to Artifact Hub.

## Prerequisites

- GitHub repository with the Helm chart
- GitHub Pages enabled (for traditional Helm repo) OR OCI registry access
- Helm CLI installed (`helm version`)
- Artifact Hub account

## Option 1: Traditional Helm Repository (GitHub Pages)

This is the recommended approach for simplicity.

### Step 1: Update Chart Metadata

Edit `charts/rustdesk-server/Chart.yaml` and update placeholders:

```yaml
home: https://github.com/YOUR_USERNAME/YOUR_REPO
sources:
  - https://github.com/YOUR_USERNAME/YOUR_REPO/tree/main/charts/rustdesk-server
maintainers:
  - name: YOUR_NAME
    email: YOUR_EMAIL
    url: https://github.com/YOUR_USERNAME
```

### Step 2: Package the Chart

```bash
# From repository root
cd charts

# Package the chart
helm package rustdesk-server

# This creates rustdesk-server-1.0.0.tgz
```

### Step 3: Create Helm Repository Index

```bash
# Create or update index.yaml
helm repo index . --url https://YOUR_USERNAME.github.io/YOUR_REPO/charts/

# This creates/updates index.yaml with chart metadata
```

### Step 4: Update artifacthub-repo.yml

Edit `charts/artifacthub-repo.yml`:

```yaml
repositoryID: YOUR_REPOSITORY_ID_FROM_ARTIFACT_HUB
owners:
  - name: YOUR_NAME
    email: YOUR_EMAIL
```

**Note**: You'll get the `repositoryID` after registering on Artifact Hub (Step 6).

### Step 5: Setup GitHub Pages

**Option A: Serve from /charts directory**

1. Go to your GitHub repository settings
2. Navigate to Pages
3. Select "Deploy from a branch"
4. Choose `main` branch and `/charts` folder
5. Save

Your chart repository will be available at:
```
https://YOUR_USERNAME.github.io/YOUR_REPO/charts/
```

**Option B: Dedicated gh-pages branch**

```bash
# Create orphan branch for charts
git checkout --orphan gh-pages
git rm -rf .

# Copy chart files
cp -r ../charts/* .

# Commit and push
git add .
git commit -m "Initial Helm repository"
git push origin gh-pages

# Switch back to main
git checkout main
```

Configure GitHub Pages to serve from `gh-pages` branch.

### Step 6: Add Repository to Artifact Hub

1. Go to https://artifacthub.io
2. Sign in (supports GitHub, Google, or email)
3. Click your profile → "Control Panel"
4. Click "Add" → "Repository"
5. Fill in the form:
   - **Name**: rustdesk-helm-charts (or your preferred name)
   - **Display Name**: RustDesk Helm Charts
   - **URL**: `https://YOUR_USERNAME.github.io/YOUR_REPO/charts/`
   - **Kind**: Helm charts
   - **Repository Type**: HTTP (default)
   - **Description**: Helm charts for RustDesk self-hosted remote desktop server
6. Click "Add"

**Copy the Repository ID** that Artifact Hub generates and update `artifacthub-repo.yml`.

### Step 7: Verify Publication

1. Wait 5-30 minutes for Artifact Hub to index your repository
2. Go to https://artifacthub.io/packages/search?kind=0&ts_query=rustdesk
3. Your chart should appear in search results

## Option 2: OCI Registry (GitHub Container Registry)

For OCI-based distribution using GitHub Container Registry (ghcr.io).

### Step 1: Enable GitHub Container Registry

1. Create a GitHub Personal Access Token (PAT):
   - Go to GitHub Settings → Developer settings → Personal access tokens
   - Generate new token (classic)
   - Select scopes: `write:packages`, `read:packages`, `delete:packages`
   - Copy the token

2. Login to GitHub Container Registry:
```bash
export GITHUB_TOKEN=your_token_here
echo $GITHUB_TOKEN | helm registry login ghcr.io -u YOUR_USERNAME --password-stdin
```

### Step 2: Package and Push Chart

```bash
# Package the chart
cd charts
helm package rustdesk-server

# Push to OCI registry
helm push rustdesk-server-1.0.0.tgz oci://ghcr.io/YOUR_USERNAME/charts
```

### Step 3: Push Artifact Hub Metadata

Install `oras` CLI:
```bash
# macOS
brew install oras

# Or download from https://github.com/oras-project/oras/releases
```

Push the metadata file:
```bash
cd charts

oras push \
  ghcr.io/YOUR_USERNAME/charts/rustdesk-server:artifacthub.io \
  --config /dev/null:application/vnd.cncf.artifacthub.config.v1+yaml \
  artifacthub-repo.yml:application/vnd.cncf.artifacthub.repository-metadata.layer.v1.yaml
```

### Step 4: Add OCI Repository to Artifact Hub

1. Go to Artifact Hub → Control Panel → Add Repository
2. Fill in:
   - **Name**: rustdesk-helm-charts-oci
   - **URL**: `oci://ghcr.io/YOUR_USERNAME/charts/rustdesk-server`
   - **Kind**: Helm charts
   - **Repository Type**: OCI (select from dropdown)
3. Click "Add"

## Publishing New Versions

### Update Chart Version

1. Edit `Chart.yaml`:
```yaml
version: 1.1.0  # Increment version
appVersion: "latest"
```

2. Update `artifacthub.io/changes` annotation with changelog:
```yaml
annotations:
  artifacthub.io/changes: |
    - kind: added
      description: New feature X
    - kind: changed
      description: Improved Y
    - kind: fixed
      description: Fixed bug Z
```

### Traditional Helm Repo

```bash
# Package new version
cd charts
helm package rustdesk-server

# Update index (merge with existing)
helm repo index . --url https://YOUR_USERNAME.github.io/YOUR_REPO/charts/ --merge index.yaml

# Commit and push
git add .
git commit -m "Release rustdesk-server v1.1.0"
git push origin main
```

Artifact Hub will automatically detect and index the new version within 30 minutes.

### OCI Registry

```bash
# Package and push new version
cd charts
helm package rustdesk-server
helm push rustdesk-server-1.1.0.tgz oci://ghcr.io/YOUR_USERNAME/charts

# Update metadata if needed
oras push \
  ghcr.io/YOUR_USERNAME/charts/rustdesk-server:artifacthub.io \
  --config /dev/null:application/vnd.cncf.artifacthub.config.v1+yaml \
  artifacthub-repo.yml:application/vnd.cncf.artifacthub.repository-metadata.layer.v1.yaml
```

## Verification

### Test Chart Installation

**From traditional Helm repo:**
```bash
helm repo add rustdesk https://YOUR_USERNAME.github.io/YOUR_REPO/charts/
helm repo update
helm search repo rustdesk

helm install rustdesk-test rustdesk/rustdesk-server \
  --namespace test \
  --create-namespace \
  --dry-run
```

**From OCI registry:**
```bash
helm show chart oci://ghcr.io/YOUR_USERNAME/charts/rustdesk-server

helm install rustdesk-test oci://ghcr.io/YOUR_USERNAME/charts/rustdesk-server \
  --version 1.0.0 \
  --namespace test \
  --create-namespace \
  --dry-run
```

### Validate Artifact Hub Listing

Check your chart on Artifact Hub:
- Search results: https://artifacthub.io/packages/search?kind=0&ts_query=rustdesk
- Direct link: `https://artifacthub.io/packages/helm/YOUR_REPO_NAME/rustdesk-server`

Verify:
- ✅ Chart description and README display correctly
- ✅ Install instructions are accurate
- ✅ Links to documentation work
- ✅ Keywords appear in search
- ✅ Version history shows correctly
- ✅ Security scanning completed (if enabled)

## Troubleshooting

### Chart Not Appearing on Artifact Hub

1. **Check repository URL**: Ensure `index.yaml` is accessible at the URL you provided
```bash
curl https://YOUR_USERNAME.github.io/YOUR_REPO/charts/index.yaml
```

2. **Verify index.yaml format**:
```bash
helm repo index --debug .
```

3. **Check Artifact Hub sync status**: Go to Control Panel → Your Repository → View sync logs

4. **Wait**: Initial indexing can take up to 30 minutes

### Invalid Chart Format

```bash
# Lint your chart before packaging
helm lint charts/rustdesk-server

# Validate package
helm package charts/rustdesk-server --debug
```

### GitHub Pages Not Serving Files

1. Check GitHub Pages deployment status in repository Actions tab
2. Ensure `index.yaml` and `.tgz` files are committed to the correct branch
3. Verify MIME types are correct (GitHub Pages should handle this automatically)

### OCI Registry Authentication Issues

```bash
# Re-login with fresh token
echo $GITHUB_TOKEN | helm registry login ghcr.io -u YOUR_USERNAME --password-stdin

# Test with oras
oras version
oras pull ghcr.io/YOUR_USERNAME/charts/rustdesk-server:1.0.0
```

## Automation with GitHub Actions

Create `.github/workflows/release-chart.yml`:

```yaml
name: Release Helm Chart

on:
  push:
    branches:
      - main
    paths:
      - 'charts/rustdesk-server/**'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Configure Git
        run: |
          git config user.name "$GITHUB_ACTOR"
          git config user.email "$GITHUB_ACTOR@users.noreply.github.com"

      - name: Install Helm
        uses: azure/setup-helm@v3

      - name: Run chart-releaser
        uses: helm/chart-releaser-action@v1.6.0
        env:
          CR_TOKEN: "${{ secrets.GITHUB_TOKEN }}"
        with:
          charts_dir: charts
```

This automatically packages and publishes charts when you push changes.

## Best Practices

1. **Semantic Versioning**: Follow semver for chart versions
2. **Changelog**: Always update `artifacthub.io/changes` annotation
3. **Testing**: Test chart installation before releasing
4. **Documentation**: Keep README.md comprehensive and up-to-date
5. **Security**: Regularly update base images and dependencies
6. **Verification**: Consider setting up verified publisher status

## Resources

- [Artifact Hub Helm Documentation](https://artifacthub.io/docs/topics/repositories/helm-charts/)
- [Helm Chart Best Practices](https://helm.sh/docs/chart_best_practices/)
- [Artifact Hub Annotations](https://artifacthub.io/docs/topics/annotations/helm/)
- [GitHub Pages Documentation](https://docs.github.com/en/pages)
- [Helm OCI Support](https://helm.sh/docs/topics/registries/)
