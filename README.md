# Github Reusable Workflows

Reusable Workflows to reduce repetitive code blocks consistent across multiple repositories within the sim-parables org.

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
