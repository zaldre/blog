+++
date = '2025-11-03T10:00:00+11:00'
draft = false
title = 'kind | kubernetes in docker'
+++

I've been working with Kubernetes for some time now, and while my preferred platform of choice is <a href="https://www.talos.dev/">talos</a> one of the challenges I've always faced is how to test and develop and iterate quickly without spinning up a full cluster or dealing with expensive cloud resources.

That is where <a href="https://kind.sigs.k8s.io">kind (Kubernetes in Docker)</a> comes in handy

## What problem does this solve? ##

Kind allows you to quickly and easily spin up kubernetes clusters from within docker itself. For me I use this when I need to test things like GitOps configurations that have multiple cluster requirements (Think ApplicationSets in ArgoCD). 

You can have multiple clusters running as docker containers on the same machine, eliminating the TTD (time to deploy) quite dramatically.

As a result of this, finding bugs and gaps in your GitOps process becomes far simpler. No one wants to be triaging misconfigurations during a disaster recovery scenario.


## How does it work? ##

kind uses Docker to create containers that act as Kubernetes nodes. Each container runs a small Kubernetes distribution, and kind handles the networking and configuration to make them work together as a cluster.

The magic happens through Docker-in-Docker—kind containers which run the Kubernetes components (kubelet, kube-apiserver, etc.) directly, giving you a real Kubernetes experience without the overhead of virtual machines or cloud resources.

### Installation ###

Getting started with kind is straightforward. You'll need <a href="https://www.docker.com/">Docker</a> installed and running on your machine.

Once that's done, you'll need to install the kind binaries

You can check the latest version and installation instructions at <a href="https://kind.sigs.k8s.io/docs/user/quick-start/#installation">kind's documentation</a>

### Creating your first cluster ###

Once kind is installed, creating a cluster is as simple as:

```bash
kind create cluster --name my-cluster
```

This creates a single-node cluster named "my-cluster". kind will automatically configure kubectl to use this cluster, so you can immediately start using `kubectl` commands.

To verify everything is working:

```bash
kubectl cluster-info --context kind-my-cluster
kubectl get nodes
```

### Multi-node clusters ###

One of kind's powerful features is the ability to create multi-node clusters using a simple configuration file. This is great for testing features that require multiple nodes, like pod scheduling or node affinity.

Create a file called `kind-config.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
```

Then create the cluster with:

```bash
kind create cluster --name multi-node --config kind-config.yaml
```

This gives you a cluster with one control plane node and two worker nodes—perfect for testing more complex scenarios.

### Loading container images ###

One common challenge with Kubernetes is that you need to pull conatainer images from container registries during testing. This makes working offline or testing local images slow and laborious.

kind solves this with the `kind load docker-image` command.

```bash
#Basic Dockerfile Example.
FROM alpine
RUN apk add --no-cache nginx
```

```bash
# Build your image locally
docker build -t my-app:latest .

# Load it into the kind cluster
kind load docker-image my-app:latest --name my-cluster
```

Now your pods can use `my-app:latest` without needing to pull from a registry.

### Example: Deploying a simple application ###

Here's a quick example of deploying an application to kind:

```bash
# Create a cluster
kind create cluster --name test-app

# Create a namespace
kubectl create namespace demo

# Deploy a simple nginx deployment
kubectl create deployment nginx --image=nginx:latest -n demo

# Expose it as a service
kubectl expose deployment nginx --port=80 --type=NodePort -n demo

# Get the service details
kubectl get svc nginx -n demo
```

### Cleaning up ###

When you're done testing, cleaning up is just as easy:

```bash
# Delete a specific cluster
kind delete cluster --name my-cluster

# List all your kind clusters
kind get clusters

# Delete all kind clusters
kind delete clusters --all
```

---

## Common use cases ##

kind is particularly useful for:

- **Local development**: Test your manifests and applications before deploying to production
- **CI/CD pipelines**: Run integration tests in a real Kubernetes environment without cloud costs
- **Operator testing**: Develop and test Kubernetes operators locally
- **Learning Kubernetes**: Experiment with Kubernetes features without the complexity of cloud setup
- **Cluster configuration testing**: Validate cluster configurations, network policies, and RBAC rules


---

That's all for this post. Hope you found it useful. If there's any information you want clarified, thoughts, opinions, please do let me know

<a href="mailto:zaldre@zaldre.com">zaldre@zaldre.com</a>

---

