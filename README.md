# Github Reusable Workflows

Reusable Workflows to reduce repetitive code blocks consistent across multiple repositories within the sim-parables org.

## External Terraform backend configuration

`tf-dispatch-plan.yml`, `tf-reusable-auth.yml`, `tf-dispatch-apply.yml`, and
`tf-dispatch-destroy.yml` accept the optional `backend_config` workflow secret.
Use it with partial backend configuration committed in your Terraform fixture:

```hcl
terraform {
  backend "remote" {}
}
```

Store the complete backend configuration as HCL in a GitHub Actions secret, just
as you would in a local, gitignored `.tfbackend` file. Do not include the outer
`terraform` or `backend` block:

```hcl
organization = "your-organization"
workspaces {
  name = "your-databricks-workspace"
}
```

Add the mapping to your existing caller (other required inputs and authentication
secrets must still be supplied):

```yaml
jobs:
  plan:
    uses: openvers/github-workflows-reusable/.github/workflows/tf-dispatch-plan.yml@AzureDatabricksV1
    with:
      ACT_MODE: false
      DATABRICKS_ADMINISTRATOR: ${{ vars.DATABRICKS_ADMINISTRATOR }}
      working_directory: ./test/databricks
    secrets:
      # Keep all existing Azure and Databricks authentication secret mappings.
      TF_API_TOKEN: ${{ secrets.TF_API_TOKEN }}
      backend_config: ${{ secrets.TF_BACKEND_DATABRICKS }}
```

For the service-account fixture, map
`backend_config: ${{ secrets.TF_BACKEND_SERVICE_ACCOUNT }}` instead. Supply the
appropriate mapping to apply, destroy, and auth workflow calls as well. Backend
selection remains separate from authentication through the existing
`TF_API_TOKEN` mechanism; do not put credentials in backend configuration.

When nonempty, the secret is written without printing its contents to a temporary
file under `$RUNNER_TEMP` with owner-only permissions. Initialization uses
`terraform init -input=false -backend-config="$backend_file"`, and the file is
removed when initialization finishes, including on failure. The file is not
uploaded as an artifact. Omitting the secret (or supplying an empty value)
preserves the existing `terraform init` behavior. No automatic state migration or
forced reconfiguration is performed.

`-backend-config` applies **only to `terraform init`**, not to `plan`, `apply`, or
`destroy`; those commands use the initialized backend. Terraform can retain
backend settings in `.terraform` metadata and saved plans, so do not print or
upload those files if they contain sensitive configuration. Keep local
`.tfbackend` files gitignored.

If your caller wraps these workflows in another reusable workflow, declare the
optional `backend_config` secret there and explicitly forward
`backend_config: ${{ secrets.backend_config }}` in the nested call's `secrets`
mapping at every boundary.

## Local workflow execution with Nektos act

Run these reusable Terraform workflows locally with [Nektos act](https://nektosact.com/) by setting `ACT_MODE=true` in the caller workflow inputs. With `ACT_MODE=false`, Azure authentication continues to use GitHub OIDC.

Local Azure authentication supports either a service principal client secret or a certificate. A client secret takes precedence when both are supplied. Pass the selected secrets to the reusable workflow through the caller's `secrets` mapping or `secrets: inherit` where supported.

The examples below cover Azure authentication only. Continue supplying the existing Databricks secrets and other inputs required by the selected workflow; Databricks authentication is unchanged.

### Client secret authentication

Example local `.secrets` file:

```env
AZURE_CLIENT_ID=<application-client-id>
AZURE_CLIENT_SECRET=<client-secret>
AZURE_SUBSCRIPTION_ID=<subscription-id>
AZURE_TENANT_ID=<tenant-id>
TF_API_TOKEN=<terraform-cloud-token>
```

```bash
act --secret-file .secrets
```

### Certificate authentication

Register the certificate's public key with the Azure service principal's application. Certificate authentication requires a PKCS#12 bundle, such as a `.pfx` file, containing the certificate and private key.

For example, store the bundle at `~/creds/azure/service-account.pfx`. Omit `AZURE_CLIENT_SECRET` to select certificate authentication, and configure the local `.secrets` file with the path visible inside the job container:

```env
AZURE_CLIENT_ID=<application-client-id>
AZURE_CLIENT_CERTIFICATE_PATH=/creds/azure/service-account.pfx
AZURE_CLIENT_CERTIFICATE_PASSWORD=<certificate-password>
AZURE_SUBSCRIPTION_ID=<subscription-id>
AZURE_TENANT_ID=<tenant-id>
TF_API_TOKEN=<terraform-cloud-token>
```

Mount the certificate directory read-only into the `act` container:

```bash
act \
  --secret-file .secrets \
  --container-options "-v $HOME/creds/azure:/creds/azure:ro"
```

The mounted path must match `AZURE_CLIENT_CERTIFICATE_PATH`. Leave the password unset for an unencrypted bundle. When `AZURE_CLIENT_SECRET` is supplied, the certificate path and password are ignored.

Keep `.secrets`, certificate bundles, and private keys out of version control.
