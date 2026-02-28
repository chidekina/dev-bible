# Pods & Deployments

## Overview

Pods and Deployments are the foundational workload objects in Kubernetes. A Pod is the smallest schedulable unit — it wraps one or more containers that share a network and storage context. A Deployment manages a set of identical Pods, providing declarative updates, rollout history, and automatic rollback.

In practice you never create Pods directly. You define a Deployment (or StatefulSet, DaemonSet, etc.) and let it manage Pods. Understanding the Pod spec in depth — resource requests/limits, liveness/readiness probes, environment variables, volume mounts — is what separates "it works" from "it works reliably in production."

## Prerequisites

- Docker basics (images, ports, environment variables)
- kubectl installed and configured
- A Kubernetes cluster (kind or minikube for local)

## Core Concepts

### Pod Anatomy

A Pod spec defines the containers to run and their configuration:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-server
  namespace: production
  labels:
    app: api-server
    version: "2.4"
spec:
  containers:
  - name: api
    image: my-org/api-server:2.4.1

    # Port declaration (informational — doesn't control network)
    ports:
    - name: http
      containerPort: 3000
      protocol: TCP

    # Environment variables
    env:
    - name: NODE_ENV
      value: production
    - name: DATABASE_URL
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: url
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: logLevel

    # Resource budget
    resources:
      requests:
        cpu: 100m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 512Mi

    # Health checks
    livenessProbe:
      httpGet:
        path: /health
        port: 3000
      initialDelaySeconds: 10
      periodSeconds: 30
      failureThreshold: 3

    readinessProbe:
      httpGet:
        path: /ready
        port: 3000
      initialDelaySeconds: 5
      periodSeconds: 10
      failureThreshold: 3

    # Volume mounts
    volumeMounts:
    - name: config
      mountPath: /app/config
      readOnly: true

  # Volumes available to all containers
  volumes:
  - name: config
    configMap:
      name: app-config

  # How long to wait for graceful shutdown (default: 30s)
  terminationGracePeriodSeconds: 60

  # Restart policy (Always, OnFailure, Never — default: Always)
  restartPolicy: Always

  # Service account for RBAC
  serviceAccountName: api-server
```

### Resource Requests vs Limits

This is one of the most misunderstood aspects of Kubernetes.

**Requests** — what the scheduler reserves. A pod with `memory: 256Mi` requested won't be placed on a node with less than 256Mi available. This is what the scheduler sees.

**Limits** — the ceiling. If a container exceeds its memory limit, it gets OOMKilled. If it exceeds its CPU limit, it gets throttled (not killed).

```
Node has 4 CPU, 8Gi memory
Pods scheduled:
  Pod A: request 1CPU/2Gi  limit 2CPU/4Gi
  Pod B: request 1CPU/2Gi  limit 2CPU/4Gi
  Pod C: request 1CPU/2Gi  limit 2CPU/4Gi  ← scheduler: 3CPU/6Gi reserved, OK
  Pod D: request 1CPU/2Gi              ← scheduler: 4CPU/8Gi reserved — FULL
  Pod E: request 0.5CPU/1Gi            ← Pending: not enough reserved capacity
```

CPU units:
- `1` = 1 vCPU
- `0.5` = half a vCPU
- `500m` = 500 millicores = 0.5 vCPU
- `100m` = 100 millicores = 0.1 vCPU

Memory units:
- `Mi` = mebibytes (1Mi = 1,048,576 bytes)
- `Gi` = gibibytes
- `M` = megabytes (1M = 1,000,000 bytes)
- Use `Mi`/`Gi`, not `M`/`G`

**QoS classes** (derived from requests/limits):
- **Guaranteed** — requests == limits (both set). First to be scheduled, last to be evicted.
- **Burstable** — requests < limits (or only one set). Middle priority.
- **BestEffort** — no requests or limits. Evicted first under pressure.

For production: set both requests and limits. For databases and stateful services: use Guaranteed QoS.

### Liveness vs Readiness Probes

Two probes, two different purposes:

**Liveness probe** — "Is this container alive?"
- Failing = container is stuck, not healthy
- Kubernetes **restarts** the container
- Used for deadlock detection, zombie detection
- Don't make it too aggressive — you'll cause restart loops

**Readiness probe** — "Is this container ready to serve traffic?"
- Failing = temporarily remove this Pod from Service load balancing
- Kubernetes **does not restart** the container
- Used for startup time, temporary overload, dependent service down
- Pod stays running but gets no traffic until probe passes again

**Startup probe** (Kubernetes 1.16+) — "Has the container finished starting up?"
- Delays liveness probes during slow startup
- Prevents premature liveness kills during initialization

```yaml
# Startup probe — disables liveness/readiness until this passes
startupProbe:
  httpGet:
    path: /health
    port: 3000
  failureThreshold: 30    # 30 × 10s = 5 minutes to start
  periodSeconds: 10

