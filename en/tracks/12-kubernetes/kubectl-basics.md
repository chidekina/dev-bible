# kubectl Basics

## Overview

`kubectl` is the command-line interface for Kubernetes. Every interaction with a cluster — reading state, applying configuration, debugging pods, forwarding ports — goes through kubectl. Mastering it makes the difference between spending 30 seconds debugging and spending 30 minutes.

This chapter covers context management, the essential read/debug commands (`get`, `describe`, `logs`, `exec`), applying and managing configuration, port-forwarding for local debugging, and the dry-run workflow for safer deployments.

## Prerequisites

- kubectl installed (`brew install kubectl` or download from kubernetes.io)
- Access to a cluster (local: kind, minikube, k3s; or cloud: EKS, GKE, AKS)
- KUBECONFIG file pointing at your cluster (provided by your cluster or `~/.kube/config`)

## Core Concepts

### Contexts and Clusters

kubectl can manage connections to multiple clusters using **contexts**. A context bundles a cluster, a user (credentials), and a default namespace.

```bash
# View your current context and all available contexts
kubectl config current-context
kubectl config get-contexts

# Switch context
kubectl config use-context my-production-cluster

# View the full kubeconfig
kubectl config view

# Set a default namespace for a context (so you don't add -n everywhere)
kubectl config set-context --current --namespace=my-namespace
```

Managing multiple contexts safely:
```bash
# Install kubectx for fast context switching
brew install kubectx

kubectx                          # list contexts
kubectx my-staging-cluster       # switch to staging
kubectx -                        # switch back to previous context
kubens                           # list namespaces in current context
kubens my-namespace              # switch default namespace
```

### The kubectl Command Structure

```
kubectl [command] [TYPE] [NAME] [flags]

command: get, describe, apply, delete, logs, exec, port-forward, ...
TYPE:    pod, deployment, service, configmap, secret, node, ...
NAME:    specific resource name (optional — omit to list all)
flags:   -n <namespace>, -o <output-format>, -l <label-selector>, ...
```

### Output Formats

```bash
# Default: human-readable table
kubectl get pods

# Wide: more columns (node, IP, etc.)
kubectl get pods -o wide

# YAML: full spec as stored in etcd
kubectl get deployment my-app -o yaml

# JSON: same as YAML but JSON
kubectl get pod my-app-xyz -o json

# Custom columns
kubectl get pods -o custom-columns='NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName'

# JSONPath: extract specific fields
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}'

# Go template
kubectl get pods -o go-template='{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}'
```

## Hands-On Examples

### Reading Resources

```bash
# List pods in current namespace
kubectl get pods

# List pods in all namespaces
kubectl get pods --all-namespaces
kubectl get pods -A  # shorthand

# List pods with a label selector
kubectl get pods -l app=api-server
kubectl get pods -l environment=production,tier=backend

# Get a specific pod
kubectl get pod my-app-7d9f8-xyz

# Watch resources (live updates)
kubectl get pods -w
kubectl get pods --watch

# List multiple resource types
kubectl get pods,services,deployments

# List with additional info (node, IP, nominated node, readiness gates)
kubectl get pods -o wide

# Sort by a field
kubectl get pods --sort-by=.metadata.creationTimestamp
kubectl get pods --sort-by=.status.startTime
```

### Describing Resources

`describe` gives you the full picture: spec, status, events, and conditions. Use it when something isn't working.

```bash
# Describe a pod
kubectl describe pod my-app-7d9f8-xyz

# Describe a node
kubectl describe node worker-node-1

# Describe a deployment
kubectl describe deployment my-app

# Describe a service
kubectl describe service my-app-svc

# Describe without knowing exact name (lists matching)
kubectl describe pods -l app=my-app
```

Describe output sections to focus on:
- **Conditions** — Node or Pod health conditions
- **Events** — recent activity, warnings, errors (most useful for debugging)
- **Containers** → **State**, **Last State** — current and previous container state

### Reading Logs

