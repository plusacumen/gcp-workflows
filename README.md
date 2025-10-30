# GCP Deployment Workflows

Reusable GitHub Actions workflows for deploying to Google Cloud Platform.

## Quick Start

Add this single workflow file to your repository:

**`.github/workflows/deploy.yml`**

```yaml
name: Deploy

on:
  push:
    branches: [development, master]

jobs:
  set-environment:
    runs-on: ubuntu-latest
    outputs:
      environment: ${{ steps.set-env.outputs.environment }}
    steps:
      - id: set-env
        run: |
          if [[ "${{ github.ref }}" == "refs/heads/development" ]]; then
            echo "environment=staging" >> $GITHUB_OUTPUT
          else
            echo "environment=production" >> $GITHUB_OUTPUT
          fi

  deploy:
    needs: set-environment
    environment: ${{ needs.set-environment.outputs.environment }}
    uses: plusacumen/gcp-workflows/.github/workflows/deploy-gcp.yml@v1
    with:
      image-name: ${{ github.event.repository.name }}
      regions: ${{ vars.REGIONS }}
    secrets: inherit
```

## Required Setup

### 1. Organization Secrets (One-time)

- `VERACODE_API_ID`
- `VERACODE_API_KEY`
- `GH_TOKEN`
- `NPM_TOKEN`

### 2. Repository Environments

Create two environments: `staging` and `production`

**Staging:**

- `GCLOUD_AUTH_KEY`
- `FB_PROJECT_ID`
- `REGIONS` (variable)

**Production:**

- `GCLOUD_AUTH_KEY`
- `FB_PROJECT_ID`
- `REGIONS` (variable)

## Versioning

- `@v1` - Latest stable (recommended)
- `@v1.2.3` - Specific version
- `@main` - Latest development

## Parameters

See [deploy-gcp.yml](.github/workflows/deploy-gcp.yml) for all available inputs.
