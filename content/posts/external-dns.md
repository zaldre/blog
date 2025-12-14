+++
date = '2025-12-15T8:00:00+11:00'
draft = false
title = 'external-dns | automating dns for pihole and cloudflare'
+++
<img src="https://github.com/kubernetes-sigs/external-dns/raw/master/docs/img/external-dns.png" width="20%">
Managing DNS records manually is one of those tasks that starts simple but quickly becomes a pain in the neck. Every time you deploy a new service, create a new route, or change an IP address, you're manually logging into your DNS provider's web interface, clicking through menus, and hoping you don't make a typo.

If you're running Kubernetes with Gateway API or Ingress resources, you're probably already annotating your routes with hostnames. Wouldn't it be nice if those DNS records just... appeared? 

Have you considered welcoming <a href="https://github.com/kubernetes-sigs/external-dns">ExternalDNS</a> into your life?

## What problem does this solve? ##

ExternalDNS automatically synchronizes Ingress resources and Gateway API routes with DNS providers. When you create an HTTPRoute/Ingress resource, ExternalDNS watches for that change and creates (or updates) the corresponding DNS record in your provider of choice.

This means:
- **No more manual DNS management** - Create a route, get a DNS record. It's that simple.
- **Consistent naming** - Your DNS records match your Kubernetes resources automatically
- **Self-healing** - If a record gets deleted or changed outside of Kubernetes, ExternalDNS will reconcile it back
- **Multi-provider support** - Works with Cloudflare, AWS Route53, Azure DNS, Google Cloud DNS, and many others, including Pi-hole for local DNS

In my home lab, I run a dual-DNS setup: Pi-hole for internal DNS resolution (so I can access services via `*.zaldre.com` on my local network) and Cloudflare for external DNS (so the world can reach my services). ExternalDNS handles both automatically.

## How does it work? ##

ExternalDNS runs as a deployment in your cluster and watches for changes to Kubernetes resources. When it detects a new Ingress or HTTPRoute it:

1. Extracts the hostname from the definition
2. Determines the target IP address (from the service, load balancer, or a default)
3. Creates or updates the DNS record in your configured provider
4. Periodically reconciles to ensure the DNS records match what's in Kubernetes

The magic happens through the hostname definition in your Ingress/HTTPRoute. ExternalDNS picks it up and will sync this periodically to your DNS provider of choice.

For providers that support ownership tracking (like Cloudflare), it also creates TXT records to track which records it manages, preventing conflicts with manually created records.

## My implementation: Dual provider setup ##

**Environment:** Talos v1.11.5 | Kubernetes v1.34.2

I'm running two separate ExternalDNS deployments - one for Pi-hole (internal only) and one for Cloudflare (internet facing). This gives me flexibility to have split brain DNS depending on whether I'm at home or not.

I also selectivly annotate the services with something that indicates the service is internet facing for the Cloudflare instance to pick up and manage.

EG: (external-dns.alpha.kubernetes.io/hostname: stats.zaldre.com)

### Architecture overview ###

Pre-requisites:
- **Envoy Gateway**: My Gateway API configuration. I cover this in another post <a href="https://zaldre.com/posts/gateway">here</a>. You can also use ingress resources if you prefer that.

- **Pi-hole ExternalDNS**: Watches for routes and creates A records pointing to my internal load balancer IP (10.0.0.180)
- **Cloudflare ExternalDNS**: Watches for routes and creates A records pointing to my external IP (1.2.3.4)


Both deployments run in the `external-dns` namespace and use separate service accounts with appropriate RBAC permissions.

### Setting up the namespace ###

First, create a namespace for ExternalDNS:

{{< code "external-dns/external-dns-namespace.yml" "yaml" >}}

### Pi-hole configuration ###

Pi-hole is my internal DNS server, so I want all internal services to resolve via Pi-hole. The Pi-hole ExternalDNS deployment is configured to:

- Watch for Gateway HTTPRoutes, TCPRoutes, and Ingress resources
- Use the Pi-hole API v6 to create DNS records
- Point all records to my internal load balancer IP (10.0.0.180)
- Use `--registry=noop` because Pi-hole doesn't support TXT ownership records