```bash
# Logs for a pod (single container)
kubectl logs my-app-7d9f8-xyz

# Follow (stream) logs
kubectl logs -f my-app-7d9f8-xyz

# Last N lines
kubectl logs my-app-7d9f8-xyz --tail=100

# Logs since time period
kubectl logs my-app-7d9f8-xyz --since=1h
kubectl logs my-app-7d9f8-xyz --since=30m

# Previous container's logs (after crash/restart)
kubectl logs my-app-7d9f8-xyz --previous
kubectl logs my-app-7d9f8-xyz -p

# Logs from a specific container in a multi-container pod
kubectl logs my-app-7d9f8-xyz -c sidecar-container

# Logs from all pods matching a label (aggregated)
kubectl logs -l app=my-app --all-containers
kubectl logs -l app=my-app --tail=50

# Timestamps
kubectl logs my-app-7d9f8-xyz --timestamps
```

### Executing Commands in Pods

```bash
# Interactive shell
kubectl exec -it my-app-7d9f8-xyz -- /bin/sh
kubectl exec -it my-app-7d9f8-xyz -- /bin/bash

# Single command
kubectl exec my-app-7d9f8-xyz -- ls /app
kubectl exec my-app-7d9f8-xyz -- env
kubectl exec my-app-7d9f8-xyz -- cat /etc/hosts

# In a specific container
kubectl exec -it my-app-7d9f8-xyz -c api -- /bin/sh

# Check DNS resolution from inside a pod
kubectl exec my-app-7d9f8-xyz -- nslookup my-service
kubectl exec my-app-7d9f8-xyz -- curl http://my-service:8080/health

# Check environment variables
kubectl exec my-app-7d9f8-xyz -- printenv | grep DATABASE
```

### Port Forwarding

Port-forward creates a tunnel from your local machine to a pod or service — invaluable for debugging.

```bash
# Forward local port 8080 to pod port 3000
kubectl port-forward pod/my-app-7d9f8-xyz 8080:3000

# Forward to a service (automatically picks a healthy pod)
kubectl port-forward service/my-app-svc 8080:80

# Forward to a deployment
kubectl port-forward deployment/my-app 8080:3000

# Listen on all interfaces (so other machines on network can reach it)
kubectl port-forward service/my-app-svc 8080:80 --address 0.0.0.0

# Port-forward to a database for local access
kubectl port-forward service/postgres 5432:5432 -n database
# Now: psql -h localhost -p 5432 -U app mydb

# Background port-forward
kubectl port-forward service/my-app-svc 8080:80 &
# Kill it:
kill %1
```

### Applying Configuration

```bash
# Apply a manifest (create or update)
kubectl apply -f deployment.yaml

# Apply all manifests in a directory
kubectl apply -f ./k8s/

# Apply recursively
kubectl apply -f ./k8s/ -R

# Apply from URL
kubectl apply -f https://raw.githubusercontent.com/org/repo/main/manifests/app.yaml

# Show diff before applying (server-side dry run)
kubectl diff -f deployment.yaml

# Dry run — validate without applying
kubectl apply -f deployment.yaml --dry-run=client
kubectl apply -f deployment.yaml --dry-run=server

# Create vs apply
kubectl create -f deployment.yaml   # fails if already exists
kubectl apply -f deployment.yaml    # create or update (idempotent) ← always prefer this
```

### Deleting Resources

```bash
# Delete a specific resource
kubectl delete pod my-app-7d9f8-xyz
kubectl delete deployment my-app

# Delete from manifest (deletes what the file describes)
kubectl delete -f deployment.yaml

# Delete by label
kubectl delete pods -l app=my-app

# Delete all resources in a namespace
kubectl delete all --all -n my-namespace

# Force delete a stuck pod (last resort)
kubectl delete pod my-app-xyz --force --grace-period=0
```

## Common Patterns & Best Practices

### The Debug Workflow

Standard debug sequence when something is broken:

```bash
# 1. What's the overall state?
kubectl get pods -n my-namespace

# 2. Which pod is unhealthy?
kubectl get pods -n my-namespace | grep -v Running

# 3. What happened?
kubectl describe pod <unhealthy-pod> -n my-namespace
# Read the Events section carefully

# 4. What did the application say?
kubectl logs <pod> -n my-namespace
kubectl logs <pod> -n my-namespace --previous  # if it crashed

# 5. Can I get inside?
kubectl exec -it <pod> -n my-namespace -- /bin/sh

# 6. Is the service routing correctly?
kubectl get endpoints my-service -n my-namespace
# If endpoints is empty: service selector doesn't match any pod labels
```

### Aliases for Daily Use

