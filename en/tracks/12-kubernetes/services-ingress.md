# Services & Ingress

## Overview

Pods are ephemeral — they get created and destroyed as your cluster scales up and down. Every new Pod gets a new IP address. If you hardcode Pod IPs, your application breaks every time a Pod restarts. Kubernetes Services solve this by providing a stable virtual IP (ClusterIP) and DNS name that routes traffic to healthy Pods.

Ingress goes a level higher: it exposes HTTP and HTTPS routes from outside the cluster to Services inside it, with host-based routing, path routing, and TLS termination — all configured declaratively.

This chapter covers all Service types, DNS-based service discovery, Ingress with nginx-ingress, and TLS automation with cert-manager.

## Prerequisites

- Running Kubernetes cluster
- kubectl basics (see `kubectl-basics.md`)
- Pods and Deployments (see `pods-deployments.md`)
- Basic networking (DNS, TCP/IP, ports, TLS)

## Core Concepts

### Service Types

**ClusterIP** (default) — Internal only. Creates a virtual IP reachable from within the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-server
  namespace: production
spec:
  type: ClusterIP          # default — can omit
  selector:
    app.kubernetes.io/name: api-server
  ports:
  - name: http
    port: 80               # port clients use
    targetPort: 3000       # port on the Pod
    protocol: TCP
```

**NodePort** — Exposes the service on each Node's IP at a static port (30000-32767).

```yaml
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 3000
    nodePort: 30080      # optional: if omitted, Kubernetes assigns one
```

Access: `<any-node-ip>:30080`. Used for development or when you manage your own load balancer externally. Not recommended for production — exposes a port on every node, bypasses Ingress.

**LoadBalancer** — Provisions a cloud load balancer (AWS ELB, GCP LB, Azure LB). Each Service gets its own external IP.

```yaml
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
  - port: 443
    targetPort: 3000
```

Expensive at scale — one cloud LB per Service. Use Ingress instead to share one LB across many services.

**ExternalName** — Maps a Service to an external DNS name (CNAME). No proxying — purely DNS.

```yaml
spec:
  type: ExternalName
  externalName: database.us-east-1.rds.amazonaws.com
```

Access: `my-db-service.namespace.svc.cluster.local` resolves to the external hostname. Useful for migrating from in-cluster to external services gradually.

**Headless Service** — No ClusterIP. Returns Pod IPs directly via DNS. Used by StatefulSets to provide stable, per-pod DNS names.

```yaml
spec:
  clusterIP: None     # makes it headless
  selector:
    app: cassandra
```

### How Services Route Traffic

A Service uses a **selector** to find Pods. Kubernetes maintains an **Endpoints** (or EndpointSlice) object — a live list of Pod IPs and ports for all healthy, ready Pods matching the selector.

```bash
# See the Endpoints for a service
kubectl get endpoints api-server -n production

# NAME        ENDPOINTS                                         AGE
# api-server  10.0.1.5:3000,10.0.1.6:3000,10.0.1.7:3000      5h
```

`kube-proxy` on each node watches Endpoints and programs iptables/IPVS rules to route traffic to any of the Pod IPs. When a Pod becomes unready (readiness probe fails), it's removed from Endpoints and stops receiving traffic.

### DNS-Based Service Discovery

Every Service gets a DNS name automatically:

```
<service-name>.<namespace>.svc.cluster.local
```

Within the same namespace, you can use just the service name:

```typescript
// From a pod in namespace 'production':
await fetch('http://api-server/health');          // short form
await fetch('http://api-server.production/health'); // namespace-qualified
await fetch('http://api-server.production.svc.cluster.local/health'); // FQDN
```

Cross-namespace:
```typescript
// From a pod in namespace 'frontend' accessing backend in 'production':
await fetch('http://api-server.production.svc.cluster.local/v1/users');
```

For headless services (StatefulSets), each Pod also gets:
```
<pod-name>.<service-name>.<namespace>.svc.cluster.local
```

This lets StatefulSet members discover each other:
```
cassandra-0.cassandra.production.svc.cluster.local
cassandra-1.cassandra.production.svc.cluster.local
cassandra-2.cassandra.production.svc.cluster.local
```

### Ingress

Ingress is an API object that defines HTTP/HTTPS routing rules. It requires an **Ingress Controller** (not built into Kubernetes itself) to implement those rules.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls-cert   # Secret with TLS cert + key
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-server
            port:
              number: 80
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-server-v2
            port:
              number: 80
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-ui
            port:
              number: 80
```

**Path types:**
- `Prefix` — matches any path starting with the given prefix (`/v1` matches `/v1`, `/v1/users`, `/v1/orders`)
- `Exact` — matches only the exact path (`/v1` matches only `/v1`, not `/v1/users`)
- `ImplementationSpecific` — behavior defined by the Ingress controller

### Installing nginx-ingress

