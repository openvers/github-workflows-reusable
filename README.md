# Github Reusable Workflows

Reusable Workflows to reduce repetitive code blocks consistent across multiple repositories within the sim-parables org.

## External Terraform backend configuration

On `AzureV1`, `tf-dispatch-plan.yml`, `tf-reusable-auth.yml`, `tf-dispatch-apply.yml`, and `tf-dispatch-destroy.yml` accept an optional `backend_config` secret containing backend settings in HCL format. This supports partial configuration such as:

```hcl
terraform {
  backend "remote" {}
}
```

Store the complete backend settings (the contents of your local, gitignored `.tfbackend` file, without a `terraform` or `backend` wrapper) in a GitHub Actions secret. For example, `TF_BACKEND_DATABRICKS` could contain:

```hcl
organization = "example-organization"
workspaces {
  name = "databricks-fixture"
}
```

Pass the secret alongside the existing authentication secrets:

```yaml
jobs:
  plan:
    uses: openvers/github-workflows-reusable/.github/workflows/tf-dispatch-plan.yml@AzureV1
    with:
      ACT_MODE: false
      working_directory: ./test/databricks
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      TF_API_TOKEN: ${{ secrets.TF_API_TOKEN }}
      backend_config: ${{ secrets.TF_BACKEND_DATABRICKS }}
```

For the service-account fixture, pass `backend_config: ${{ secrets.TF_BACKEND_SERVICE_ACCOUNT }}` to `tf-reusable-auth.yml` with that fixture's working directory. Supply the appropriate backend secret to each plan, apply, or destroy invocation as well. Authentication remains separate through the existing `TF_API_TOKEN` mechanism; do not put API tokens in the backend configuration.

When supplied, the workflow writes the settings to a mode-`0600` temporary file under `$RUNNER_TEMP`, initializes with `terraform init -input=false -backend-config="$backend_file"`, and removes the file when initialization exits, including on failure. The workflow does not print the configuration or upload it as an artifact. Without the secret (or with an empty value), initialization remains plain `terraform init`.

`-backend-config` applies **only to `terraform init`**, not to plan, apply, or destroy commands. Those commands use the initialized backend. No automatic state migration or forced reconfiguration is performed. Terraform can cache backend settings in `.terraform` and saved plan files; do not commit or upload those files when they contain sensitive settings.

If a caller is itself a reusable workflow, declare the optional secret in its `on.workflow_call.secrets` and explicitly forward it at every nested call:

```yaml
on:
  workflow_call:
    secrets:
      backend_config:
        description: Optional Terraform backend configuration in HCL format
        required: false

# In the nested job's existing secrets mapping, alongside authentication secrets:
# backend_config: ${{ secrets.backend_config }}
```

## Local workflow execution with Nektos act

These reusable Terraform workflows support local execution with [Nektos act](https://nektosact.com/) by setting `ACT_MODE=true` in the caller workflow inputs.

For Azure authentication, local runs support either service principal client secret authentication or service principal certificate authentication. Client secret authentication takes precedence when both client secret and certificate values are provided.

### Client secret authentication

Example `.secrets` file:

```env
AZURE_CLIENT_ID=<application-client-id>
AZURE_CLIENT_SECRET=<client-secret>
AZURE_SUBSCRIPTION_ID=<subscription-id>
AZURE_TENANT_ID=<tenant-id>
TF_API_TOKEN=<terraform-cloud-token>
```

### Certificate authentication

Certificate authentication requires a PKCS#12 certificate bundle, such as a `.pfx` file.

Example local certificate path:

```bash
~/creds/azure/service-account.pfx
```

When running `act`, mount the certificate directory into the job container and set the certificate path to the mounted container path.

Example `.secrets` file:

```env
AZURE_CLIENT_ID=<application-client-id>
AZURE_CLIENT_CERTIFICATE_PATH=/creds/azure/service-account.pfx
AZURE_CLIENT_CERTIFICATE_PASSWORD=<certificate-password>
AZURE_SUBSCRIPTION_ID=<subscription-id>
AZURE_TENANT_ID=<tenant-id>
TF_API_TOKEN=<terraform-cloud-token>
```

Example `act` command:

```bash
act \
  --secret-file .secrets \
  --container-options "-v $HOME/creds/azure:/creds/azure:ro"
```

The mounted path in `AZURE_CLIENT_CERTIFICATE_PATH` must match the path visible inside the `act` container. If `AZURE_CLIENT_SECRET` is set, certificate authentication variables are ignored by the reusable workflows.
