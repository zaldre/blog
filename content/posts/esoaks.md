+++
date = '2025-10-21T20:40:02+11:00'
draft = false
title = 'external secrets operator | azure keyvault'
+++
<img src="https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcRXA1jFeQlizkxcKRoUQtNHY6ZDkWlmwYCZ8w&s" width="30%">
In recent times I've been automating the builds of many Kubernetes clusters (1500+!) across my organisation, but one of the challenges I've faced has been how to manage <a href="https://kubernetes.io/docs/concepts/configuration/secret/">secrets</a>.

It's one of those things that can be a real headache if you don't automate the pain away, particularly when it comes to a large number of clusters


Enter <a href="https://external-secrets.io">The External Secrets Operator</a>

## What problem does this solve? ##

If your organisation is anything like mine, secret management at scale can become a real issue (Yes, we have historically seen secrets committed to git).

Managing secrets using ESO means you can quickly rotate compromised credentials, update expired credentials and keep secrets out of your code repository while still continuing to use your GitOps workflow. It also offers the ability to programmatically rotate the credentials periodically, improving security posture and minimising human error.


## What does it do? ##

The external secrets operator manages the lifecycle of secrets within a Kubernetes cluster. This could be a container registry pull credential, an API key, password or any other sensitive data worth protecting.

ESO integrates with a number of backends such as Hashicorp Vault, Azure Key Vault, AWS, etc.
https://external-secrets.io/latest/provider/azure-key-vault/


### How does it work? ###

ESO adds a new Custom Resource Definition to the cluster called an ExternalSecret. This object is a reference to an entry stored within your vault of choice. You can then set a reconcile threshold to periodically query your vault for changes.

### Scenario ###

You have a one year lifecycle on a pull credential for Azure Container Registry. You receive a notification that in 30 days the credential is going to expire. You generate a new credential, update the value in the vault and the External Secrets Operator will automatically pull in the new value without any further user interaction

### Example ###

Here's a registry credential I make available to my Jenkins instance. This makes the secret "regcred" available for my build agents to consume. The artifact is committed to git, but contains no secret data of its own. Only a reference point to the vault.

{{< code "eso/jenkins-externalsecret.yml" "yaml" >}}

## Creating the Azure Key Vault and Configuring Authentication

**Environment:** Talos v1.11.5 | Kubernetes v1.34.2

Before storing or retrieving secrets, you need an Azure Key Vault instance for your secrets, and configure how your cluster will authenticate to it.

---

### 1. Create the Azure Key Vault

If you do not already have a Key Vault, create one:

```bash
az keyvault create --name <VAULT_NAME> --resource-group <RESOURCE_GROUP> --location <LOCATION>
```

### 2. Authentication Methods

You can configure the External Secrets Operator to authenticate to Azure Key Vault in a variety of ways depending on where it's hosted. 
We won't touch on all the possibilities as they are well documented <a href="https://external-secrets.io/latest/provider/azure-key-vault/">here</a>.

However, here are the 2 primary ones we consume in my environment:

#### Example A: Using a Managed Identity (Recommended in AKS)

This is the one we use for both our AKS Clusters, and also for our Openshift clusters at the edge. A Workload Identity or Managed Identity is created with access to the vault and then we minimise the requirements for ongoing secrets management into the vault itself.


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

#### Example B: Using a Client ID/Secret

This is handy for clusters that may not easily integrate with Azure, It uses a more traditional client id/secret method. This means you will still have to maintain at least 1 secret value per SecretStore/ClusterSecretStore that you want to authenticate with.


First, create an App Registration/Enterprise application and grant it access to the vault with a read only role.

Once created, create a client secret and create a yaml definition with your values as shown.

{{< code "eso/azure-client-secret.yml" "yaml" >}}

Then configure your `SecretStore` or `ClusterSecretStore`:

EG: 

```yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: azure-keyvault-cluster-store
spec:
  provider:
    azurekv:
      tenantId: "9b05fdb9-40c0-4f4a-a654-386013ae7d6a"
      vaultUrl: "https://zaldre.vault.azure.net/"
      authSecretRef:
        clientId:
          name: azure-credentials
          key: client-id
          namespace: external-secrets
        clientSecret:
          name: azure-credentials
          key: client-secret
          namespace: external-secrets
```

For more details and advanced authentication options, see the [external-secrets documentation](https://external-secrets.io/latest/provider-azure-key-vault/).

Be sure your app registration/enterprise app, service principal, or managed identity has `set`, `get`, and `list` permissions on secrets:

```bash
az keyvault set-policy \
  --name <VAULT_NAME> \
  --secret-permissions get list set \
  --object-id <IDENTITY_OBJECT_ID>
```

---

### SecretStores - Where the objects are referenced ###

External Secrets Operator supports two main types of "secret stores". A SecretStore/ClusterSecretStore is a reference to your vault of choice within Kubernetes and acts as a logical permissions boundary for the secrets to be ingested.

- **SecretStore**: This is a namespaced resource. Secrets retrieved via a `SecretStore` are only available inside the same namespace the store is defined in.
- **ClusterSecretStore**: This is a cluster-scoped resource. Secrets from a `ClusterSecretStore` can be referenced by `ExternalSecrets` objects in any namespace, not just one.

ClusterSecretStore is especially useful when you want to reference the same secret value across multiple namespaces, such as a common container registry pull credential minimising the need to update the value in multiple places.


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
      tenantId: "abcd-1234-efgh-9876"
      vaultUrl: "https://zaldres-secret-vault.vault.azure.net/"
      authSecretRef:
        clientId:
          name: azure-credentials
          key: client-id
          namespace: external-secrets
        clientSecret:
          name: azure-credentials
          key: client-secret
          namespace: external-secrets
```

_SecretStore (namespace-scoped, only for one namespace):_
```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: jenkins-secretstore
  namespace: jenkins
spec:
  provider:
    azurekv:
      tenantId: "abcd-1234-efgh-9876"
      vaultUrl: "https://zaldres-secret-vault.vault.azure.net/"
      authSecretRef:
        clientId:
          name: azure-credentials
          key: client-id
          namespace: jenkins
        clientSecret:
          name: azure-credentials
          key: client-secret
          namespace: jenkins
```

The important takeaway from this is you now only manage a much smaller subset of secrets - the one secret into the Vault(s). Everything else is abstracted away.

Updating a secret can now be done easily by updating the value directly in Azure Key Vault—either using the Azure CLI or the Azure Portal web interface.

---

## Managing and updating secrets ##

Once you've set up authentication into the vault, you'll then want to manage and update the secrets contained in the vault for consumption in your cluster. There are a few ways to approach it.

The general rule of thumb is that you want to use the "az" cli commands for files, and the webUI for strings

**A. Using Azure CLI:**

For example, to update (set) a secret value from the command line:
```bash
az keyvault secret set --vault-name <VAULT_NAME> --name <SECRET_NAME> --value <NEW_SECRET_VALUE>

az keyvault secret set --vault-name <VAULT_NAME> --name <SECRET_NAME> --file <FILE_NAME_WITH_ABSOLUTE_PATH>
```

**B. Using the Azure Portal (GUI):**

1. Go to [https://portal.azure.com](https://portal.azure.com) and open your Azure Key Vault.
2. In the left menu, select **Secrets**.
3. Select the secret you want to update.
4. Click **+ New Version**, enter the new value, and click **Create**.

After updating the secret value in Key Vault (via CLI or the Portal), the External Secret will automatically sync the change to your Kubernetes secret on the next refresh.

---
That's all for this post. I hope you found it useful. If there's any information you want clarified, thoughts, or opinions, please do let me know.

<a href="mailto:zaldre@zaldre.com">zaldre@zaldre.com</a>

---