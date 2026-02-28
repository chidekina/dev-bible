# Kubernetes in Production

## Overview

Running Kubernetes in production means layering security, reliability, and operational controls on top of the basic Pods and Services covered earlier. A cluster that works in staging can fail in production because of missing RBAC (privilege escalation), missing NetworkPolicies (unrestricted lateral movement), missing Pod Disruption Budgets (rolling updates take the service down), or missing resource quotas (one team's runaway deployment starving another team's workloads).

This chapter covers the controls that separate a production-ready cluster from a development cluster: RBAC for access control, NetworkPolicy for network segmentation, PodDisruptionBudget for reliability, HPA for autoscaling, resource quotas for multi-tenant fairness, and a multi-namespace strategy.

## Prerequisites

- Pods, Deployments, Services (previous chapters)
- kubectl basics
- Helm basics (for resource deployment patterns)

## Core Concepts

### RBAC — Role-Based Access Control

Kubernetes RBAC controls who can do what to which resources. Every API request is authenticated (who are you?) and then authorized (are you allowed?).

The four RBAC objects:
- **Role** — namespaced set of permissions
- **ClusterRole** — cluster-wide set of permissions (or applies across all namespaces)
- **RoleBinding** — binds a Role (or ClusterRole) to a user/group/ServiceAccount within a namespace
- **ClusterRoleBinding** — binds a ClusterRole to a user/group/ServiceAccount cluster-wide

**Subjects** — who the binding applies to:
- `User` — human user (managed by auth provider)
- `Group` — group of users
- `ServiceAccount` — identity for a Pod or process

```yaml
# Role: namespaced permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-reader
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
```

```yaml
# RoleBinding: bind the role to a user in a namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-deployment-reader
  namespace: production
subjects:
- kind: User
  name: alice
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: deployment-reader
  apiGroup: rbac.authorization.k8s.io
```

```yaml
# ServiceAccount RBAC — Pods use ServiceAccounts for API access
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api-server
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: api-server-role
  namespace: production
rules:
# Only allow reading ConfigMaps (for dynamic config)
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: api-server-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: api-server
  namespace: production
roleRef:
  kind: Role
  name: api-server-role
  apiGroup: rbac.authorization.k8s.io
```

**Common ClusterRoles** built into Kubernetes:
| ClusterRole | Access |
|-------------|--------|
| `cluster-admin` | Full control of everything |
| `admin` | Full control in a namespace |
| `edit` | Read/write most namespaced resources |
| `view` | Read-only on most namespaced resources |

### NetworkPolicy

By default, all Pods can communicate with all other Pods in the cluster. NetworkPolicy restricts this — think of it as a firewall for pod-to-pod traffic.

NetworkPolicy requires a CNI plugin that supports it (Calico, Cilium, Weave Net). Flannel does not support NetworkPolicy.

```yaml
# Default deny all ingress and egress in a namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}         # applies to all pods
  policyTypes:
  - Ingress
  - Egress
  # No ingress/egress rules = deny all
```

```yaml
# Allow only api-server to reach postgres
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-allow-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app.kubernetes.io/name: api-server
    ports:
    - protocol: TCP
      port: 5432
```

```yaml
# Allow api-server egress to postgres and Redis, and to internet (for external APIs)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-server-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: api-server
  policyTypes:
  - Egress
  egress:
  # To postgres
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - port: 5432
  # To Redis
  - to:
    - podSelector:
        matchLabels:
          app: redis
    ports:
    - port: 6379
  # To DNS (always needed)
  - to:
    - namespaceSelector: {}
    ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
  # To external services (internet)
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8      # exclude internal cluster IPs
        - 172.16.0.0/12
        - 192.168.0.0/16
    ports:
    - port: 443
```

### PodDisruptionBudget (PDB)

A PDB limits how many Pods of a Deployment/StatefulSet can be simultaneously unavailable during voluntary disruptions (rolling updates, node drains, cluster upgrades). It does NOT protect against involuntary disruptions (node failures, OOMKill).

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-server-pdb
  namespace: production
