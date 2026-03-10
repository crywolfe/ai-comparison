---
module: Infrastructure
date: 2026-03-09
problem_type: integration_issue
component: tooling
symptoms:
  - "InvalidTemplate: The value for the template parameter 'repositoryUrl' at line '1' and column '712' is not provided"
  - "RepositoryToken is invalid. Please ensure the Github repository exists and the RepositoryToken is for an admin of the repository."
  - "Provided token has invalid permissions and cannot be used to setup Github Action CI/CD. Admin rights are required for the repository."
root_cause: config_error
resolution_type: workflow_improvement
severity: high
tags: [azure, static-web-apps, github-enterprise, emu, github-actions, deployment, pat, fine-grained-pat]
---

# Troubleshooting: Azure Static Web App Creation Fails with RepositoryToken Invalid (GitHub Enterprise Managed Users)

## Problem

`az staticwebapp create` fails with "RepositoryToken is invalid" when attempting to link a GitHub repository owned by a GitHub Enterprise organization, even when the user has org admin rights.

## Environment

- Module: Infrastructure
- Affected Component: Azure CLI (`az staticwebapp create`), GitHub Enterprise Cloud (EMU)
- Date: 2026-03-09

## Symptoms

- `InvalidTemplate: The value for the template parameter 'repositoryUrl' at line '1' and column '712' is not provided` — first error when omitting `--source`
- `RepositoryToken is invalid. Please ensure the Github repository exists and the RepositoryToken is for an admin of the repository.` — when using `--login-with-github`
- Same error when using `--token` with a classic PAT (with `repo` + `workflow` scopes)
- Same error when using `--token` with a fine-grained PAT scoped to the org, with Administration/Contents/Workflows read+write permissions
- User has confirmed org admin role (`gh api orgs/Konexo-US/memberships/<user> --jq '.role'` returns `"admin"`)
- Repository is accessible (`gh repo view` works)

## What Didn't Work

**Attempted Solution 1:** `--login-with-github` device flow
- **Why it failed:** The GitHub device auth flow completed but Azure rejected the resulting token. GitHub Enterprise Managed User accounts have restricted OAuth token issuance.

**Attempted Solution 2:** Classic PAT with `repo` + `workflow` scopes via `--token`
- **Why it failed:** Same rejection. The PAT was created under a personal EMU account not under the org resource owner. Classic PATs from EMU accounts may not work with Azure's token validation.

**Attempted Solution 3:** Fine-grained PAT with resource owner = org, Administration/Contents/Workflows Read+Write
- **Why it failed:** Still rejected. Azure's `az staticwebapp create --token` validation appears incompatible with GitHub Enterprise Managed User (EMU) accounts regardless of token type or permissions. The token validation mechanism Azure uses does not work through the GitHub Enterprise identity management layer.

## Solution

Create the Azure Static Web App **without GitHub integration**, then wire up CI/CD manually via GitHub Actions.

**Step 1: Create the SWA without linking GitHub**
```bash
az staticwebapp create \
    --name <app-name> \
    --resource-group <resource-group> \
    --location "eastus2"
```

**Step 2: Get the deployment token**
```bash
az staticwebapp secrets list \
    --name <app-name> \
    --resource-group <resource-group> \
    --query "properties.apiKey" -o tsv
```

**Step 3: Add the token as a GitHub Actions secret**
```bash
gh secret set AZURE_STATIC_WEB_APPS_API_TOKEN \
    --repo <org>/<repo> \
    --body "<TOKEN_FROM_STEP_2>"
```

**Step 4: Create `.github/workflows/azure-static-web-apps.yml`**
```yaml
name: Deploy to Azure Static Web Apps

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN }}
          repo_token: ${{ secrets.GITHUB_TOKEN }}
          action: upload
          app_location: "/"
          skip_app_build: true
```

## Why This Works

1. **Root cause:** Azure Static Web Apps' `--token` and `--login-with-github` flows use GitHub's OAuth/token APIs to set up webhooks and repository secrets programmatically. GitHub Enterprise Managed User (EMU) accounts are identity-provider managed (e.g., via Okta/AAD), and the resulting tokens are restricted in ways that break Azure's token validation — even with all necessary permissions.

2. **Why the workaround works:** By decoupling Azure SWA creation from GitHub integration entirely, we bypass the problematic OAuth handshake. The deployment token (SWA API key) is a pure Azure credential. The GitHub Actions workflow uses `GITHUB_TOKEN` (issued natively by GitHub Actions runner) for repository context, which works fine with EMU accounts.

3. **Key insight:** The `skip_app_build: true` flag is important for pre-built static sites (plain HTML/JS/CSS) — without it, the action tries to run a build step and may fail if no build tool is configured.

## Prevention

- If the GitHub org is managed by a corporate identity provider (GitHub Enterprise Cloud with EMU), **always use the manual workflow approach** — do not attempt `--login-with-github` or `--token` flags.
- Check for EMU: look for "Users managed by [Company]" badge in the GitHub top navigation bar.
- To verify org admin status before spending time debugging token issues: `gh api orgs/<org>/memberships/<username> --jq '.role'`
- Store the Azure SWA deployment token in GitHub Secrets immediately after creation — it's needed for every CI/CD run.

## Related Issues

No related issues documented yet.
