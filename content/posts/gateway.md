+++
date = '2025-11-23T10:00:00+11:00'
draft = false
title = 'gatewayapi | replacing ingress-nginx with envoy'
+++
<img src="https://gateway-api.sigs.k8s.io/images/logo/logo-text-horizontal.png" width="30%">
If you've been living under a rock, or maybe just busy with real life stuff, you might've missed the recent news that the fan favorite <a href="https://github.com/kubernetes/ingress-nginx">ingress-nginx</a> is going to be <a href="https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/">deprecated</a>

As someone who uses this software extensively both personally and professionally, this is a hard pill to swallow, but a perfect opportunity to prioritise learning the Gateway API. After all, if you're going to migrate anyway, you might as well take the leap sooner rather than later

## What is it? ##

The Gateway API is Kubernetes' next-generation way to control traffic into your clusters from outside. Think of it as Ingress 2.0, but with way more power and flexibility. It provides a vendor-agnostic, standardized way for traffic to enter your cluster while giving you much more granular control over routing, security, and traffic management.

Unlike the original Ingress resource (which was more of a suggestion than a specification), the Gateway API is a proper standard that multiple implementations can follow. This means you're not locked into a single vendor's interpretation of how things should work.

## What problem does this solve? ##

Historically with Ingress resources, there are a few key limitations that can make your life difficult:

**Limited expressiveness**: Want to do something slightly complex? Good luck. Ingress resources are pretty basic - they can only route traffic based on hostname and path, but that's about it. 

**Vendor lock-in through annotations**: They're the bane of portability. What works with ingress-nginx might not work with Traefik, and vice versa. You end up with manifests that are tightly coupled to your ingress controller of choice.

**Role separation**: With Ingress, you typically need a high level of permissions to create routes. There's no separation of concerns. This means you may need to give developers more access than you care to.

**Limited traffic management**: Want session affinity? Rate limiting? Advanced load balancing? You're back to those vendor-specific annotations again.

The Gateway API addresses all of these by providing a richer, more expressive API that's consistent across implementations. Plus, it introduces role-based access control at the API level.

## How does it work? ##

The Gateway API introduces several new resource types that work together:

- **GatewayClass**: Defines which controller implementation you want to use (like choosing between ingress-nginx and Traefik, but at the API level)
- **Gateway**: The actual entry point into your cluster - think of it as the load balancer or reverse proxy
- **HTTPRoute**: The routing rules that attach to a Gateway (this is the replacement for Ingress)
- **BackendTrafficPolicy**: Advanced traffic management like load balancing, timeouts, and retries
- **SecurityPolicy**: Security controls like IP whitelisting, authentication, and rate limiting

The beauty of this design is that it separates concerns. Infrastructure teams can manage the Gateway (the entry point), while application teams can manage their HTTPRoutes (the routing rules) without stepping on each other's toes.

## My implementation: Envoy Gateway ##

**Environment:** Talos v1.11.5 | Kubernetes v1.34.2

For my home lab setup, I chose <a href="https://gateway.envoyproxy.io">Envoy Gateway</a> as the implementation. Envoy is battle-tested, performant, and has excellent observability. Plus, the Envoy Gateway project makes it dead simple to deploy, it's just a Helm chart.

In this example we will be keeping it simple and using 1 Gateway that will handle both internal (LAN) and external (WAN) traffic. In production, you'd likely want to create 2 different Gateways using 2 Load Balancers to create the required segregation.

### Installation ###

I'm running this on a Talos cluster with ArgoCD managing everything via GitOps, so the installation is handled through an ArgoCD Application:

{{< code "gateway/envoy-helm.yml" "yaml" >}}

This deploys Envoy Gateway v1.6.0 into the `envoy-gateway-system` namespace. The Helm chart handles all the heavy lifting - deploying the controller, setting up RBAC, and managing the lifecycle. This also installs the Gateway API CRDs, which at the time of writing are not installed in the cluster by default.

### Setting up the Gateway ###

Once Envoy Gateway is installed, you need to create three resources to get traffic flowing:

**1. GatewayClass** - This tells Kubernetes which controller should handle Gateway resources:

{{< code "gateway/envoy-gatewayclass.yml" "yaml" >}}

**2. EnvoyProxy** - This is the Envoy-specific configuration for how the gateway should be deployed. In my case, I'm using MetalLB for load balancing, so I specify a static IP:

{{< code "gateway/envoy-proxy.yml" "yaml" >}}

**3. Gateway** - The actual entry point. This is where you define your listeners (ports, protocols, TLS settings):

{{< code "gateway/envoy-gateway.yml" "yaml" >}}

