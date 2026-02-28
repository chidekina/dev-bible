# Kubernetes Concepts

## Overview

Kubernetes (K8s) is a container orchestration platform that automates deploying, scaling, and managing containerized applications. Instead of telling Kubernetes what to do ("run this container on this machine"), you declare the desired state ("I want 3 replicas of this app") and Kubernetes continuously works to make reality match your declaration.

Originally created by Google and now maintained by the CNCF, Kubernetes has become the de facto standard for running containers at scale. Understanding its architecture — control plane vs worker nodes, the role of etcd, how the scheduler places workloads — is essential for troubleshooting, performance tuning, and making good infrastructure decisions.

## Prerequisites

- Docker basics (images, containers, ports, volumes)
- Basic networking (IP addresses, DNS, ports, TCP/UDP)
- YAML syntax
- Linux command line fundamentals

## Core Concepts

### The Declarative Model

The fundamental shift Kubernetes introduces is **declarative configuration**: you describe what you want, not the steps to get there.

```yaml
# Imperative: "start a container running nginx"
docker run -d nginx

# Declarative: "ensure 3 nginx pods are always running"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

Kubernetes stores this desired state in etcd. Its control loops continuously compare desired state to actual state and reconcile differences. If a node fails and two pods die, the Deployment controller notices the replica count is wrong and schedules two new pods.

### Control Plane vs Worker Nodes

A Kubernetes cluster consists of a **control plane** and **worker nodes**.

**Control Plane** (manages the cluster):

```
┌─────────────────────────────────────────────┐
│                Control Plane                 │
│                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │  kube-   │  │  kube-   │  │  cloud-  │  │
│  │  apiserver│  │ scheduler│  │controller│  │
│  └──────────┘  └──────────┘  └──────────┘  │
│                                             │
│  ┌──────────────────┐  ┌──────────────────┐ │
│  │ kube-controller- │  │      etcd        │ │
│  │     manager      │  │  (state store)   │ │
│  └──────────────────┘  └──────────────────┘ │
└─────────────────────────────────────────────┘
```

- **kube-apiserver** — The front door. All cluster communication goes through it (kubectl, other control plane components, kubelets). Validates, authenticates, and persists API objects.
- **etcd** — Distributed key-value store. The single source of truth for all cluster state. Backing up etcd = backing up your cluster.
- **kube-scheduler** — Watches for unscheduled Pods and assigns them to nodes based on resource requirements, affinity rules, taints/tolerations, and topology.
- **kube-controller-manager** — Runs a collection of controllers (Deployment controller, Node controller, ReplicaSet controller, etc.). Each controller watches the API server and acts when actual state diverges from desired state.
- **cloud-controller-manager** — Integrates with cloud provider APIs (AWS, GCP, Azure) for provisioning load balancers, persistent volumes, and node lifecycle.

**Worker Nodes** (run your workloads):

```
┌───────────────────────────────┐
│          Worker Node          │
│                               │
│  ┌─────────┐  ┌────────────┐  │
│  │ kubelet │  │ kube-proxy │  │
│  └─────────┘  └────────────┘  │
│                               │
│  ┌────────────────────────┐   │
│  │   Container Runtime    │   │
│  │  (containerd / CRI-O)  │   │
│  └────────────────────────┘   │
│                               │
│  ┌──────┐ ┌──────┐ ┌──────┐  │
│  │ Pod  │ │ Pod  │ │ Pod  │  │
│  └──────┘ └──────┘ └──────┘  │
└───────────────────────────────┘
```

- **kubelet** — Agent running on every node. Receives Pod specs from the API server and ensures the specified containers are running and healthy. Reports node and Pod status back to the API server.
- **kube-proxy** — Maintains network rules on nodes. Implements the Service abstraction by routing traffic to the correct Pods. Uses iptables or IPVS under the hood.
- **Container runtime** — The software that actually runs containers. Kubernetes uses the Container Runtime Interface (CRI); common runtimes are containerd and CRI-O.

### Core API Objects

Everything in Kubernetes is an API object with:
- `apiVersion` — which API group and version
- `kind` — type of object
- `metadata` — name, namespace, labels, annotations
- `spec` — desired state
- `status` — current state (written by controllers)

**Pod** — The smallest deployable unit. A Pod wraps one or more containers that share:
- Same network namespace (same IP, same `localhost`)
- Same storage volumes
- Same lifecycle (scheduled together, die together)

**ReplicaSet** — Ensures N copies of a Pod are always running. Rarely used directly — use Deployments.

**Deployment** — Manages ReplicaSets and provides declarative updates with rollout history and rollback.

**Service** — Stable network endpoint for a set of Pods. Pods are ephemeral (their IPs change); Services provide a fixed IP and DNS name.

**ConfigMap** — Non-sensitive configuration data (env vars, config files).

**Secret** — Sensitive data (passwords, tokens, TLS certs) — base64-encoded in etcd, encrypted at rest with proper cluster configuration.

**Namespace** — Virtual cluster within a cluster. Provides resource isolation and access control boundaries.

**Node** — A worker machine (VM or physical).

**PersistentVolume / PersistentVolumeClaim** — Storage provisioning abstraction.

**ServiceAccount** — Identity for processes running in Pods.

### Labels and Selectors

Labels are key/value pairs attached to objects. Selectors filter objects by labels. This is how Services find their Pods, how Deployments manage their ReplicaSets, and how you organize resources.

```yaml
metadata:
  labels:
    app: api-server
    version: "2.4"
    environment: production
    team: platform
