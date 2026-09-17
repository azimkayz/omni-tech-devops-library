# omni-tech-devops-library

Central "single source of truth" for Omni-Tech's Azure Terraform deployment pipeline.

This repository defines one reusable GitHub Actions workflow that every project repo calls instead of maintaining its own copy of the deployment logic. Update the pipeline once here, and the change automatically applies the next time any consumer repo runs — no manual edits across 15+ repos.

## What it does

`.github/workflows/terraform-deploy.yml` runs, in order:

1. Checkout the **consumer repo's** code (not this repo's)
2. Azure login via OIDC (no stored credentials/secrets on the Azure side)
3. Terraform setup + `init` (authenticated against the shared remote state backend)
4. `terraform validate`
5. TFLint
6. Checkov security scan (hard fail on violations — this is the enforced guardrail)
7. `terraform plan`
8. `terraform apply` (only if `apply: true` is passed in, e.g. only on pushes to `main`)

## How to call it from a project repo

Add a short caller workflow to your repo — no deployment logic, just inputs:

```yaml
name: Deploy Infra

on:
  push:
    branches: [main]
  pull_request:

permissions:
  id-token: write
  contents: read

jobs:
  call-central-pipeline:
    uses: azimkayz/omni-tech-devops-library/.github/workflows/terraform-deploy.yml@main
    with:
      environment: "production"
      resource_group: "rg-your-project-name"
      working_directory: "./infra"
      apply: ${{ github.event_name == 'push' }}
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

## Inputs

| Name | Required | Default | Description |
|---|---|---|---|
| `environment` | yes | — | Deployment environment name (e.g. `production`, `staging`) |
| `resource_group` | yes | — | Azure Resource Group this project deploys into |
| `terraform_version` | no | `1.9.5` | Pinned Terraform CLI version, keeps all projects in sync |
| `working_directory` | no | `.` | Path to the project's Terraform config, relative to its repo root |
| `apply` | no | `false` | Whether to run `terraform apply` after a successful plan |

## Secrets

Each consumer repo must configure its own three repository secrets, tied to its own Azure OIDC federated credential:

| Secret | Description |
|---|---|
| `AZURE_CLIENT_ID` | App Registration (client) ID |
| `AZURE_TENANT_ID` | Azure AD tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Target Azure subscription ID |

## Prerequisites for a new consumer repo

1. An Azure AD App Registration with a **federated credential** scoped to `repo:<org>/<your-repo>:ref:refs/heads/main` (or the branch you deploy from)
2. That App Registration's service principal granted **Contributor** on the target resource group
3. The three secrets above added under the consumer repo's **Settings → Secrets and variables → Actions**
4. A remote Terraform state backend (this org uses a shared `azurerm` backend — see `rg-omni-tech-tfstate` / `omnitechtfstate` storage account); the service principal also needs **Storage Blob Data Contributor** on that storage account if `use_azuread_auth = true` is set in your backend block

## Security guardrails enforced here

- Checkov scan blocks `terraform apply` on any unaddressed security finding (TLS version, public access, encryption, etc.)
- TFLint enforces style/correctness
- Terraform version is pinned centrally, eliminating version drift between projects
- OIDC login means no long-lived Azure credentials are stored in any repo

## Making changes

Any edit to `.github/workflows/terraform-deploy.yml` on `main` takes effect for **all** consumer repos on their next run — no coordination needed with individual project teams. Treat this file with the same care as production infrastructure code.