This Gateway listens on port 443 for HTTPS traffic to any `*.zaldre.com` hostname, terminates TLS using a wildcard certificate, and allows routes from all namespaces.

### Creating routes ###

Now for the fun part - creating routes for your applications. Each application gets its own HTTPRoute resource. Let's start with a simple example—my stats dashboard, which is accessible from both internal and external networks:

{{< code "gateway/stats-route.yml" "yaml" >}}

This is a straightforward route:
- Routes traffic for `stats.zaldre.com` to the `stats` service on port 80
- Sets the `X-Forwarded-Proto` header so the backend knows it's receiving HTTPS traffic
- Uses a BackendTrafficPolicy for session affinity (cookie-based load balancing)
- Notice there's no SecurityPolicy here - this route is open to both internal and external traffic

The annotations on the HTTPRoute are for external-dns integration, which automatically creates DNS records for the hostname.

### Adding security policies ###

For services that should only be accessible from internal networks, you can add a SecurityPolicy. Here's my Immich instance with IP whitelisting:

{{< code "gateway/immich-route.yml" "yaml" >}}

Let me break down what's happening here:

**The HTTPRoute** defines the basic routing:
- It attaches to the `gateway` in the `envoy-gateway-system` namespace
- Routes traffic for `immich.zaldre.com` to the `immich-server` service on port 2283
- Sets the `X-Forwarded-Proto` header so the backend knows it's receiving HTTPS traffic

**The SecurityPolicy** adds IP whitelisting:
- Only allows traffic from private IP ranges (10.0.0.0/8 and 192.168.0.0/16)
- Denies everything else by default
- This is way cleaner than the annotation-based approach you'd use with ingress-nginx

**The BackendTrafficPolicy** configures session affinity:
- Uses cookie-based consistent hashing for load balancing
- Ensures users stick to the same backend pod (important for stateful applications)
- Sets a 48-hour cookie TTL with SameSite=Lax

### Routing to external services ###

One cool thing I discovered is that Gateway API can route to services outside your cluster. I use this for my NAS:

{{< code "gateway/nas-route.yml" "yaml" >}}

This creates a Service without a selector (making it a headless service), then uses an EndpointSlice to point to an external IP address (10.0.0.200). The HTTPRoute then routes to this service just like any other. The gateway terminates TLS and forwards plain HTTP to the backend, which is perfect for devices that don't handle TLS termination well.

## Migration experience ##

Migrating from ingress-nginx was surprisingly straightforward. The main steps were:

1. **Install Envoy Gateway** - One Helm chart, done
2. **Create the Gateway resources** - Three YAML files (GatewayClass, EnvoyProxy, Gateway)
3. **Convert Ingress to HTTPRoute** - For each application, create an HTTPRoute. The mapping is pretty direct:
   - `spec.rules[].host` → `spec.hostnames[]`
   - `spec.rules[].http.paths[]` → `spec.rules[].matches[]`
   - `spec.rules[].http.paths[].backend` → `spec.rules[].backendRefs[]`

4. **Add policies** - This is where it gets interesting. Things that required annotations in ingress-nginx (like IP whitelisting) are now proper API resources (SecurityPolicy). This makes them more discoverable, testable, and maintainable.

5. **Test and switch over** - I ran both ingress-nginx and Envoy Gateway in parallel for a bit, routing different hostnames to each, then gradually migrated everything over.

The biggest win? No more annotation soup. Everything is declarative, type-safe, and follows Kubernetes resource patterns. Plus, the separation of concerns means I can delegate route management to different teams without giving them access to the Gateway itself.

## What about the downsides? ##

It's not all sunshine and rainbows. The Gateway API is still evolving, and some features you might be used to from ingress-nginx aren't available yet (or require different approaches). Also, if you're heavily invested in ingress-nginx-specific annotations, you'll need to rethink some of your configurations.

The spec is also far more complex than ingress. It is by no means a panacea and may introduce more problems than it solves. If your organisation values simplicity (or if you're operating in a smaller team) it may be worth looking at <a href="https://doc.traefik.io/traefik/reference/install-configuration/providers/kubernetes/kubernetes-ingress/">Traefik Ingress Controller</a>. They've recently written a blog post <a href="https://traefik.io/blog/migrate-from-ingress-nginx-to-traefik-now">here</a> that details a more simplified migration strategy while maintaining support for most of the ingress-nginx annotations.

That said, the deprecation of ingress-nginx means you're going to have to migrate to <strong>SOMETHING</strong> eventually anyway. Better to do it now while you have time to plan and test, rather than in a panic when something breaks.

---

That's all for this post. I hope you found it useful. If there's any information you want clarified, thoughts, or opinions, please do let me know.

<a href="mailto:zaldre@zaldre.com">zaldre@zaldre.com</a>

---

