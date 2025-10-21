+++
date = '2025-10-19T20:40:02+11:00'
draft = false
title = 'external secrets operator | azure keyvault'
+++

In recent times I've been building multiple kubernetes clusters (1500+!) across my organisation, but one of the challenges I've faced has been how to manage secret.

It's one of those things that can be a real headache if you don't automate the pain away, particularly when it comes to a large number of clusters, having something automated gets around those pesky change control requirements

Enter, The External Secrets Operator (https://external-secrets.io)

**What does it do?**

The external secrets operator manages the lifecycle of secrets within a Kubernetes cluster. This could be a container registry pull credential, an application authentication mechanism or really anything else that you think is worth protecting

ESO integrates with a number of back ends such as Hashicorp Vault, Azure Key Vault and much much more: https://external-secrets.io/latest/provider/aws-secrets-manager/

**What problem does this solve?**

If your organisation is anything like mine, secret management at scale can become a real issue (Yes, We have seen secrets committed to git)

Managing secrets using ESO means you can quickly rotate compromised credentials, update expired credentials and keep secrets out of your code repository while still continuing to use your GitOps workflow.

**How does it work?**

ESO adds a new Custom Resource Definition to the cluster called an ExternalSecret. This object is a reference to an entry stored within your vault of choice. You can then set a reconcile threshold to periodicially query your vault for changes.

**Example:**

You have a one year lifecycle on a pull credential for Azure Container Registry. You receive a notification that in 30 days the credential is going to expire. You generate a new credential, update the value in the vault and the External Secrets Operator will automatically pull in the new value without any further user interaction

Consider the following ArgoCD Application to deploy PiHole. There is a reference to a secret called "pihole-password"
```bash {class="my-class" id="my-codeblock" lineNos=inline tabWidth=2}
declare a=1ffff
echo "$a"
exit
```