# Liveness — running but stuck?
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 0   # startup probe already handled delay
  periodSeconds: 30
  failureThreshold: 3      # 3 failures → restart

# Readiness — ready for traffic?
readinessProbe:
  httpGet:
    path: /ready
    port: 3000
  initialDelaySeconds: 0
  periodSeconds: 10
  failureThreshold: 3      # 3 failures → remove from LB
  successThreshold: 1      # 1 success → add back to LB
```

Probe types:
```yaml
# HTTP GET — most common for web apps
httpGet:
  path: /healthz
  port: 8080
  httpHeaders:
  - name: Custom-Header
    value: Awesome

# TCP socket — for non-HTTP servers
tcpSocket:
  port: 3306

# Exec — run a command
exec:
  command:
  - sh
  - -c
  - "redis-cli ping | grep PONG"

# gRPC — for gRPC services (Kubernetes 1.24+)
grpc:
  port: 9090
```

### Deployments

A Deployment manages a ReplicaSet which manages Pods. You interact with the Deployment — it handles the rest.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
spec:
  replicas: 3

  # Which pods this deployment manages
  selector:
    matchLabels:
      app: api-server

  # Rolling update strategy (default)
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1    # at most 1 pod unavailable during rollout
      maxSurge: 1          # at most 1 extra pod during rollout

  # Pod template
  template:
    metadata:
      labels:
        app: api-server
        version: "2.4"
    spec:
      # ... (same as Pod spec)
```

Rolling update behavior with `replicas: 3`, `maxUnavailable: 1`, `maxSurge: 1`:
```
Start:  [v1] [v1] [v1]
Step 1: [v1] [v1] [v1] [v2]   ← surge: +1 new
Step 2: [v1] [v2] [v2]         ← kill one old, bring one new
Step 3: [v2] [v2] [v2]         ← done
```

**Recreate strategy** — kills all old pods before creating new ones:
```yaml
strategy:
  type: Recreate
```
Use when you can't run two versions simultaneously (e.g., database schema migrations that are not backward compatible). Causes downtime.

### Deployment Rollout Operations

```bash
# Check rollout status
kubectl rollout status deployment/api-server

# See rollout history
kubectl rollout history deployment/api-server

# See details of a specific revision
kubectl rollout history deployment/api-server --revision=3

# Rollback to previous version
kubectl rollout undo deployment/api-server

# Rollback to a specific revision
kubectl rollout undo deployment/api-server --to-revision=2

# Pause a rollout (to do staged rollout manually)
kubectl rollout pause deployment/api-server

# Resume a paused rollout
kubectl rollout resume deployment/api-server

# Trigger a rollout without changing the spec (e.g., force new pods)
kubectl rollout restart deployment/api-server
```

## Hands-On Examples

### Complete Deployment with All Best Practices

```yaml
# k8s/api-server/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
  labels:
    app.kubernetes.io/name: api-server
    app.kubernetes.io/version: "2.4.1"
    app.kubernetes.io/component: backend
    app.kubernetes.io/managed-by: helm
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: api-server
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app.kubernetes.io/name: api-server
        app.kubernetes.io/version: "2.4.1"
      annotations:
        # Force rollout when configmap changes (with helm --set checksum)
        checksum/config: "{{ .Values.configChecksum }}"
    spec:
      serviceAccountName: api-server
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      terminationGracePeriodSeconds: 60
      containers:
      - name: api
        image: my-org/api-server:2.4.1
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 3000
        env:
        - name: NODE_ENV
          value: production
        - name: PORT
          value: "3000"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: api-config
              key: logLevel
        envFrom:
        - configMapRef:
            name: api-config
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        startupProbe:
          httpGet:
            path: /health
            port: 3000
          failureThreshold: 30
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          periodSeconds: 30
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          periodSeconds: 10
          failureThreshold: 3
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
      volumes:
      - name: tmp
        emptyDir: {}
      # Spread across zones for resilience
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app.kubernetes.io/name: api-server
      # Ensure pods don't stack on same node
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app.kubernetes.io/name: api-server
              topologyKey: kubernetes.io/hostname
```

