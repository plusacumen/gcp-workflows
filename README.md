# GCP Deployment Workflows

Reusable GitHub Actions workflows for deploying to Google Cloud Platform.

## Overview

This repository contains reusable GitHub Actions workflows that can be called from any repository in your organization. The workflows handle building, testing, and deploying applications to Google Cloud Platform.

## Complete GitHub Setup Walkthrough

Follow these steps in order to set up the workflow in your GitHub repository.

### Step 1: Configure Organization Secrets (One-Time Setup)

These secrets are shared across all repositories in your organization, so you only need to set them once.

1. **Navigate to Organization Settings**
   - Go to your GitHub organization page (e.g., `github.com/plusacumen`)
   - Click on **Settings** (in the top navigation bar)

2. **Access Secrets and Variables**
   - In the left sidebar, click **Secrets and variables** → **Actions**

3. **Add Organization Secrets**
   - Click on the **Secrets** tab
   - Click **New organization secret**
   - Add each of the following secrets one by one:

   **Secret 1: `VERACODE_API_ID`**
   - Name: `VERACODE_API_ID`
   - Value: Your Veracode API identifier
   - Click **Add secret**

   **Secret 2: `VERACODE_API_KEY`**
   - Name: `VERACODE_API_KEY`
   - Value: Your Veracode API key
   - Click **Add secret**

   **Secret 3: `GH_TOKEN`**
   - Name: `GH_TOKEN`
   - Value: A GitHub Personal Access Token (PAT) with `repo` scope
   - Click **Add secret**

   **Secret 4: `NPM_TOKEN`**
   - Name: `NPM_TOKEN`
   - Value: Your NPM authentication token
   - Click **Add secret**

> **Note:** Organization secrets are automatically available to all repositories in your organization, so you won't need to configure these again for other repositories.

### Step 2: Configure Repository Variables

1. **Navigate to Your Repository**
   - Go to the repository where you want to add the workflow
   - Click on **Settings** (in the repository navigation bar)

2. **Access Variables**
   - In the left sidebar, click **Secrets and variables** → **Actions**
   - Click on the **Variables** tab (not Secrets)

3. **Add REGIONS Variable**
   - Click **New repository variable**
   - Name: `REGIONS`
   - Value: Comma-separated list of GCP regions (e.g., `us-central1,us-east1`)
   - Click **Add variable**

### Step 3: Create Repository Environments

You need to create two environments: `staging` and `production`.

#### Create Staging Environment

1. **Navigate to Environments**
   - In your repository, go to **Settings**
   - In the left sidebar, click **Environments**

2. **Create New Environment**
   - Click **New environment**
   - Name: `staging` (must match exactly)
   - Click **Configure environment**

3. **Add Staging Secrets and Variables**
   - Scroll down to the **Environment secrets** section
   - Click **Add secret**

   **Secret 1: `GCLOUD_AUTH_KEY`**
   - Name: `GCLOUD_AUTH_KEY`
   - Value: Your Google Cloud service account JSON key (paste the entire JSON content)
   - Click **Add secret**

   **Secret 2: `FB_PROJECT_ID`**
   - Name: `FB_PROJECT_ID`
   - Value: Your Firebase/GCP Project ID (e.g., `my-project-12345`)
   - Click **Add secret**

   **Optional: Environment Variables** (if you need different values per environment)
   - Scroll to **Environment variables** section
   - Click **Add variable**
   - Name: `REGIONS`
   - Value: Staging regions (e.g., `us-central1`)
   - Click **Add variable**

4. **Save Environment**
   - Scroll to the top and click **Save protection rules** (or just navigate away - it auto-saves)

#### Create Production Environment

1. **Create Another Environment**
   - Click **New environment** again
   - Name: `production` (must match exactly)
   - Click **Configure environment**

2. **Add Production Secrets and Variables**
   - Add the same secrets as staging:
     - `GCLOUD_AUTH_KEY` (can be the same or different service account)
     - `FB_PROJECT_ID` (can be the same or different project)

   **Optional: Environment Variables** (if you need different values per environment)
   - Scroll to **Environment variables** section
   - Click **Add variable**
   - Name: `REGIONS`
   - Value: Production regions (e.g., `us-central1,us-east1`)
   - Click **Add variable**

3. **Save Environment**
   - Click **Save protection rules** or navigate away

> **Optional:** You can configure environment protection rules (required reviewers, deployment branches, etc.) in the environment settings if needed.

### Step 4: Create the Workflow File

1. **Navigate to Your Repository**
   - Go to the main page of your repository