**Service Account:**

{{< code "external-dns/external-dns-pihole-serviceaccount.yml" "yaml" >}}

**ClusterRole:**

{{< code "external-dns/external-dns-pihole-clusterrole.yml" "yaml" >}}

**ClusterRoleBinding:**

{{< code "external-dns/external-dns-pihole-clusterrolebinding.yml" "yaml" >}}

**Deployment:**

{{< code "external-dns/external-dns-pihole-deployment.yml" "yaml" >}}

A few important notes about the Pi-hole configuration:

- `--registry=noop`: Pi-hole only supports A/AAAA/CNAME records, so there's no mechanism for ownership tracking. This flag prevents ExternalDNS from trying to create TXT records and spamming your logs with warnings.
- `--policy=sync`: This means ExternalDNS will delete records it doesn't manage. If you have manually created records in Pi-hole that you want to keep, change this to `upsert-only`.
- `--pihole-api-version=6`: Pi-hole v6 introduced a new API. Make sure you're using the correct version for your Pi-hole installation.
- `--default-targets=10.0.0.180`: All DNS records will point to this IP. This is my internal load balancer that routes traffic to the appropriate service.
- `--force-default-targets`: Forces all records to use the default target, even if ExternalDNS could determine a different IP from the service. This makes sense in my home lab as all Gateway API services are hitting the same MetalLB IP address.

**Authentication:**

Pi-hole requires authentication via the web API. I'm using External Secrets Operator to pull the password from Azure Key Vault:

{{< code "external-dns/pihole-external-secret.yml" "yaml" >}}

The deployment references this secret via `envFrom`, which automatically injects `EXTERNAL_DNS_PIHOLE_PASSWORD` as an environment variable.

For more information on this integration, see my post <a href="https://zaldre.com/posts/esoaks">here</a> for all the gory details.

### Cloudflare configuration ###

For public facing DNS, I'm using Cloudflare. The Cloudflare ExternalDNS deployment:

- Watches for Ingress and Gateway HTTPRoute resources
- Creates A records pointing to my external IP
- Uses TXT records for ownership tracking (so it knows which records it manages)
- Only processes resources with the `external-dns.alpha.kubernetes.io/hostname` annotation. This is an intentional decision to make sure theres separation between things that should and should not be available to the outside world

**Service Account:**

{{< code "external-dns/external-dns-cloudflare-serviceaccount.yml" "yaml" >}}

**ClusterRole:**

{{< code "external-dns/external-dns-cloudflare-clusterrole.yml" "yaml" >}}

**ClusterRoleBinding:**

{{< code "external-dns/external-dns-cloudflare-clusterrolebinding.yml" "yaml" >}}

**Deployment:**

{{< code "external-dns/external-dns-cloudflare-deployment.yml" "yaml" >}}

Key configuration points for Cloudflare:

- `--txt-owner-id=default`: This is used to track ownership. All TXT records created by this ExternalDNS instance will be tagged with this owner ID. If you run multiple ExternalDNS instances (e.g., for different environments), use different owner IDs.
- `--annotation-filter=external-dns.alpha.kubernetes.io/hostname`: Only processes resources that have this annotation. This gives you fine-grained control over which routes get exposed to the internet.
- `--default-targets=1.2.3.4`: My external IP address. All A records will point here.
- `--cloudflare-dns-records-per-page=5000`: Cloudflare's API paginates results. This increases the page size to reduce API calls.
- `--cloudflare-record-comment`: Adds a comment to each DNS record for easier identification in the Cloudflare dashboard.

**Authentication:**

Cloudflare requires an API key and email. Again, I'm using the External Secrets Operator:

{{< code "external-dns/cloudflare-external-secret.yml" "yaml" >}}

The deployment references the `cloudflareapikey` and `email` keys from this secret.

### Using it in your routes ###

Once ExternalDNS is deployed, using it depends on whether you want the service to be accessible internal only, or both externally and internally.

**Internal-only service (Pi-hole only):**