### Node.js Health Check Endpoint

```typescript
// src/health.ts
import { FastifyInstance } from 'fastify';

export async function healthRoutes(app: FastifyInstance) {
  // Liveness: is the process alive?
  app.get('/health', async () => {
    return { status: 'ok', timestamp: new Date().toISOString() };
  });

  // Readiness: can we serve traffic?
  app.get('/ready', async (req, reply) => {
    try {
      // Check database connectivity
      await app.db.raw('SELECT 1');

      // Check Redis connectivity
      await app.redis.ping();

      return { status: 'ready', timestamp: new Date().toISOString() };
    } catch (err) {
      reply.status(503);
      return {
        status: 'not ready',
        error: (err as Error).message,
        timestamp: new Date().toISOString(),
      };
    }
  });
}
```

### Graceful Shutdown

Kubernetes sends SIGTERM before killing a container. Handle it:

```typescript
// src/server.ts
const app = Fastify({ logger: true });

// Register routes
await app.register(routes);

// Start server
await app.listen({ port: 3000, host: '0.0.0.0' });

// Graceful shutdown
async function shutdown(signal: string) {
  app.log.info(`Received ${signal}, shutting down gracefully`);

  try {
    // Stop accepting new requests
    await app.close();
    app.log.info('Server closed');

    // Close database connections
    await db.destroy();
    app.log.info('Database connections closed');

    process.exit(0);
  } catch (err) {
    app.log.error('Error during shutdown:', err);
    process.exit(1);
  }
}

// terminationGracePeriodSeconds gives you time to finish in-flight requests
process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));
```

## Common Patterns & Best Practices

### Init Containers for Setup Tasks

```yaml
spec:
  initContainers:
  # Run database migrations before app starts
  - name: run-migrations
    image: my-org/api-server:2.4.1
    command: ['node', 'dist/migrate.js']
    env:
    - name: DATABASE_URL
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: url

  # Wait for a dependency to be ready
  - name: wait-for-redis
    image: busybox
    command:
    - sh
    - -c
    - |
      until nc -z redis-service 6379; do
        echo "Waiting for Redis..."
        sleep 2
      done
      echo "Redis is ready"

  containers:
  - name: api
    image: my-org/api-server:2.4.1
    # runs only after both init containers succeed
```

### Pod Disruption Budget

Protect against too many pods going down at once (during rolling updates, node drains):

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-server-pdb
spec:
  minAvailable: 2          # At least 2 pods always available
  # OR: maxUnavailable: 1  # At most 1 pod unavailable at a time
  selector:
    matchLabels:
      app.kubernetes.io/name: api-server
```

### Annotations for Change Tracking

```yaml
metadata:
  annotations:
    # Who deployed this
    deployment.kubernetes.io/deployed-by: "github-actions"
    # When
    deployment.kubernetes.io/deployed-at: "2024-01-15T14:30:00Z"
    # From which git commit
    deployment.kubernetes.io/git-commit: "abc123def"
    # Which pipeline run
    deployment.kubernetes.io/pipeline-run: "https://github.com/org/repo/actions/runs/123"
```

## Anti-Patterns to Avoid

**No resource requests or limits**
```yaml
# Bad: scheduler is blind, nodes can be OOMKilled
containers:
- name: api
  image: my-app:latest

# Good
resources:
  requests: { cpu: 100m, memory: 256Mi }
  limits: { cpu: 500m, memory: 512Mi }
```

**No health probes**
```yaml
# Bad: Kubernetes routes traffic to pods that aren't ready,
# and doesn't restart deadlocked pods
containers:
- name: api
  image: my-app:latest

# Good: always add at minimum a readiness probe
readinessProbe:
  httpGet:
    path: /health
    port: 3000
```

**Using `latest` image tag**
```yaml
# Bad: unpredictable, rollback impossible
image: my-app:latest

# Good
image: my-app:2.4.1
```

**Running as root**
```yaml
# Bad: container runs as root by default
containers:
- name: api
  image: my-app:2.4.1

