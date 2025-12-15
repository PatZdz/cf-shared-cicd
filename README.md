# cf-shared-cicd

This repository hosts a reusable GitHub Action that standardises Terraform CI
for Azure-based projects. It is consumed from other repositories (for example
`cf_azure_static_website` and `cf_container_app`) to run a consistent pipeline
for validation, planning and applying infrastructure changes.

## Overview

The core of this repository is the composite action
`/.github/actions/terraform-pipeline/action.yml`. It performs the following
steps against an AzureRM backend authenticated via OIDC:

- check out the repository
- log in to Azure with a "plan" identity
- install a specific Terraform version
- initialise the AzureRM backend using remote state in Azure Storage
- run `terraform fmt -check`, `terraform validate` and `tflint`
- create a Terraform plan (`tfplan`)
- optionally re-authenticate with an "apply" identity and run
  `terraform apply tfplan`

Using a shared action keeps Terraform workflows in individual repositories
small, consistent and easy to maintain.

## Inputs

The action exposes the following inputs (see
`.github/actions/terraform-pipeline/action.yml`):

- `working-directory` (string, default `.`): relative path to the directory
  containing Terraform configuration.
- `terraform-version` (string, default `1.13.0`): Terraform CLI version to
  install via `hashicorp/setup-terraform`.
- `apply` (string/boolean, default `false`): when set to `true`, the action
  will run both plan and apply. When `false`, only validation and plan are
  executed.
- `extra-env-vars` (multiline string, optional): additional `KEY=VALUE` pairs
  appended to the environment. This is commonly used to pass extra `TF_VAR_*`
  variables such as container image tags.

## Required environment variables

Callers must provide the following environment variables when using the
action. They are typically mapped from repository secrets in the calling
workflow:

- `AZURE_TENANT_ID`: Azure AD tenant ID.
- `AZURE_SUBSCRIPTION_ID`: target subscription for the deployment.
- `AZURE_CLIENT_ID_PLAN`: client ID of the workload identity used for
  planning and validation.
- `AZURE_CLIENT_ID_APPLY`: client ID of the workload identity used for
  applying changes.
- `TF_STATE_RG`: name of the resource group that holds the Terraform state
  storage account.
- `TF_STATE_STORAGE_ACCOUNT`: name of the storage account used for the
  Terraform state backend.
- `TF_STATE_CONTAINER`: name of the blob container that stores state files.
- `TF_STATE_KEY`: key (blob name) of the state file within the container.
- `SUBSCRIPTION_ID` (optional but recommended): subscription ID propagated to
  Terraform as `TF_VAR_subscription_id` so modules can consume it as a
  regular input variable.

## Usage example

In a consuming repository, the action can be used from a workflow step like
this:

```yaml
- uses: PatZdz/cf-shared-cicd/.github/actions/terraform-pipeline@main
  with:
    working-directory: infra
    terraform-version: 1.13.0
    apply: ${{ inputs.apply == true }}
    extra-env-vars: |
      TF_VAR_container_image=cfacrpatryktf.azurecr.io/cf-hello-app:${{ github.sha }}
  env:
    AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
    AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
    AZURE_CLIENT_ID_PLAN: ${{ secrets.AZURE_CLIENT_ID_PLAN }}
    AZURE_CLIENT_ID_APPLY: ${{ secrets.AZURE_CLIENT_ID_APPLY }}
    TF_STATE_RG: ${{ secrets.TF_STATE_RG }}
    TF_STATE_STORAGE_ACCOUNT: ${{ secrets.TF_STATE_STORAGE_ACCOUNT }}
    TF_STATE_CONTAINER: ${{ secrets.TF_STATE_CONTAINER }}
    TF_STATE_KEY: ${{ secrets.TF_STATE_KEY }}
    SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

## Best practices

- Pin the action to a specific ref (tag or commit SHA) instead of `main` in
  production workloads to avoid accidental behaviour changes.
- Keep Terraform versions in the calling workflow aligned with the
  `required_version` constraint declared in your Terraform configuration.
- Use separate identities for plan and apply to follow the principle of
  least privilege.
- Reuse the same remote backend across related projects when you want a
  single source of truth for state, or separate it per environment/project
  when isolation is more important.

