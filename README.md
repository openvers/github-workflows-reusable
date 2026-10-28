# Github Reusable Workflows

Reusable Workflows to reduce repetitive code blocks consistent across multiple repositories within the sim-parables org.

## External Terraform backend configuration

The `tf-dispatch-plan.yml`, `tf-dispatch-apply.yml`, and
`tf-dispatch-destroy.yml` workflows accept an optional `backend_config` secret.
Use it when your Terraform root module declares a partial backend:

```hcl
terraform {
  backend "remote" {}
}
```

Store the complete backend attributes in a GitHub Actions secret, equivalent to
your local gitignored `.tfbackend` file. Do not include enclosing `terraform` or
`backend` blocks in the secret:

```hcl
organization = "example-org"
workspaces {
  name = "databricks-fixture"
}
```

Example caller on this branch (ref names are case-sensitive):

```yaml
jobs:
  plan:
    uses: openvers/github-workflows-reusable/.github/workflows/tf-dispatch-plan.yml@AWSv1
    with:
      ACT_MODE: false
      working_directory: ./test
    secrets:
      # Existing authentication secrets remain unchanged.
      AWS_CLIENT_ID: ${{ secrets.AWS_CLIENT_ID }}
      AWS_CLIENT_SECRET: ${{ secrets.AWS_CLIENT_SECRET }}
      AWS_REGION: ${{ secrets.AWS_REGION }}
      AWS_ROLE_TO_ASSUME: ${{ secrets.AWS_ROLE_TO_ASSUME }}
      TF_API_TOKEN: ${{ secrets.TF_API_TOKEN }}
      backend_config: ${{ secrets.TF_BACKEND_DATABRICKS }}
```

Retain any other authentication secrets your caller already supplies. For a
service-account fixture, pass `${{ secrets.TF_BACKEND_SERVICE_ACCOUNT }}` as
`backend_config` instead. Use the same fixture's backend configuration for plan,
apply, and destroy. Backend configuration selects the organization/workspace;
authentication remains separate through the existing `TF_API_TOKEN` mechanism.
Do not put tokens or credentials in `backend_config`.

When supplied, the workflow writes the secret to a unique file under
`$RUNNER_TEMP` with owner-only permissions, runs
`terraform init -input=false -backend-config="$backend_file"`, and removes the
file when initialization exits, including on failure. The workflow does not
print the configuration or upload it as an artifact. When omitted or empty,
initialization remains `terraform init`, preserving existing callers.

`-backend-config` applies **only to `terraform init`**, not to `plan`, `apply`,
or `destroy`. These workflows do not add `-migrate-state`, `-force-copy`, or
`-reconfigure`; existing backend changes must be handled deliberately.

Terraform persists backend settings in `.terraform/` metadata and may include
them in saved plans, even after the temporary input file is removed. Do not
print, cache, or upload that metadata or saved plans containing sensitive
settings. Keep local `.tfbackend` files gitignored. If you introduce an
intermediate reusable workflow, it must declare the optional secret and
explicitly forward `backend_config: ${{ secrets.backend_config }}` to the
workflow that initializes Terraform.