```

Selector examples:
```bash
# kubectl filter
kubectl get pods -l app=api-server
kubectl get pods -l environment=production,team=platform

# In a Service spec
selector:
  app: api-server
  environment: production
```

### The Reconciliation Loop

Every Kubernetes controller runs a loop:

```
watch API server for changes
  → compare desired state (spec) to actual state (status)
  → take action to move actual toward desired
  → update status
  → repeat
```

This is why Kubernetes is **eventually consistent** and **self-healing**. Delete a Pod that belongs to a Deployment? The Deployment controller sees 2 replicas instead of 3 and creates a new one within seconds.

### etcd — The Brain

etcd is a distributed key-value store built on the Raft consensus algorithm. All cluster state lives here. Key properties:
- **Strongly consistent** — reads always return the latest committed value
- **Watch mechanism** — clients can watch keys for changes (how controllers get notified)
- **Distributed** — typically 3 or 5 nodes for HA (must be odd for Raft quorum)

```bash
# Backing up etcd (do this regularly in production)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

## Hands-On Examples

### Exploring a Cluster's Architecture

```bash
# See all nodes (control plane + workers)
kubectl get nodes -o wide

# Describe a node to see allocatable resources, pods, conditions
kubectl describe node worker-node-1

# See control plane components running as pods (in kubeadm clusters)
kubectl get pods -n kube-system

# Watch the scheduler decisions in real time
kubectl get events --sort-by=.metadata.creationTimestamp -n default -w
```

### Understanding API Groups

```bash
# List all available API resources and their groups
kubectl api-resources

# Check API versions for a resource
kubectl explain deployment --api-version apps/v1
kubectl explain deployment.spec.strategy
```

### Watching Reconciliation in Action

```bash
# Terminal 1: watch pods
kubectl get pods -w

# Terminal 2: delete a pod
kubectl delete pod my-app-7d9f8-xyz

# Observe: Deployment controller immediately creates a replacement
```

### Checking etcd Health (on control plane node)

```bash
# Check etcd cluster health
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key

# List all keys (shows what's stored)
ETCDCTL_API=3 etcdctl get / --prefix --keys-only | head -30
```

## Common Patterns & Best Practices

### Namespace Strategy

```yaml
# Separate by environment
namespaces:
  - production
  - staging
  - development

# Or by team + environment
namespaces:
  - platform-production
  - platform-staging
  - api-production
  - api-staging
```

Namespaces provide:
- RBAC boundaries (different teams, different access)
- Resource quota enforcement
- Network policy scoping
- DNS isolation (`service.namespace.svc.cluster.local`)

### Always Use Labels Consistently

Adopt a labeling convention and apply it everywhere:

```yaml
metadata:
  labels:
    app.kubernetes.io/name: api-server
    app.kubernetes.io/version: "2.4.1"
    app.kubernetes.io/component: backend
    app.kubernetes.io/part-of: commerce-platform
    app.kubernetes.io/managed-by: helm
    environment: production
    team: platform
```

The `app.kubernetes.io/` prefix is a Kubernetes convention supported by many tools.

### Resource Requests and Limits (Always Set Them)