```bash
# ~/.zshrc or ~/.bashrc
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgpa='kubectl get pods -A'
alias kgs='kubectl get services'
alias kgd='kubectl get deployments'
alias kgn='kubectl get nodes'
alias kdp='kubectl describe pod'
alias kdd='kubectl describe deployment'
alias kl='kubectl logs'
alias klf='kubectl logs -f'
alias kex='kubectl exec -it'
alias kaf='kubectl apply -f'
alias kdelf='kubectl delete -f'

# Enable kubectl autocompletion
source <(kubectl completion zsh)
complete -F __start_kubectl k
```

### Server-Side Apply

Prefer server-side apply in CI/CD — it tracks field ownership and handles conflicts better:

```bash
kubectl apply --server-side -f deployment.yaml
kubectl apply --server-side -f ./k8s/
```

### Generating Manifests from Commands

Use `--dry-run=client -o yaml` to generate manifest templates:

```bash
# Generate a deployment manifest
kubectl create deployment my-app \
  --image=my-app:1.2.3 \
  --replicas=3 \
  --dry-run=client \
  -o yaml > deployment.yaml

# Generate a service manifest
kubectl expose deployment my-app \
  --port=80 \
  --target-port=3000 \
  --dry-run=client \
  -o yaml > service.yaml

# Generate a ConfigMap from a file
kubectl create configmap app-config \
  --from-file=config.json \
  --dry-run=client \
  -o yaml > configmap.yaml
```

### Temporary Debug Pods

When you need a pod for debugging (curl, dig, psql) without modifying your app:

```bash
# Run a temporary busybox pod
kubectl run debug --image=busybox --rm -it --restart=Never -- sh

# Run a curl-capable pod
kubectl run debug --image=curlimages/curl --rm -it --restart=Never -- sh

# Debug a specific node's network
kubectl run debug --image=busybox --rm -it --restart=Never \
  --overrides='{"spec":{"nodeName":"worker-node-1"}}' -- sh

# Run with a specific service account
kubectl run debug --image=busybox --rm -it --restart=Never \
  --serviceaccount=my-service-account -- sh
```

## Anti-Patterns to Avoid

**Using `kubectl create` instead of `kubectl apply`**

`create` fails on second run; `apply` is idempotent. Use `apply` everywhere.

**Editing resources directly with `kubectl edit`**

```bash
# Avoid this in production
kubectl edit deployment my-app

# Use instead: update the YAML file, apply it, commit to git
vim deployment.yaml && kubectl apply -f deployment.yaml
```

`kubectl edit` creates changes that aren't in version control and bypasses review.

**Force-deleting pods as a first response**

```bash
# Don't do this first
kubectl delete pod --force --grace-period=0

# Do this first: understand why
kubectl describe pod <name>
kubectl logs <name> --previous
```

**Accessing production with the wrong context**

```bash
# Add a confirmation for production commands
function kprod() {
  echo "WARNING: You are about to run: kubectl $@"
  echo "Context: $(kubectl config current-context)"
  read -p "Continue? (y/N) " confirm
  [[ $confirm == "y" ]] && kubectl "$@"
}
```

## Debugging & Troubleshooting

### Pod in CrashLoopBackOff

```bash
kubectl describe pod <name>    # check Events
kubectl logs <name> --previous # check last crash output
kubectl logs <name>            # check current output
# Common causes: bad env vars, bad startup command, missing dependency, OOMKill
```

### ImagePullBackOff

```bash
kubectl describe pod <name>
# Events: Failed to pull image — check:
# 1. Image name typo
# 2. Image tag doesn't exist
# 3. Registry is private — missing imagePullSecret
# 4. Registry is down or rate-limited (Docker Hub)
```

### Pod Stuck in Pending

```bash
kubectl describe pod <name>
# Events section shows why:
# "0/3 nodes available: 3 Insufficient memory" → add nodes or reduce requests
# "0/3 nodes have the required toleration" → check taints/tolerations
# "0/3 nodes available: persistentvolumeclaim not bound" → PVC issue
```

### Service Not Routing Traffic

```bash
# Check that endpoints exist (selector must match pod labels)
kubectl get endpoints my-service

# If endpoints is empty:
kubectl get pods -l app=my-app     # pods must exist
kubectl describe service my-service  # check selector
kubectl get pods --show-labels       # verify labels

# Test connectivity from within cluster
kubectl run test --image=busybox --rm -it --restart=Never \
  -- wget -qO- http://my-service:80/health
```

