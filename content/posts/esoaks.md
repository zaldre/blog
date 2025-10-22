+++
date = '2025-10-19T20:40:02+11:00'
draft = false
title = 'external secrets operator | azure keyvault'
+++

In recent times I've been automating the builds of many Kubernetes clusters (1500+!) across my organisation, but one of the challenges I've faced has been how to manage <a href="https://kubernetes.io/docs/concepts/configuration/secret/">secrets</a>.

It's one of those things that can be a real headache if you don't automate the pain away, particularly when it comes to a large number of clusters


Enter <a href="https://external-secrets.io">The External Secrets Operator</a>

## What problem does this solve? ##

If your organisation is anything like mine, secret management at scale can become a real issue (Yes, We have historically seen secrets committed to git)

Managing secrets using ESO means you can quickly rotate compromised credentials, update expired credentials and keep secrets out of your code repository while still continuing to use your GitOps workflow. It also offers the ability to programatically rotate the credentials periodically, improving security posture and minimising human error.


## What does it do? ##

The external secrets operator manages the lifecycle of secrets within a Kubernetes cluster. This could be a container registry pull credential, an API key, or any other sensitive data worth protecting.

ESO integrates with a number of back ends such as Hashicorp Vault, Azure Key Vault, AWS, etc 
https://external-secrets.io/latest/provider/azure-key-vault/


### How does it work? ###

ESO adds a new Custom Resource Definition to the cluster called an ExternalSecret. This object is a reference to an entry stored within your vault of choice. You can then set a reconcile threshold to periodicially query your vault for changes.

### Scenario ###

You have a one year lifecycle on a pull credential for Azure Container Registry. You receive a notification that in 30 days the credential is going to expire. You generate a new credential, update the value in the vault and the External Secrets Operator will automatically pull in the new value without any further user interaction

### Example ###

Heres a registry credential I make available to my Jenkins instance. This makes the secret "regcred" available for my build agents to consume

{{< code "eso/jenkins-externalsecret.yml" "yaml" >}}

### ClusterSecretStore vs SecretStore ###

External Secrets Operator supports two main types of "secret stores" as external secret providers in Kubernetes. A SecretStore/ClusterSecret store is a reference to your vault of choice within Kubernetes and acts as a logical permissions boundary for the secrest to be ingested.

- **SecretStore**: This is a namespaced resource. Secrets retrieved via a `SecretStore` are only available inside the same namespace the store is defined in.
- **ClusterSecretStore**: This is a cluster-scoped resource. Secrets from a `ClusterSecretStore` can be referenced by `ExternalSecrets` objects in any namespace, not just one.

ClusterSecretStore is especially useful when you want to reference the same secret value across multiple namespaces, such as a common container registry pull credential.


Example YAML fragments:

_ClusterSecretStore (cluster-scoped, available to all namespaces):_
```yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: azure-keyvault-cluster-store
spec:
  provider:
    azurekv:
      # Azure Key Vault config here
```

_SecretStore (namespace-scoped, only for one namespace):_
```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: azure-keyvault-ns-store
  namespace: my-namespace
spec:
  provider:
    azurekv:
      # Azure Key Vault config here
```

---

#### How to create and store a docker registry secret in Azure Key Vault ####

1. **Create a docker registry secret as a file (base64 encoded json):**

```bash
kubectl create secret docker-registry regcred \
  --docker-server=<ACR_LOGIN_SERVER> \
  --docker-username=<ACR_USERNAME> \
  --docker-password=<ACR_PASSWORD> \
  --docker-email=<YOUR_EMAIL> \
  --dry-run=client -o json | \
jq -r '.data[".dockerconfigjson"]' > docker-config-json.b64
```

2. **Store the secret in Azure Key Vault**

First, decode the base64 secret to get the JSON for Key Vault:
```bash
base64 -d docker-config-json.b64 > docker-config.json
```

Then upload to Azure Key Vault (replace `<VAULT_NAME>` and `<KEY_NAME>`):

```bash
az keyvault secret set \
  --vault-name <VAULT_NAME> \
  --name docker-config-json \
  --file docker-config.json
```

