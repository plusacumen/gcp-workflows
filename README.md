# GCP Deployment Workflows

Reusable GitHub Actions workflows for deploying to Google Cloud Platform.

## Overview

This repository contains reusable GitHub Actions workflows that can be called from any repository in your organization. The workflows handle building, testing, and deploying applications to Google Cloud Platform.

## Adding to Your Repository

### Step 1: Create the Workflow File

In your repository, create a new workflow file at `.github/workflows/deploy.yml`:

```bash
mkdir -p .github/workflows
```

Then create the file with the following content:

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

> **Note:** This example uses a repository-level `REGIONS` variable. If you need different regions per environment, see the [Alternative workflow](#alternative-using-environment-specific-regions) below.

### Step 2: Configure Organization Secrets

Configure the following secrets at the **organization level** (Settings → Secrets and variables → Actions → Organization secrets):

- `VERACODE_API_ID` - Veracode API identifier
- `VERACODE_API_KEY` - Veracode API key
- `GH_TOKEN` - GitHub token for repository access
- `NPM_TOKEN` - NPM token for package installation

> **Note:** Organization secrets are automatically available to all repositories in the organization.

### Step 3: Configure Repository Variables

Go to **Settings → Secrets and variables → Actions → Variables** and add:

- `REGIONS` - Comma-separated list of GCP regions (e.g., `us-central1,us-east1`)

> **Note:** If you need different regions per environment, use the workflow approach below that reads environment variables.

### Step 4: Create Repository Environments

In your repository, go to **Settings → Environments** and create two environments:

#### Staging Environment

1. Click **New environment** and name it `staging`
2. Add the following **secrets**:
   - `GCLOUD_AUTH_KEY` - Google Cloud service account key (JSON)
   - `FB_PROJECT_ID` - Firebase project ID

#### Production Environment

1. Click **New environment** and name it `production`
2. Add the same **secrets** as staging:
   - `GCLOUD_AUTH_KEY` - Google Cloud service account key (JSON)
   - `FB_PROJECT_ID` - Firebase project ID

### Step 5: Commit and Push

```bash
git add .github/workflows/deploy.yml
git commit -m "Add GCP deployment workflow"
git push
```

The workflow will automatically trigger on pushes to `development` or `master` branches.

## How It Works

1. **Branch Detection**: The workflow detects which branch triggered the run
   - `development` → deploys to `staging` environment
   - `master` → deploys to `production` environment

2. **Reusable Workflow**: Your workflow calls the reusable workflow from this repository
   - Uses `uses: plusacumen/gcp-workflows/.github/workflows/deploy-gcp.yml@v1`
   - Passes configuration via `with:` parameters
   - Inherits secrets from the environment context

3. **Environment Protection**: Each environment can have protection rules configured in GitHub (required reviewers, deployment branches, etc.)

## Versioning

When referencing the reusable workflow, you can use:

- `@v1` - Latest stable version (recommended for production)
- `@v1.2.3` - Specific version (recommended for stability)
- `@main` - Latest development version (use with caution)

## Alternative: Using Environment-Specific Regions

If you need different regions for staging and production, you can set `REGIONS` as an environment variable in each environment and use this workflow:

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

  get-regions:
    runs-on: ubuntu-latest
    needs: set-environment
    environment: ${{ needs.set-environment.outputs.environment }}
    outputs:
      regions: ${{ steps.get-regions.outputs.regions }}
    steps:
      - id: get-regions
        run: echo "regions=${{ vars.REGIONS }}" >> $GITHUB_OUTPUT

  deploy:
    needs: [set-environment, get-regions]
    environment: ${{ needs.set-environment.outputs.environment }}
    uses: plusacumen/gcp-workflows/.github/workflows/deploy-gcp.yml@v1
    with:
      image-name: ${{ github.event.repository.name }}
      regions: ${{ needs.get-regions.outputs.regions }}
    secrets: inherit
```

In this case, add `REGIONS` as a variable in each environment (staging and production) instead of at the repository level.

## Customization

### Changing Branch Names

To use different branch names, modify the `on.push.branches` section:

```yaml
on:
  push:
    branches: [staging, main]  # Your branch names
```

And update the environment detection logic accordingly:

```yaml
if [[ "${{ github.ref }}" == "refs/heads/staging" ]]; then
  echo "environment=staging" >> $GITHUB_OUTPUT
else
  echo "environment=production" >> $GITHUB_OUTPUT
fi
```

### Additional Parameters

See the [deploy-gcp.yml](.github/workflows/deploy-gcp.yml) workflow file for all available input parameters that can be passed via the `with:` section.

## Troubleshooting

### Workflow Not Triggering

- Ensure the workflow file is in `.github/workflows/` directory
- Check that you're pushing to the configured branches
- Verify the workflow file has valid YAML syntax

### Authentication Errors

- Verify `GCLOUD_AUTH_KEY` is set correctly in the environment secrets
- Ensure the service account has necessary permissions in GCP
- Check that `FB_PROJECT_ID` matches your Firebase project

### Missing Secrets

- Organization secrets are available to all repos automatically
- Repository environment secrets must be set per repository
- Variables (like `REGIONS`) can be set at repository level or environment level
  - Repository level: Access directly with `${{ vars.REGIONS }}` in `with:` section
  - Environment level: Must be read in a job step and passed as output (see Alternative workflow above)

### Variable Access Errors

If you see errors about `vars.REGIONS` not being accessible:

- **Repository-level variable**: Can be used directly in `with:` sections
- **Environment-level variable**: Must be read in a job step first, then passed as an output to the deploy job
- Ensure the variable is set in the correct location (repository vs environment)

## Support

For issues or questions, please open an issue in this repository.