# Good
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
```

**Single replica in production**
```yaml
# Bad: no resilience
replicas: 1

# Good: minimum 2, prefer 3+
replicas: 3
```

## Debugging & Troubleshooting

### Pod Lifecycle Debug

```bash
# Full pod state
kubectl describe pod <name> -n <ns>

# Conditions to check:
# PodScheduled: False → scheduler couldn't place it (resource constraints, taints)
# Initialized: False → init container failed
# ContainersReady: False → readiness probe failing or container crashed
# Ready: False → pod not ready to serve traffic

# Container state:
# Waiting (Reason: CrashLoopBackOff) → container keeps crashing, check logs --previous
# Waiting (Reason: ImagePullBackOff) → can't pull image
# Running → OK
# Terminated (Reason: OOMKilled) → exceeded memory limit, increase it
# Terminated (Exit Code: 1) → app exited with error, check logs
```

### Viewing OOMKill Events

```bash
# See OOMKill events
kubectl get events --field-selector reason=OOMKilling -A

# Or in describe output
kubectl describe pod <name> | grep -A5 "Last State"
# Last State: Terminated
#   Reason: OOMKilled
#   Exit Code: 137
#   Started: ...
#   Finished: ...
```

### Watching a Rollout

```bash
# Monitor rolling update in real time
kubectl rollout status deployment/api-server -n production --watch

# If rollout is hanging, check pod events
kubectl get pods -n production -l app=api-server
kubectl describe pod <new-pending-pod>

# Rollback if something is wrong
kubectl rollout undo deployment/api-server -n production
```

## Real-World Scenarios

### Scenario 1: Zero-Downtime Deployment

Guarantee zero-downtime with correct probe + strategy configuration:

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0    # never take a pod down before replacement is ready
      maxSurge: 1          # create one extra pod first
  template:
    spec:
      containers:
      - name: api
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          # New pod must pass readiness before old one is terminated
          initialDelaySeconds: 5
          periodSeconds: 5
          successThreshold: 2    # pass twice before considered ready
        lifecycle:
          preStop:
            exec:
              # Sleep to allow in-flight requests to finish
              # after SIGTERM is sent but before pod is removed from endpoints
              command: ["/bin/sh", "-c", "sleep 5"]
      terminationGracePeriodSeconds: 30
```

### Scenario 2: Canary Deployment with Labels

```bash
# Deploy canary (10% of pods)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server-canary
spec:
  replicas: 1          # 1 of 10 total = 10% traffic
  selector:
    matchLabels:
      app: api-server
      track: canary
  template:
    metadata:
      labels:
        app: api-server    # same label → Service routes here too
        track: canary
    spec:
      containers:
      - name: api
        image: my-org/api-server:2.5.0-rc1   # new version
        # ... rest of spec
EOF

# Monitor canary
kubectl logs -l track=canary -f
kubectl top pods -l track=canary

# Promote: update stable deployment to new version
kubectl set image deployment/api-server api=my-org/api-server:2.5.0

# Remove canary
kubectl delete deployment api-server-canary
```

## Further Reading

- [Kubernetes Deployments — Official Docs](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Pod Disruption Budgets](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
- [Security Context for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)

## Summary

Pods are the smallest runnable unit; Deployments manage sets of Pods with rolling updates and rollback.

Key Pod spec fields:
- **resources** — always set both `requests` (for scheduling) and `limits` (for isolation). Memory limit violations cause OOMKill.
- **livenessProbe** — is the container stuck? Kubernetes restarts on failure.
- **readinessProbe** — is the container ready for traffic? Kubernetes removes from load balancer on failure.
- **startupProbe** — slow-starting apps; delays liveness checks during initialization.
- **terminationGracePeriodSeconds** — time for graceful shutdown; handle SIGTERM in your app.
- **securityContext** — run as non-root, read-only filesystem, drop all capabilities.

Deployment best practices:
- `replicas: 3+` in production — single replica means any disruption = downtime
- `RollingUpdate` with `maxUnavailable: 0, maxSurge: 1` for zero-downtime
- Never use `latest` tag — use explicit image digests or version tags
- Always add a `PodDisruptionBudget` for critical services
- Use `kubectl rollout undo` for fast rollback — don't fight a bad deployment