The scheduler uses requests to decide placement. Limits prevent runaway containers from starving neighbors.

```yaml
resources:
  requests:
    cpu: 100m      # 0.1 CPU cores — what the scheduler reserves
    memory: 128Mi  # what the scheduler reserves
  limits:
    cpu: 500m      # 0.5 cores — hard cap
    memory: 512Mi  # hard cap — exceed this → OOMKilled
```

Without requests, the scheduler has no data and places pods randomly. Without limits, a single buggy pod can starve an entire node.

## Anti-Patterns to Avoid

**Running workloads on control plane nodes** — taints prevent this by default; don't remove them.

**Not backing up etcd** — losing etcd = losing the cluster state. Automate daily snapshots.

**Using the `default` namespace for everything** — makes access control, resource quotas, and cleanup impossible.

**Ignoring resource requests and limits** — leads to noisy neighbor problems and unpredictable scheduling.

**Using `latest` image tags** — makes rollbacks impossible and pulls unpredictable image versions. Always use specific version tags.

**Direct control plane access for workloads** — Pods should use ServiceAccounts, not kubeconfig files mounted as Secrets.

## Debugging & Troubleshooting

### Cluster Health Check

```bash
# Check component statuses
kubectl get componentstatuses

# Check node conditions
kubectl get nodes
kubectl describe node <node-name> | grep -A5 Conditions

# Check system pods
kubectl get pods -n kube-system

# View cluster events (sorted newest first)
kubectl get events --sort-by=.metadata.creationTimestamp --all-namespaces | tail -30
```

### API Server Issues

```bash
# Check API server logs (on control plane node)
sudo journalctl -u kube-apiserver --since "10 min ago"

# Or if running as a pod:
kubectl logs -n kube-system kube-apiserver-<node-name>
```

### Node Not Ready

```bash
kubectl describe node <node-name>
# Look for: Conditions, Events, Allocated resources

# Check kubelet on the node
sudo systemctl status kubelet
sudo journalctl -u kubelet --since "5 min ago"
```

## Real-World Scenarios

### Scenario 1: Pod Keeps Restarting

```bash
# See restart count
kubectl get pod my-app-7d9f8-xyz

# Check current logs
kubectl logs my-app-7d9f8-xyz

# Check previous container's logs (after restart)
kubectl logs my-app-7d9f8-xyz --previous

# Describe for events and state
kubectl describe pod my-app-7d9f8-xyz
# Look for: OOMKilled → increase memory limit
# Look for: CrashLoopBackOff → check application logs
# Look for: ImagePullBackOff → check image name/tag and pull secret
```

### Scenario 2: Understanding Scheduling Decisions

```bash
# Pod is Pending — why?
kubectl describe pod my-app-pending
# Events section: "0/3 nodes are available: 3 Insufficient memory"
# Solution: increase memory requests or add more nodes

# Or: "0/3 nodes have the required toleration"
# Solution: check taints on nodes and tolerations in pod spec
```

## Further Reading

- [Kubernetes Documentation — Concepts](https://kubernetes.io/docs/concepts/)
- [Kubernetes the Hard Way — Kelsey Hightower](https://github.com/kelseyhightower/kubernetes-the-hard-way) — understand by building from scratch
- [The Kubernetes Book — Nigel Poulton](https://nigelpoulton.com/books/) — concise, practical intro
- [etcd Documentation](https://etcd.io/docs/) — clustering, operations, backup
- [Kubernetes API Reference](https://kubernetes.io/docs/reference/kubernetes-api/)

## Summary

Kubernetes is a declarative container orchestrator. You declare desired state; controllers reconcile it with reality continuously.

Architecture:
- **Control plane**: API server (front door), etcd (state), scheduler (pod placement), controller-manager (reconciliation loops)
- **Worker nodes**: kubelet (runs pods), kube-proxy (networking), container runtime (containerd/CRI-O)

Key concepts:
- **Declarative model** — describe what you want, not how to get there
- **Labels and selectors** — how resources find and group each other
- **Reconciliation loop** — every controller watches for drift and corrects it
- **etcd** — the cluster's single source of truth; back it up
- **Namespaces** — isolation and organization boundaries; don't use just `default`
- **Always set resource requests/limits** — essential for scheduling and stability
- **Never use `latest` image tag** — use explicit version tags for predictable rollouts and rollbacks