spec:
  # At least 2 pods always available
  minAvailable: 2

  # OR: at most 1 pod unavailable at a time
  # maxUnavailable: 1

  # OR: percentage
  # minAvailable: "80%"

  selector:
    matchLabels:
      app.kubernetes.io/name: api-server
```

A PDB of `minAvailable: 2` with `replicas: 3` means:
- Rolling update: replaces one pod at a time (blocks at 1 unavailable until replacement is ready)
- `kubectl drain node`: node drain waits until replacement pod is Running before evicting

```bash
# Check PDB status
kubectl get pdb -n production
# NAME           MIN AVAILABLE  MAX UNAVAILABLE  ALLOWED DISRUPTIONS  AGE
# api-server-pdb  2              N/A              1                    5h
# ALLOWED DISRUPTIONS = current replicas - minAvailable = 3 - 2 = 1
```

### HPA — Horizontal Pod Autoscaler

HPA automatically scales the number of Pod replicas based on observed metrics.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-server-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 2
  maxReplicas: 20
  metrics:
  # Scale on CPU utilization
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70   # target 70% CPU across all pods
  # Scale on memory utilization
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  # Scale on custom metric (e.g., requests per second from Prometheus)
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # wait 5 min before scaling down
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60    # max 2 pods removed per minute
    scaleUp:
      stabilizationWindowSeconds: 0     # scale up immediately
      policies:
      - type: Pods
        value: 4
        periodSeconds: 60    # max 4 pods added per minute
```

HPA requires `metrics-server` to be installed:
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

For custom metrics, use the Prometheus Adapter with Prometheus installed in the cluster.

### Resource Quotas

ResourceQuota limits the total resources consumed in a namespace — prevents one team from consuming all cluster capacity.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    # Pod count limits
    pods: "50"
    # Compute resource limits
    requests.cpu: "10"       # total CPU requests in namespace
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    # Object count limits
    configmaps: "20"
    secrets: "20"
    services: "10"
    services.loadbalancers: "2"
    persistentvolumeclaims: "10"
```

```yaml
# LimitRange: sets default requests/limits for pods that don't specify them
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
  - type: Container
    default:
      cpu: 200m
      memory: 256Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    max:
      cpu: "2"
      memory: 2Gi
    min:
      cpu: 50m
      memory: 64Mi
```

### Multi-Namespace Strategy

Structure namespaces around teams and environments:

```
Cluster
├── kube-system          (cluster components — don't touch)
├── ingress-nginx        (ingress controller)
├── cert-manager         (TLS automation)
├── monitoring           (Prometheus, Grafana)
├── platform-production  (platform team — production)
├── platform-staging     (platform team — staging)
├── api-production       (API team — production)
├── api-staging          (API team — staging)
└── data-production      (data team — production)
```

Each namespace gets:
- RBAC bindings (team has `edit` in their namespaces, `view` in others)
- ResourceQuota (CPU/memory budget per namespace)
- LimitRange (default requests/limits)
- NetworkPolicy (default deny, explicit allow rules)

## Hands-On Examples

### Checking What You Can Do

```bash
# What can I do in this namespace?
kubectl auth can-i --list -n production

# Can I delete deployments?
kubectl auth can-i delete deployments -n production

# What can the api-server ServiceAccount do?
kubectl auth can-i --list \
  --as=system:serviceaccount:production:api-server \
  -n production
```

### Setting Up a New Team Namespace

```bash
#!/bin/bash
# scripts/setup-namespace.sh
TEAM=$1
ENV=$2
NS="${TEAM}-${ENV}"

# Create namespace
kubectl create namespace "$NS"

# Apply default deny NetworkPolicy
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: $NS
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF

# Apply ResourceQuota
kubectl apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ${NS}-quota
  namespace: $NS
spec:
  hard:
    pods: "30"
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
EOF

# Apply LimitRange
kubectl apply -f - <<EOF
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: $NS
spec:
  limits:
  - type: Container
    default:
      cpu: 200m
      memory: 256Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
EOF

# Bind team group to edit role
kubectl create rolebinding "${TEAM}-editors" \
  --clusterrole=edit \
  --group="${TEAM}-team" \
  --namespace="$NS"

echo "Namespace $NS configured"
```

### Implementing Security Context Best Practices

```yaml
# Secure pod spec
spec:
  # Disable auto-mounting service account token (least privilege)
  automountServiceAccountToken: false
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: api
    image: my-app:1.0.0
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
    volumeMounts:
    # Writable temp dir when readOnlyRootFilesystem: true
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
```

### Monitoring HPA Behavior

```bash
# Watch HPA scale in real time
kubectl get hpa api-server-hpa -n production -w

# Generate load to trigger scaling (from inside cluster)
kubectl run load-test --image=busybox --rm -it --restart=Never \
  -- sh -c "while true; do wget -qO- http://api-server.production/; done"

# Check events for scaling decisions
kubectl describe hpa api-server-hpa -n production
```

## Common Patterns & Best Practices

### The Principle of Least Privilege for ServiceAccounts

```yaml
# Each workload gets its own ServiceAccount
# Only bind the exact permissions it needs

# api-server needs to read configmaps
kind: Role
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "watch", "list"]
  resourceNames: ["api-config"]  # restrict to specific ConfigMap