3. **(Optional) Confirm upload**
```bash
az keyvault secret show --vault-name <VAULT_NAME> --name docker-config-json
```

Your ExternalSecret configuration can now reference the `docker-config-json` object stored in Azure Key Vault!


---

## Creating the Azure Key Vault and Configuring Authentication

Before storing or retrieving secrets, you need an Azure Key Vault instance for your secrets, and configure how your cluster will authenticate to it.

### 1. Create the Azure Key Vault

If you do not already have a Key Vault, create one:

```bash
az keyvault create --name <VAULT_NAME> --resource-group <RESOURCE_GROUP> --location <LOCATION>
```

Be sure your user, service principal, or managed identity has `set`, `get`, and `list` permissions on secrets:

```bash
az keyvault set-policy \
  --name <VAULT_NAME> \
  --secret-permissions get list set \
  --object-id <IDENTITY_OBJECT_ID>
```

---

### 2. Authentication Methods

You can configure the External Secrets Operator to authenticate to Azure Key Vault in two main ways:

#### Example A: Using a Managed Identity (Recommended in AKS)

Configure your `SecretStore` or `ClusterSecretStore` to use a managed identity assigned to your AKS cluster/node pool.

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: azure-keyvault-ns-store
  namespace: my-namespace
spec:
  provider:
    azurekv:
      tenantId: <AZURE_TENANT_ID>
      vaultUrl: https://<VAULT_NAME>.vault.azure.net/
      authType: WorkloadIdentity # or ManagedIdentity for older setups
      # identityId is optional, specify if using user-assigned identity
      # identityId: <MANAGED_IDENTITY_CLIENT_ID>
```

**Notes:**  
- Use `authType: WorkloadIdentity` for clusters using Azure AD Workload Identity  
- Use `authType: ManagedIdentity` if using legacy AKS managed identities  
- Set `identityId` to the client ID of your user-assigned managed identity if not relying on the default

#### Example B: Using a Client ID/Secret (Service Principal)

If you want to use an Azure AD application (service principal) for authentication:

First, create a Service Principal and give it access:

```bash
az ad sp create-for-rbac --name <SP_NAME> --sdk-auth
```

Assign the necessary Key Vault secret permissions as done above, using the service principal's object ID.

Then configure your `SecretStore` or `ClusterSecretStore`:

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: azure-keyvault-ns-store
  namespace: my-namespace
spec:
  provider:
    azurekv:
      tenantId: <AZURE_TENANT_ID>
      vaultUrl: https://<VAULT_NAME>.vault.azure.net/
      authType: ServicePrincipal
      clientId: <CLIENT_ID>
      clientSecret:
        # reference to Kubernetes secret containing client secret value
        name: azure-sp-secret
        key: client-secret
```

You'll need to create the referenced Kubernetes secret first:

```bash
kubectl create secret generic azure-sp-secret \
  --from-literal=client-secret=<CLIENT_SECRET> \
  -n my-namespace
```

---

For more details and advanced authentication options, see the [external-secrets documentation](https://external-secrets.io/latest/provider-azure-key-vault/).


The important take away from this is you now only manage a much smaller subset of secrets, The one secret into the Vault(s). Everything else is abstracted away.

Updating a secret can now be done easily by updating the value directly in Azure Key Vault—either using the Azure CLI or the Azure Portal web interface.

**A. Using Azure CLI:**

For example, to update (set) a secret value from the command line:
```bash
az keyvault secret set --vault-name <VAULT_NAME> --name <SECRET_NAME> --value '<NEW_SECRET_VALUE>'
```

**B. Using the Azure Portal (GUI):**

1. Go to [https://portal.azure.com](https://portal.azure.com) and open your Azure Key Vault.
2. In the left menu, select **Secrets**.
3. Select the secret you want to update.
4. Click **+ New Version**, enter the new value, and click **Create**.

After updating the secret value in Key Vault (via CLI or the Portal), the External Secret will automatically sync the change to your Kubernetes secret on the next refresh.

---