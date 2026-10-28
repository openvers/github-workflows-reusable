# Github Reusable Workflows

Reusable Workflows to reduce repetitive code blocks consistent across multiple repositories within the sim-parables org.

## External Terraform backend configuration (GCPv1)

The `tf-dispatch-plan.yml`, `tf-dispatch-apply.yml`, and
`tf-dispatch-destroy.yml` workflows accept an optional `backend_config` secret.
This supports fixtures with partial backend configuration:

```hcl
terraform {
  backend "remote" {}
}
```

Store the complete backend settings as the secret's HCL contents, without a
surrounding `terraform` or `backend` block, just as in a local gitignored
`.tfbackend` file:

```hcl
organization = "example-org"
workspaces {
  name = "databricks-fixture"
}
```

Caller example (existing authentication secrets remain unchanged):

```yaml
jobs:
  plan:
    uses: openvers/github-workflows-reusable/.github/workflows/tf-dispatch-plan.yml@GCPv1
    with:
      ACT_MODE: false
      working_directory: ./test
    secrets:
      GOOGLE_IMPERSONATE_SERVICE_ACCOUNT_EMAIL: ${{ secrets.GOOGLE_IMPERSONATE_SERVICE_ACCOUNT_EMAIL }}
      GOOGLE_PROJECT: ${{ secrets.GOOGLE_PROJECT }}
      GOOGLE_WORKLOAD_IDENTITY_PROVIDER: ${{ secrets.GOOGLE_WORKLOAD_IDENTITY_PROVIDER }}
      TF_API_TOKEN: ${{ secrets.TF_API_TOKEN }}
      backend_config: ${{ secrets.TF_BACKEND_DATABRICKS }}
```

Use `backend_config: ${{ secrets.TF_BACKEND_SERVICE_ACCOUNT }}` for the
service-account fixture's separate backend. Pass the same fixture's backend
configuration to its plan, apply, and destroy calls. Authentication remains
separate through the existing `TF_API_TOKEN` mechanism; do not put tokens in
`backend_config`.

When supplied, the workflow writes the configuration to a mode-`0600` temporary
file under `$RUNNER_TEMP`, initializes with
`terraform init -input=false -backend-config="$backend_file"`, and removes the
file when the initialization step exits, including on failure. The workflow
does not print the configuration or upload it as an artifact. When omitted or
empty, it runs the existing `terraform init` unchanged. No state migration or
forced reconfiguration flags are added.

`-backend-config` applies **only to `terraform init`**, not to `plan`, `apply`,
or `destroy`; subsequent commands use the initialized backend. Terraform caches
backend settings in `.terraform` and saved plans, so keep these out of version
control and avoid uploading them as artifacts containing sensitive settings.

There are no nested reusable-workflow calls in these GCPv1 dispatch workflows.
If you wrap them in another reusable workflow, declare `backend_config` as an
optional `workflow_call` secret there and explicitly forward it in the nested
job's `secrets` mapping:

```yaml
backend_config: ${{ secrets.backend_config }}
```