# worker needs to write to a queue ConfigMap
kind: Role
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "update", "patch"]
  resourceNames: ["job-queue"]
```

### Namespace-Level Default NetworkPolicy

Apply this to every new namespace as part of setup:

```yaml
# Allow DNS everywhere (kube-dns), deny everything else by default
# Then add explicit allows for each service

---
# Allow DNS egress from everything
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP

---
# Deny all other ingress and egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### Taints and Tolerations for Node Pools

Separate workloads across dedicated node pools:

```yaml
# Taint a node pool (done by cloud provider or manually)
kubectl taint nodes node-pool-gpu gpu=true:NoSchedule

# Add toleration to pods that should run on GPU nodes
spec:
  tolerations:
  - key: gpu
    operator: Equal
    value: "true"
    effect: NoSchedule
  nodeSelector:
    node-pool: gpu          # or use nodeAffinity for more control
```

## Anti-Patterns to Avoid

**Using `cluster-admin` for everything**
```bash
# Bad: all ServiceAccounts are cluster admins
kubectl create clusterrolebinding default-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=default:default

# Good: assign minimum required permissions per ServiceAccount
```

**No NetworkPolicy (flat network)**
Without NetworkPolicy, any compromised pod can reach any other pod, including databases and secrets stores. Always apply default-deny and explicit allows.

**No PodDisruptionBudget for stateful services**
```bash
# A cluster upgrade or node drain without PDB can take down all replicas simultaneously
# Always add PDB for services with replicas > 1
```

**HPA without requests set**
```yaml
# Bad: HPA can't calculate % CPU/memory without requests
containers:
- name: api
  image: my-app:1.0.0
  # No resources.requests → HPA cannot function

# Good: always set requests for HPA targets
resources:
  requests:
    cpu: 100m
    memory: 256Mi
```

**Wildcard RBAC permissions**
```yaml
# Bad: grants full access to all resources
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]

# Good: specific resources and verbs
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "update", "patch"]
```

## Debugging & Troubleshooting

### RBAC Debugging

```bash
# Why is a request forbidden?
kubectl auth can-i create pods -n production \
  --as=system:serviceaccount:production:api-server
# Returns yes/no

# Get detailed RBAC view for a ServiceAccount
kubectl get rolebindings,clusterrolebindings -A \
  -o json | jq '.items[] | select(
    .subjects[]? |
    (.kind=="ServiceAccount" and .name=="api-server" and .namespace=="production")
  ) | .metadata.name'

# Use kubectl-who-can (krew plugin)
kubectl krew install who-can
kubectl who-can create pods -n production
```

### NetworkPolicy Testing

```bash
# Test connectivity between two pods
kubectl run nettest --image=busybox --rm -it --restart=Never \
  --overrides='{"spec":{"serviceAccountName":"default"}}' \
  -- sh

# Inside the pod:
wget -qO- http://postgres.production:5432   # should fail with NetworkPolicy
wget -qO- http://api-server.production:80   # should succeed
nc -zv redis.production 6379               # TCP test

# Or with a named pod:
kubectl exec api-server-pod -- nc -zv postgres.production 5432
```

### HPA Not Scaling