```bash
# Helm install (production-ready)
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.replicaCount=2 \
  --set controller.metrics.enabled=true \
  --set controller.podAnnotations."prometheus\.io/scrape"=true

# Get the external IP (cloud LoadBalancer)
kubectl get service ingress-nginx-controller -n ingress-nginx
# Point your DNS records to this IP
```

### TLS with cert-manager

cert-manager automates TLS certificate provisioning from Let's Encrypt (or any ACME CA).

```bash
# Install cert-manager
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

```yaml
# ClusterIssuer — Let's Encrypt production
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
    - http01:
        ingress:
          class: nginx
```

```yaml
# Ingress with automatic TLS via cert-manager
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls-cert   # cert-manager creates this Secret
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-server
            port:
              number: 80
```

cert-manager watches for Ingress objects with its annotation, provisions the cert via HTTP-01 challenge, and stores it in the referenced Secret. It also renews certs automatically before expiry.

## Hands-On Examples

### Service for a Node.js API

```yaml
# k8s/api-server/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: api-server
  namespace: production
  labels:
    app.kubernetes.io/name: api-server
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: api-server
  ports:
  - name: http
    port: 80
    targetPort: 3000
```

```bash
# Apply
kubectl apply -f k8s/api-server/service.yaml

# Verify endpoints (should list pod IPs)
kubectl get endpoints api-server -n production

# Test from within the cluster
kubectl run test --image=curlimages/curl --rm -it --restart=Never \
  -- curl http://api-server.production/health
```

### Multi-Service Ingress

```yaml
# k8s/ingress.yaml — routes multiple services through one load balancer
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  namespace: production
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    - app.example.com
    - admin.example.com
    secretName: example-com-tls
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-server
            port:
              number: 80
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-ui
            port:
              number: 80
```

### Testing Service Discovery

```typescript
// src/services/user.service.ts
// In-cluster service discovery via DNS
const USER_SERVICE_URL = process.env.USER_SERVICE_URL
  ?? 'http://user-service.services.svc.cluster.local';

export async function getUserById(id: string) {
  const response = await fetch(`${USER_SERVICE_URL}/users/${id}`);
  if (!response.ok) {
    throw new Error(`User service error: ${response.statusText}`);
  }
  return response.json();
}
```

### Rate Limiting with nginx-ingress

```yaml
metadata:
  annotations:
    # Rate limiting: 10 requests per second per IP
    nginx.ingress.kubernetes.io/limit-rps: "10"
    nginx.ingress.kubernetes.io/limit-connections: "20"
    # Custom response for rate-limited requests
    nginx.ingress.kubernetes.io/limit-req-status-code: "429"
    # Whitelist internal IPs
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8"
```

### Authentication with nginx-ingress (Basic Auth)

```bash
# Create htpasswd file
htpasswd -c auth admin
kubectl create secret generic basic-auth \
  --from-file=auth \
  -n monitoring
```

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
```

## Common Patterns & Best Practices

### Service Naming Conventions

```yaml
# Name the service the same as the deployment
# This makes DNS predictable: my-api.production.svc.cluster.local
metadata:
  name: my-api   # same as deployment name

# Use named ports — allows changing the actual port without updating all consumers
ports:
- name: http      # consumers use port name, not number
  port: 80
  targetPort: http   # matches containerPort name in pod spec
```

### Session Affinity

For applications that don't support distributed sessions, enable sticky sessions:

```yaml
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600    # 1 hour
```

Note: sticky sessions hurt load distribution. Better: use a shared session store (Redis).

### Internal vs External Services

```yaml
# Public API: exposed via Ingress
kind: Service
metadata:
  name: api-server
  annotations:
    # Prevent external load balancer provisioning
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"

---
# Internal-only database service
kind: Service
metadata:
  name: postgres
spec:
  type: ClusterIP    # not accessible from outside cluster
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
```

### NetworkPolicy to Restrict Service Access

Restrict which services can reach your database:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-access
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
  - Ingress
  ingress:
  - from:
    # Only allow api-server and worker pods
    - podSelector:
        matchLabels:
          app.kubernetes.io/name: api-server
    - podSelector:
        matchLabels:
          app.kubernetes.io/name: worker
    ports:
    - protocol: TCP
      port: 5432
```

## Anti-Patterns to Avoid

**Using NodePort in production**
NodePort exposes a port on every node, bypasses Ingress, and has no TLS termination. Use ClusterIP + Ingress instead.

**One LoadBalancer Service per application**
Each LoadBalancer provisions a cloud load balancer ($$$). Use one Ingress with one LoadBalancer Service for nginx-ingress instead.

**Hardcoding Pod IPs**
Pod IPs change every restart. Always use the Service DNS name.

**Skipping TLS on internal services**
Traffic between services inside a cluster is unencrypted by default. For sensitive data (auth tokens, PII), use mutual TLS with a service mesh (Istio, Linkerd) or at minimum TLS at the Ingress level.

**Missing timeout annotations on Ingress**
```yaml
# Bad: default 60s proxy timeout breaks long-running requests
# Good: set timeouts explicitly
nginx.ingress.kubernetes.io/proxy-read-timeout: "300"   # for file uploads
nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
nginx.ingress.kubernetes.io/proxy-connect-timeout: "5"
```

## Debugging & Troubleshooting

### Service Not Routing Traffic

```bash
# Step 1: Check endpoints — if empty, selector doesn't match any pod
kubectl get endpoints my-service -n production
# If ENDPOINTS column shows <none>:

# Step 2: Check service selector
kubectl describe service my-service -n production | grep Selector

# Step 3: Check pod labels
kubectl get pods -n production --show-labels | grep app=my-app
# If pod labels don't match service selector → fix the mismatch

# Step 4: Check pod readiness (only Ready pods appear in Endpoints)
kubectl get pods -n production -l app=my-app
# If pods show 0/1 READY → readiness probe is failing
# kubectl describe pod <name> to see why
```

### Ingress Not Working

```bash
# Check Ingress controller is running
kubectl get pods -n ingress-nginx

# Check Ingress resource
kubectl describe ingress my-ingress -n production

# Check nginx-ingress logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

# Test with port-forward to bypass DNS/LB issues
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8443:443
curl -k -H "Host: api.example.com" https://localhost:8443/health
```

### Certificate Not Provisioned

```bash
# Check cert-manager Certificate resource
kubectl get certificate -n production
kubectl describe certificate api-tls-cert -n production

# Check the CertificateRequest
kubectl get certificaterequests -n production

# Check cert-manager logs
kubectl logs -n cert-manager deployment/cert-manager

# Common issue: HTTP-01 challenge failing
# The Ingress controller must be reachable from Let's Encrypt servers
# at http://<your-domain>/.well-known/acme-challenge/<token>
```

### Testing DNS Resolution

```bash
# From inside a pod
kubectl run dns-test --image=busybox --rm -it --restart=Never \
  -- nslookup api-server.production.svc.cluster.local

# Test cross-namespace
kubectl run dns-test --image=busybox --rm -it --restart=Never \
  -- nslookup postgres.database.svc.cluster.local
```

## Real-World Scenarios

### Scenario 1: Microservices with Shared Ingress

Three microservices behind a single Ingress:

```
api.acme.com/v1/*      → api-v1-service:80
api.acme.com/v2/*      → api-v2-service:80
app.acme.com/*         → frontend-service:80
admin.acme.com/*       → admin-service:80 (basic auth protected)
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: acme-ingress
  namespace: production
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts: [api.acme.com, app.acme.com, admin.acme.com]
    secretName: acme-tls
  rules:
  - host: api.acme.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service: { name: api-v1, port: { number: 80 } }
      - path: /v2
        pathType: Prefix
        backend:
          service: { name: api-v2, port: { number: 80 } }
  - host: app.acme.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: { name: frontend, port: { number: 80 } }
```

### Scenario 2: Blue-Green Switch via Service Selector

```yaml
# Blue deployment (current)
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-blue
spec:
  template:
    metadata:
      labels:
        app: api
        slot: blue
        version: "2.4.1"

# Green deployment (new version)
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-green
spec:
  template:
    metadata:
      labels:
        app: api
        slot: green
        version: "2.5.0"

# Service — switch between blue and green by updating the slot label
---
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
    slot: blue    # change to 'green' to switch all traffic instantly
```

```bash
# Switch to green (instant, no rolling update)
kubectl patch service api \
  -p '{"spec":{"selector":{"slot":"green"}}}'

# Roll back (instant)
kubectl patch service api \
  -p '{"spec":{"selector":{"slot":"blue"}}}'
```

## Further Reading

- [Kubernetes Services — Official Docs](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Ingress — Official Docs](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [nginx-ingress Documentation](https://kubernetes.github.io/ingress-nginx/)
- [cert-manager Documentation](https://cert-manager.io/docs/)
- [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)

## Summary

Services give Pods a stable identity (virtual IP + DNS name) independent of Pod lifecycles. Traffic is routed to healthy, ready Pods via Endpoints.

Service types:
- **ClusterIP** — internal only; default and most common
- **NodePort** — exposes on every node; avoid in production
- **LoadBalancer** — one cloud LB per service; use sparingly
- **ExternalName** — CNAME to external DNS; for migration or abstraction

Ingress routes external HTTP/HTTPS to internal Services via one cloud LoadBalancer:
- Install nginx-ingress via Helm
- Add `cert-manager` for automatic Let's Encrypt TLS
- Configure routing by host and path
- Use annotations for rate limiting, auth, timeouts

Debugging checklist:
1. `kubectl get endpoints <service>` — empty means selector mismatch
2. `kubectl get pods --show-labels` — verify labels match service selector
3. `kubectl describe ingress` — check routing rules and TLS config
4. `kubectl describe certificate` — check cert-manager provisioning status
5. `kubectl logs -n ingress-nginx` — nginx controller logs for routing errors