### Checking Resource Usage

```bash
# Node resource usage (requires metrics-server)
kubectl top nodes

# Pod resource usage
kubectl top pods
kubectl top pods --sort-by=memory
kubectl top pods -n my-namespace
```

## Real-World Scenarios

### Scenario 1: Investigating a Production Issue

```bash
# Step 1: Get the lay of the land
kubectl get pods -n production
# OUTPUT: my-api-7d8f9 is in Error state, 2/3 available

# Step 2: Get details
kubectl describe pod my-api-7d8f9 -n production
# Last State: Terminated, Exit Code 1, OOMKilled

# Step 3: Increase memory limit in deployment
kubectl get deployment my-api -n production -o yaml > /tmp/my-api.yaml
vim /tmp/my-api.yaml  # increase memory limit from 256Mi to 512Mi
kubectl apply -f /tmp/my-api.yaml

# Step 4: Watch rollout
kubectl rollout status deployment/my-api -n production

# Step 5: Verify
kubectl top pods -n production
```

### Scenario 2: Automated Health Check Script

```bash
#!/bin/bash
# scripts/k8s-health-check.sh

NAMESPACE="${1:-default}"
ERRORS=0

echo "=== Kubernetes Health Check: $NAMESPACE ==="

# Check for non-Running pods
NOT_RUNNING=$(kubectl get pods -n "$NAMESPACE" --no-headers | grep -v "Running\|Completed" | wc -l)
if [ "$NOT_RUNNING" -gt 0 ]; then
  echo "WARN: $NOT_RUNNING pods not in Running state"
  kubectl get pods -n "$NAMESPACE" | grep -v "Running\|Completed"
  ERRORS=$((ERRORS + 1))
fi

# Check for pods with high restart counts
HIGH_RESTARTS=$(kubectl get pods -n "$NAMESPACE" -o jsonpath='{.items[*].status.containerStatuses[*].restartCount}' | tr ' ' '\n' | awk '$1 > 5' | wc -l)
if [ "$HIGH_RESTARTS" -gt 0 ]; then
  echo "WARN: $HIGH_RESTARTS containers have restarted more than 5 times"
  ERRORS=$((ERRORS + 1))
fi

# Check for services with no endpoints
for svc in $(kubectl get services -n "$NAMESPACE" -o jsonpath='{.items[*].metadata.name}'); do
  ENDPOINTS=$(kubectl get endpoints "$svc" -n "$NAMESPACE" -o jsonpath='{.subsets[*].addresses[*].ip}' 2>/dev/null)
  if [ -z "$ENDPOINTS" ] && [ "$svc" != "kubernetes" ]; then
    echo "WARN: Service $svc has no endpoints"
    ERRORS=$((ERRORS + 1))
  fi
done

if [ "$ERRORS" -eq 0 ]; then
  echo "OK: All checks passed"
  exit 0
else
  echo "FAIL: $ERRORS issue(s) found"
  exit 1
fi
```

## Further Reading

- [kubectl Cheat Sheet — Kubernetes Docs](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [kubectl Reference Documentation](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands)
- [kubectx and kubens](https://github.com/ahmetb/kubectx) — fast context/namespace switching
- [stern](https://github.com/stern/stern) — multi-pod log tailing with regex
- [k9s](https://k9scli.io/) — terminal UI for Kubernetes

## Summary

`kubectl` is your primary interface to Kubernetes. Core commands:

- `kubectl config use-context` / `kubectx` — switch clusters
- `kubectl get <type>` — list resources; `-o wide`, `-o yaml`, `-l label` for filtering
- `kubectl describe <type> <name>` — full details including Events (use this when debugging)
- `kubectl logs <pod>` — application output; `--previous` for crashed containers; `-f` to stream
- `kubectl exec -it <pod> -- /bin/sh` — interactive shell inside a container
- `kubectl port-forward service/<name> <local>:<remote>` — tunnel to a pod for local debugging
- `kubectl apply -f <file>` — create or update resources (always prefer over `create`)
- `kubectl diff -f <file>` — see what would change before applying
- `kubectl rollout status deployment/<name>` — watch a deployment roll out

Always use `apply` not `create`. Keep changes in version control. Use dry-run before destructive operations. Install `kubectx`, `stern`, and `k9s` to move faster.