For services that should only be accessible from your internal network, create an HTTPRoute without the external-dns annotation. Here's my Immich instance, which is IP-whitelisted to only allow internal traffic:

{{< code "external-dns/immich-route.yml" "yaml" >}}

Since this route doesn't have the `external-dns.alpha.kubernetes.io/hostname` annotation, only the Pi-hole ExternalDNS deployment will create a DNS record:
- Pi-hole creates `immich.zaldre.com` → `10.0.0.180` (internal only)

**External service (both Pi-hole and Cloudflare):**

For services that should be accessible from the internet, add the `external-dns.alpha.kubernetes.io/hostname` annotation. Here's my stats dashboard, which is accessible from both internal and external networks:

{{< code "external-dns/stats-route.yml" "yaml" >}}

With this annotation, both ExternalDNS deployments will create DNS records:
- Pi-hole creates `stats.zaldre.com` → `10.0.0.180` (internal)
- Cloudflare creates `stats.zaldre.com` → `1.2.3.4` (external)

This gives you split-brain DNS: internal clients resolve to your internal IP, external clients resolve to your external IP, all managed automatically.

## Why two deployments? ##

You might be wondering why I'm running two separate deployments instead of configuring one ExternalDNS instance to handle both providers. The main reasons are:

1. **Different default targets**: Internal DNS needs to point to my internal IP, external DNS needs to point to my external IP
2. **Different policies**: I want different sync policies for each (Pi-hole uses `sync`, Cloudflare could use `upsert-only` as i do manage some records manually (Like this blog page)
3. **Different sources**: I might want to watch different resource types or use different annotation filters
4. **Isolation**: If one provider has issues, the other keeps working

## Troubleshooting tips ##

If DNS records aren't appearing, here are some things to check:

1. **Check the logs**: `kubectl logs -n external-dns deployment/external-dns-pihole` (or `external-dns-cloudflare`)
2. **Verify annotations**: For internet facing configs, Make sure your HTTPRoute/Ingress has `external-dns.alpha.kubernetes.io/hostname` annotation and that the "hostname" is correctly defined in your HTTPRoute/Ingress
3. **Check RBAC**: Ensure the service account has permissions to read the resources you're watching
4. **Authentication**: Verify secrets are correctly mounted and environment variables are set
5. **Provider connectivity**: For Pi-hole, ensure the service is reachable at the configured URL. For Cloudflare, verify API credentials are valid

ExternalDNS logs are usually quite verbose and will tell you exactly what it's doing (or not doing) with each resource.

## What about the downsides? ##

ExternalDNS is great, but it's not perfect:

- **Provider-specific quirks**: Each DNS provider has its own API limitations and quirks. Pi-hole doesn't support ownership tracking, some providers have rate limits, etc.
- **Reconciliation delays**: Changes aren't instant. ExternalDNS polls the DNS provider and compares the local entries for changes, so there's a delay between creating a route and the DNS record appearing (By default, 60 seconds. Reasonable, but still worth considering)
- **Ownership conflicts**: If you manually create DNS records that ExternalDNS thinks it should manage, you might see conflicts. This is why ownership tracking (TXT records) is important for providers that support it.
- **Risk**: Granting access to an automated system to manage something as fundamental as DNS can potentially cause havoc in your environment if you don't plan ahead. As always, test on something non-prod before you start blasting<br>
<img src="https://ih1.redbubble.net/image.5003976761.3901/flat,750x,075,f-pad,750x1000,f8f8f8.jpg" width="10%"><br>

## Other callouts ##

This configuration is specific to my setup, you'll want to tweak this to your needs. In particular, the `--default-targets` and `--force-default-targets` params are useful in a home lab, but you probably want to omit this for prod. You can configure it to pickup the IP address of the relevant loadbalancer your ingress/api gateway is attached to which is likely preferable for business use cases.


---
That's all for this post. I hope you found it useful. If there's any information you want clarified, thoughts, or opinions, please do let me know.

<a href="mailto:zaldre@zaldre.com">zaldre@zaldre.com</a>

---
