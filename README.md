# Unlock TF State Github Action

This action unlocks locked terraform state.

This action can download private modules from the Optimizely's Github Org if a private key is provided (more information below).

### Features

In more details, this action will:

* Authenticate with GCP either using a SA json key file or Workload Identity Federation (recommended)
* Install terraform dependencies (using [hasicorp-setup-terraform github action](https://github.com/marketplace/actions/hashicorp-setup-terraform))
* Initialize the specified directory as a working directory (terraform init)
* Unlocks the state with the provided lock-id

# Usage

### Basic example of this action in a workflow

```
# https://github.com/optimizely/experimentation-golden-path/blob/main/golden_path/pipelines_and_gitflow/extra-1-unlock-state-file.md
name: "Unblock Terraform Deployments"
on:
  workflow_dispatch:
    inputs:
      env:
        required: true
        type: choice
        description: Environment
        options:
          - inte
          - prep
          - prod
      lock_id:
        required: true
        type: string
        description: Lock Id
jobs:
  unblock:
    runs-on: ubuntu-latest
    permissions:
      contents: 'read'
      id-token: 'write'
    steps:
      - uses: actions/checkout@v4
      - name: Terraform Unlock TF State
        uses: optimizely/unlock-tf-state-github-action@main
        with:
          terraform_lock_id: ${{ inputs.lock_id }}
          terraform_directory: "infra/${{ github.event.inputs.env }}"
          github_token: ${{ secrets.GITHUB_TOKEN }}
          ssh_private_key: ${{ secrets.OPTI_EXP_MINION_TERRAFORM_PLAN_SSH_KEY }} ### This is a secret defined at the org level in the Optimizely org
```

## Authentication with Cloud providers

This action might be used to perform actions in a cloud environment. For this, it is required to configure authentication in the GH Actions runners used for this action.

### GCP

This action uses [Google's auth github action](https://github.com/google-github-actions/auth/tree/main) to authenticate this runner in GCP. This action supports two ways to authenticate against GCP via Google's github action:

#### Authenticating via Workload Identity Federation

Workload Identity Federation is recommended over Service Account Keys as it obviates the need to export a long-lived credential and establishes a trust delegation relationship between a particular GitHub Actions workflow invocation and permissions on Google Cloud.
If *gcloud_sa_json_key_file* is not specified, this is the authentication method that will be used.

```
    steps:
      - name: Terraform Unlock TF State
        uses: optimizely/unlock-tf-state-github-action@main
        with:
          terraform_lock_id: ${{ inputs.lock_id }}
          terraform_directory: "infra/dev"
          github_token: ${{ secrets.GITHUB_TOKEN }}
          ssh_private_key: ${{ secrets.TERRAFORM_PLAN_SSH_KEY }}
```

In the previous example, there was no need to configure anything related to Workload Identity Federation because the required settings are already configured as defaults. In some scenarios a different workload_identity_pool or service_account might be required, so it is also possible to overwrite these defaults:

```
    steps:
      - name: Terraform Unlock TF State
        uses: optimizely/unlock-tf-state-github-action@main
        with:
          terraform_lock_id: ${{ inputs.lock_id }}
          terraform_directory: "infra/dev"
          github_token: ${{ secrets.GITHUB_TOKEN }}
          ssh_private_key: ${{ secrets.TERRAFORM_PLAN_SSH_KEY }}
          workload_identity_provider: 'projects/599682587101/locations/global/workloadIdentityPools/github-actions-workflows/providers/github-actions-workflows'
          service_account: 'github-actions-workflows@optimizely-iac-bootstrap.iam.gserviceaccount.com'
```

GCP's Workload Identity Federation for Optimizely's Github is configured [here](https://github.com/Optimizely-Experimentation-RE/optimizely-gcp-organization/blob/main/gh_workload_identity_federation.tf)

#### Authenticating via Service Account Key JSON

If *gcloud_sa_json_key_file* variable is defined, this is the authentication method that will be used. A SA json key file is required as a GH actions secret, and then configured in the action. This is NOT recommended and it is only an option for legacy reasons.

```
    steps:
      - name: Terraform Unlock TF State
        uses: optimizely/unlock-tf-state-github-action@main
        with:
          terraform_lock_id: ${{ inputs.lock_id }}
          terraform_directory: "infra/dev"
          gcloud_sa_json_key_file: ${{secrets.TEST_GITHUB_ACTIONS_SA_KEY}}
          github_token: ${{ secrets.GITHUB_TOKEN }}
          ssh_private_key: ${{ secrets.TERRAFORM_PLAN_SSH_KEY }}
```

### Full example

This example requires a secret called *TEST_GITHUB_ACTIONS_SA_KEY* containing the json of a GCloud SA key.

```
name: "Unblock Terraform Deployments"
on:
  workflow_dispatch:
    inputs:
      env:
        required: true
        type: choice
        description: Environment
        options:
          - inte
          - prep
          - prod
      lock_id:
        required: true
        type: string
        description: Lock Id
jobs:
  unblock:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: optimizely/unlock-tf-state-github-action@main
        with:
          terraform_lock_id: ${{ github.event.inputs.lock_id }}
          terraform_directory: "infra/${{ github.event.inputs.env }}"
          gcloud_sa_json_key_file: ${{secrets.TEST_GITHUB_ACTIONS_SA_KEY}}
          github_token: ${{ secrets.GITHUB_TOKEN }}
          ssh_private_key: ${{ secrets.TERRAFORM_PLAN_SSH_KEY }}
```

# Inputs

The action supports the following inputs:

* terraform_directory - Path to the terraform root module for which the execution plan will be generated
* gcloud_sa_json_key_file - JSON key for GCP service account that this runner will use. If specified, Workload Identity Federation will NOT be used. Using a SA json key file is not recommended, and it is only supported here for legacy reasons
* workload_identity_provider - The full identifier of the Workload Identity Provider, including the project number, pool name, and provider name. This is only used if gcloud_sa_json_key_file is not specified. Defaults to Optimizely's settings
* service_account - Email address or unique identifier of the Google Cloud service account for which to generate credentials as part of the Workload Identity Federation setup. This is only used if gcloud_sa_json_key_file is not specified. Defaults to Optimizely's settings
* github-token - This token will be used to add a message to the PRs containing the Terraform Plan output. It is recommended to use the [token offered by the runner](https://docs.github.com/en/actions/security-guides/automatic-token-authentication#about-the-github_token-secret), but this requires *pull-requests: write* permission (see full example).
* terraform_lock_id - Terraform lock id which is used to unlock the state.
* ssh_private_key (optional) - SSH private key used to download terraform private modules hosted in the Optimizely github organization (if any).
