# Github Reusable Workflows

Reusable Workflows to reduce repetitive code blocks consistent across multiple repositories within the sim-parables org.

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