```bash
# Check metrics-server is working
kubectl top pods -n production

# Check HPA status
kubectl describe hpa api-server-hpa -n production
# Look for: "Warning: FailedGetResourceMetric" → metrics-server issue
# Look for: "DesiredReplicas: X, CurrentReplicas: X" → might be at max already

# Verify requests are set (HPA requirement)
kubectl get pods -n production -o jsonpath='{.items[*].spec.containers[*].resources}'
```

### ResourceQuota Blocking Deployments

```bash
# See quota usage
kubectl describe resourcequota production-quota -n production

# See what's consuming the quota
kubectl get pods -n production -o custom-columns=\
  'NAME:.metadata.name,CPU-REQ:.spec.containers[*].resources.requests.cpu,MEM-REQ:.spec.containers[*].resources.requests.memory'
```

## Real-World Scenarios

### Scenario 1: Production RBAC for a Development Team

```bash
# Create roles for different responsibilities
kubectl apply -f - <<EOF
# Developers: read-only in production
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-view
  namespace: production
subjects:
- kind: Group
  name: dev-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
---
# Developers: full edit in staging
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-edit
  namespace: staging
subjects:
- kind: Group
  name: dev-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
---
# On-call: can restart deployments in production
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: restart-deployments
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "update", "patch"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "delete"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
EOF
```

### Scenario 2: Complete Production Namespace Hardening

```bash
# Apply the full production hardening stack
kubectl apply -f k8s/production/namespace-config/

# k8s/production/namespace-config/ contains:
# - resourcequota.yaml
# - limitrange.yaml
# - networkpolicy-default-deny.yaml
# - networkpolicy-allow-dns.yaml
# - networkpolicy-allow-ingress-controller.yaml
# Per-service network policies are applied with each service deployment
```

### Scenario 3: Graceful Node Drain (Rolling Cluster Upgrade)

```bash
# Cordon: stop new pods being scheduled on this node
kubectl cordon node-1

# Drain: evict all pods (respects PDBs — won't violate minAvailable)
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data

# Node is now safe to upgrade OS, Kubernetes version, etc.
# ...upgrade node...

# Uncordon: allow scheduling again
kubectl uncordon node-1

# If drain hangs: check which PDB is blocking
kubectl get pdb -A
# If PDB is misconfigured (minAvailable > replicas), fix it:
kubectl patch pdb api-server-pdb -n production \
  -p '{"spec":{"minAvailable":1}}'
```

## Further Reading

- [RBAC Authorization — Kubernetes Docs](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Network Policies — Kubernetes Docs](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Pod Disruption Budgets](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
- [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [NSA/CISA Kubernetes Hardening Guide](https://media.defense.gov/2022/Aug/29/2003066362/-1/-1/0/CTR_KUBERNETES_HARDENING_GUIDANCE_1.2_20220829.PDF)

## Summary

Production Kubernetes requires security and reliability controls layered on top of basic workloads.

**RBAC** — who can do what:
- Principle of least privilege: each ServiceAccount gets only the permissions it needs
- Use built-in ClusterRoles (`view`, `edit`, `admin`) for namespaced access
- Never use `cluster-admin` for application service accounts
- `kubectl auth can-i` to verify permissions

**NetworkPolicy** — what can talk to what:
- Default-deny all ingress and egress in every namespace
- Explicitly allow: ingress-controller → web/api, api → database, api → cache
- Always allow DNS egress (port 53)

**PodDisruptionBudget** — service continuity during disruptions:
- `minAvailable: 2` or `maxUnavailable: 1` for all production Deployments with replicas > 1
- Without PDB: cluster upgrade or node drain can evict all pods at once

**HPA** — right-sizing under load:
- Requires `resources.requests` to be set
- Configure `scaleDown.stabilizationWindowSeconds` (300s) to prevent thrashing
- Test scaling behavior before production cutover

**ResourceQuota + LimitRange** — multi-tenant fairness:
- Quota limits total namespace consumption
- LimitRange sets defaults for pods without explicit requests/limits

**Namespace strategy** — isolation and organization:
- Separate namespaces by team × environment
- Apply RBAC, NetworkPolicy, and ResourceQuota to every new namespace
- Use a setup script to bootstrap new namespaces consistently