2. **Create Workflow Directory**
   - Click **Add file** → **Create new file**
   - Type: `.github/workflows/deploy.yml` (this will create the directories automatically)

3. **Add Workflow Content**
   - Copy and paste the workflow YAML from the section below
   - Make sure the indentation is correct (use spaces, not tabs)

4. **Commit the File**
   - Scroll down to the commit section
   - Add a commit message: `Add GCP deployment workflow`
   - Choose **Commit directly to the `main` branch** (or your default branch)
   - Click **Commit new file**

### Step 5: Verify Your Setup

After completing all steps, verify your configuration:

1. **Check Organization Secrets**
   - Go to Organization Settings → Secrets and variables → Actions → Secrets
   - Verify all 4 secrets are present: `VERACODE_API_ID`, `VERACODE_API_KEY`, `GH_TOKEN`, `NPM_TOKEN`

2. **Check Repository Variables**
   - Go to Repository Settings → Secrets and variables → Actions → Variables
   - Verify `REGIONS` variable is set

3. **Check Environments**
   - Go to Repository Settings → Environments
   - Verify both `staging` and `production` environments exist
   - Click on each environment to verify secrets are set:
     - `GCLOUD_AUTH_KEY`
     - `FB_PROJECT_ID`

4. **Check Workflow File**
   - Go to the **Actions** tab in your repository
   - You should see the workflow file listed (it may show as inactive until you push to `development` or `master`)

5. **Test the Workflow**
   - Push a commit to the `development` branch (or `master` branch)
   - Go to the **Actions** tab
   - You should see a workflow run start automatically
   - Click on it to monitor the deployment progress

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

> **Note:** This example uses a repository-level `REGIONS` variable. If you need different regions per environment, see the [Differentiating Staging and Production Variables](#differentiating-staging-and-production-variables) section below.

### Quick Reference: What You Need

**Organization Secrets** (set once for all repos):

- `VERACODE_API_ID`
- `VERACODE_API_KEY`
- `GH_TOKEN`
- `NPM_TOKEN`

**Repository Variables**:

- `REGIONS` (e.g., `us-central1,us-east1`)

**Environment Secrets** (per environment):

- `GCLOUD_AUTH_KEY` (in both `staging` and `production`)
- `FB_PROJECT_ID` (in both `staging` and `production`)

### Step 2: Commit and Push

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

## Differentiating Staging and Production Variables

If you need **different variables** for staging and production (e.g., different regions, different project IDs), you have two options:

### Option 1: Environment-Level Variables (Recommended for Different Values)

Use this approach when staging and production need different values.

#### Step 1: Update Your Workflow

Replace your current workflow with this version that reads environment variables:

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
      - name: Set environment
        id: set-env
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
      - name: Get regions from environment
        id: get-regions
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

#### Step 2: Set Variables in Each Environment

1. Go to **Repository Settings → Environments**
2. Click on the **`staging`** environment
3. Scroll to **Environment variables** section (not secrets)
4. Click **Add variable**
   - Name: `REGIONS`
   - Value: Your staging regions (e.g., `us-central1`)
   - Click **Add variable**
5. Click on the **`production`** environment
6. Scroll to **Environment variables** section
7. Click **Add variable**
   - Name: `REGIONS`
   - Value: Your production regions (e.g., `us-central1,us-east1`)
   - Click **Add variable**

#### Step 3: Remove Repository-Level Variable (if it exists)

- Go to **Repository Settings → Secrets and variables → Actions → Variables**
- If `REGIONS` exists there, delete it (environment variables take precedence)

### Option 2: Repository-Level Variable (Use When Values Are the Same)

If staging and production use the **same values**, keep your current workflow and set the variable at the repository level:

- Go to **Repository Settings → Secrets and variables → Actions → Variables**
- Set `REGIONS` once (both environments will use it)

### Common Use Cases

**Different Regions:**

- Staging: `us-central1` (single region for testing)
- Production: `us-central1,us-east1` (multiple regions for redundancy)

**Different Project IDs:**

- You can also set `FB_PROJECT_ID` as an environment variable if needed
- But typically, you'd use different secrets for `FB_PROJECT_ID` in each environment

**Different Ports:**

- If you need different ports, you can add `port` to the `with:` section:

  ```yaml
  with:
    image-name: ${{ github.event.repository.name }}
    regions: ${{ vars.REGIONS }}
    port: ${{ vars.PORT }}  # Set PORT in each environment
  ```

## Alternative: Using Environment-Specific Regions (Legacy Reference)

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
