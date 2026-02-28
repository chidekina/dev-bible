# ConfigMaps & Secrets

## Overview

Applications need configuration: database URLs, feature flags, log levels, API endpoints, TLS certificates, and credentials. Kubernetes provides two objects for this purpose:

- **ConfigMap** — non-sensitive configuration (strings, config files, JSON data)
- **Secret** — sensitive data (passwords, tokens, private keys) — stored base64-encoded and encrypted at rest with proper cluster configuration

Both can be injected into Pods as environment variables or mounted as files. This chapter covers creation, injection patterns, the external-secrets-operator for syncing from AWS Secrets Manager / GCP Secret Manager / HashiCorp Vault, and Sealed Secrets for storing encrypted values in git.

## Prerequisites

- Pods and Deployments basics
- kubectl basics
- Basic understanding of YAML and base64 encoding

## Core Concepts

### ConfigMap

A ConfigMap stores key-value pairs. Keys are strings; values can be any string, including multi-line (config files, JSON).

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
  namespace: production
data:
  # Simple key-value pairs
  LOG_LEVEL: "info"
  PORT: "3000"
  FEATURE_NEW_DASHBOARD: "true"

  # File-like entries (multi-line values)
  app.json: |
    {
      "timeout": 5000,
      "retries": 3,
      "baseUrl": "https://api.example.com"
    }

  nginx.conf: |
    server {
      listen 80;
      location / {
        proxy_pass http://localhost:3000;
      }
    }
```

### Secret

Secrets store sensitive data. Values must be base64-encoded in YAML (Kubernetes decodes them before injection).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
type: Opaque       # generic — most common type
data:
  # base64 encoded: echo -n "postgresql://..." | base64
  DATABASE_URL: cG9zdGdyZXNxbDovL2FwcDpzZWNyZXRAcG9zdGdyZXM6NTQzMi9teWRi
  API_KEY: c2VjcmV0LWFwaS1rZXktMTIz
stringData:
  # stringData values are NOT base64 encoded — Kubernetes encodes them
  # Use this for readability in manifests you're creating by hand
  REDIS_PASSWORD: "my-redis-password"
```

Secret types:
| Type | Purpose |
|------|---------|
| `Opaque` | Arbitrary key-value data (default) |
| `kubernetes.io/tls` | TLS certificate and key (`tls.crt`, `tls.key`) |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials |
| `kubernetes.io/service-account-token` | ServiceAccount token (auto-created) |
| `kubernetes.io/ssh-auth` | SSH private key |

### Injection Methods

**Method 1: Individual environment variables**

```yaml
env:
- name: LOG_LEVEL
  valueFrom:
    configMapKeyRef:
      name: api-config
      key: LOG_LEVEL
- name: DATABASE_URL
  valueFrom:
    secretKeyRef:
      name: db-credentials
      key: DATABASE_URL
```

**Method 2: All keys from a ConfigMap/Secret as env vars**

```yaml
envFrom:
- configMapRef:
    name: api-config
- secretRef:
    name: db-credentials
# All keys become environment variables with the same name
```

**Method 3: Volume mount (files)**

```yaml
volumeMounts:
- name: app-config
  mountPath: /app/config
  readOnly: true
- name: tls-certs
  mountPath: /etc/ssl/app
  readOnly: true

volumes:
- name: app-config
  configMap:
    name: api-config
    items:                    # optional: select specific keys
    - key: app.json
      path: app.json          # filename in the mounted directory
- name: tls-certs
  secret:
    secretName: api-tls-cert
    defaultMode: 0400         # restrictive permissions for key files
```

**Volume mounts update automatically** — when a ConfigMap is updated, the files in the volume are updated (with a ~1 minute delay). Environment variables do NOT update — they require a pod restart.

## Hands-On Examples

### Creating Resources with kubectl

```bash
# ConfigMap from literal values
kubectl create configmap api-config \
  --from-literal=LOG_LEVEL=info \
  --from-literal=PORT=3000 \
  --namespace=production

# ConfigMap from a file
kubectl create configmap nginx-config \
  --from-file=nginx.conf \
  --namespace=production

# ConfigMap from a directory (each file becomes a key)
kubectl create configmap app-configs \
  --from-file=./config/ \
  --namespace=production

# Secret from literals
kubectl create secret generic db-credentials \
  --from-literal=DATABASE_URL='postgresql://app:secret@postgres:5432/mydb' \
  --from-literal=REDIS_PASSWORD='redis-secret' \
  --namespace=production

# Secret from a file (e.g., private key)
kubectl create secret generic ssh-key \
  --from-file=id_rsa=~/.ssh/id_rsa \
  --namespace=production

# TLS secret from cert and key files
kubectl create secret tls api-tls-cert \
  --cert=server.crt \
  --key=server.key \
  --namespace=production

# Docker registry credentials
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password='mypassword' \
  --namespace=production
```

### Complete Pod with ConfigMap and Secret Injection

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
      - name: api
        image: my-org/api:1.0.0
        envFrom:
        # Load all non-sensitive config as env vars
        - configMapRef:
            name: api-config
        env:
        # Load individual secrets
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: DATABASE_URL
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: JWT_SECRET
        volumeMounts:
        # Mount a config file
        - name: app-json
          mountPath: /app/config
          readOnly: true
        # Mount TLS cert for HTTPS
        - name: tls
          mountPath: /etc/ssl/app
          readOnly: true
      volumes:
      - name: app-json
        configMap:
          name: api-config
          items:
          - key: app.json
            path: app.json
      - name: tls
        secret:
          secretName: api-tls-cert
          defaultMode: 0400
```

### Reading Mounted Config in Node.js

```typescript
// src/config.ts
import { readFileSync } from 'fs';
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'staging', 'production']).default('development'),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
  JWT_SECRET: z.string().min(32),
});

export const env = envSchema.parse(process.env);

// Read mounted config file
function readAppConfig() {
  try {
    const raw = readFileSync('/app/config/app.json', 'utf-8');
    return JSON.parse(raw);
  } catch {
    return {}; // fallback to defaults
  }
}

export const appConfig = readAppConfig();
```

### Watching for Config File Updates

When ConfigMaps are mounted as volumes, Kubernetes updates the files in place (~1 min). Implement a watcher to reload without restart:

```typescript
import { watch } from 'fs';

const CONFIG_PATH = '/app/config/app.json';
let config = readConfig();

watch(CONFIG_PATH, (event) => {
  if (event === 'change') {
    try {
      config = readConfig();
      console.log('Config reloaded');
    } catch (err) {
      console.error('Failed to reload config:', err);
    }
  }
});

function readConfig() {
  return JSON.parse(require('fs').readFileSync(CONFIG_PATH, 'utf-8'));
}
```

## Common Patterns & Best Practices

### External Secrets Operator

The External Secrets Operator syncs secrets from external stores (AWS Secrets Manager, GCP Secret Manager, Azure Key Vault, HashiCorp Vault) into Kubernetes Secrets automatically.

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace
```

```yaml
# SecretStore — defines connection to the external store
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secretsmanager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        # Use IRSA (IAM Roles for Service Accounts) in EKS
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets

---
# ExternalSecret — maps external secret to a K8s Secret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: ClusterSecretStore
  target:
    name: db-credentials         # name of K8s Secret to create/update
    creationPolicy: Owner
  data:
  - secretKey: DATABASE_URL      # key in K8s Secret
    remoteRef:
      key: production/api/database  # AWS Secrets Manager path
      property: url               # JSON property within the secret
  - secretKey: DATABASE_PASSWORD
    remoteRef:
      key: production/api/database
      property: password
```

### Sealed Secrets (GitOps-friendly)

Sealed Secrets let you store encrypted secrets in git. Only the controller in the cluster can decrypt them.

```bash
# Install controller
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system

# Install kubeseal CLI
brew install kubeseal

# Seal a secret (encrypt for your cluster)
kubectl create secret generic db-credentials \
  --from-literal=DATABASE_URL='postgresql://...' \
  --dry-run=client \
  -o yaml | \
  kubeseal \
    --controller-name=sealed-secrets \
    --controller-namespace=kube-system \
    --format yaml \
  > sealed-db-credentials.yaml

# Commit to git (safe — only your cluster can decrypt)
git add sealed-db-credentials.yaml
git commit -m "chore: add sealed database credentials"
```

```yaml
# sealed-db-credentials.yaml — safe to store in git
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  encryptedData:
    DATABASE_URL: AgBy8hBl...  # encrypted ciphertext
```

### Namespace-Scoped vs Cluster-Wide Secrets

```yaml
# Default: namespace-scoped — only pods in 'production' can use it
metadata:
  namespace: production

# For shared secrets (e.g., registry credentials): replicate to each namespace
# Or use ClusterSecretStore with External Secrets Operator
```

### Environment Separation Pattern

```
secrets/
  production/
    db-credentials.yaml       # sealed or external reference
    app-secrets.yaml
  staging/
    db-credentials.yaml
    app-secrets.yaml

configmaps/
  base/
    api-config.yaml           # shared defaults
  production/
    api-config-patch.yaml     # production overrides
  staging/
    api-config-patch.yaml     # staging overrides
```

## Anti-Patterns to Avoid

**Storing plaintext secrets in git**
```yaml
# NEVER do this
data:
  DATABASE_PASSWORD: mypassword123   # plaintext in git = breach waiting to happen

# Use Sealed Secrets or External Secrets Operator
```

**Using ConfigMaps for secrets**
```yaml
# Bad: ConfigMaps have no encryption, no audit trail, no RBAC differentiation
kind: ConfigMap
data:
  JWT_SECRET: "my-jwt-secret"

# Good
kind: Secret
data:
  JWT_SECRET: <base64>
```

**Mounting all secrets as env vars (`envFrom`)**
```yaml
# Risky: exposes ALL keys in the secret, including future ones
envFrom:
- secretRef:
    name: all-the-secrets

# Better: explicit mapping of only what the pod needs
env:
- name: DATABASE_URL
  valueFrom:
    secretKeyRef:
      name: db-credentials
      key: DATABASE_URL
```

**Not setting `readOnly: true` on volume mounts**
```yaml
# Bad: container can modify mounted config
volumeMounts:
- name: app-config
  mountPath: /app/config

# Good
volumeMounts:
- name: app-config
  mountPath: /app/config
  readOnly: true
```

**Using ConfigMaps for large binary data**
ConfigMaps have a 1 MiB size limit. For large files, use a PersistentVolume or an object store (S3).

## Debugging & Troubleshooting

### Viewing Secret Values

```bash
# View a secret (values are base64 encoded)
kubectl get secret db-credentials -n production -o yaml

# Decode a specific key
kubectl get secret db-credentials -n production \
  -o jsonpath='{.data.DATABASE_URL}' | base64 -d

# Decode all keys
kubectl get secret db-credentials -n production -o json | \
  jq -r '.data | to_entries[] | "\(.key): \(.value | @base64d)"'
```

### Checking if Pod Received Config

```bash
# Check environment variables in a running pod
kubectl exec -n production my-api-pod -- printenv | sort

# Check specific variable
kubectl exec -n production my-api-pod -- printenv DATABASE_URL

# Check mounted files
kubectl exec -n production my-api-pod -- ls /app/config
kubectl exec -n production my-api-pod -- cat /app/config/app.json
```

### ConfigMap Not Updating

```bash
# Verify the ConfigMap was updated
kubectl get configmap api-config -n production -o yaml

# Volume-mounted files update within ~1 minute
# Check the inode timestamp inside the pod
kubectl exec -n production my-api-pod -- ls -la /app/config

# For env vars: restart the pods
kubectl rollout restart deployment/api-server -n production
```

### External Secret Not Syncing

```bash
# Check ExternalSecret status
kubectl describe externalsecret db-credentials -n production

# Check External Secrets Operator logs
kubectl logs -n external-secrets deployment/external-secrets

# Verify IAM permissions (for AWS)
# The IRSA role must have secretsmanager:GetSecretValue permission
# on the specific secret ARN
```

## Real-World Scenarios

### Scenario 1: Complete Config Setup for a Node.js Microservice

```bash
# 1. Create namespace
kubectl create namespace production

# 2. Create ConfigMap from a config file
kubectl create configmap api-config \
  --from-literal=LOG_LEVEL=info \
  --from-literal=NODE_ENV=production \
  --from-literal=PORT=3000 \
  --from-literal=CORS_ORIGINS=https://app.example.com \
  -n production

# 3. Create secrets (or apply from Sealed Secrets in GitOps)
kubectl create secret generic api-secrets \
  --from-literal=DATABASE_URL="$(cat .env.production | grep DATABASE_URL | cut -d= -f2-)" \
  --from-literal=JWT_SECRET="$(openssl rand -hex 32)" \
  --from-literal=REDIS_URL="$(cat .env.production | grep REDIS_URL | cut -d= -f2-)" \
  -n production

# 4. Apply deployment (references the above)
kubectl apply -f k8s/production/deployment.yaml

# 5. Verify
kubectl exec -n production deploy/api-server -- printenv | grep -E "DATABASE_URL|LOG_LEVEL|NODE_ENV"
```

### Scenario 2: Config Rotation Without Downtime

When a secret needs to be rotated (e.g., database password change):

```bash
# 1. Update the Secret with the new value
kubectl create secret generic db-credentials \
  --from-literal=DATABASE_URL="postgresql://app:newpassword@postgres:5432/mydb" \
  -n production \
  --dry-run=client \
  -o yaml | kubectl apply -f -

# 2. Trigger a rolling restart (pods pick up the new env var)
kubectl rollout restart deployment/api-server -n production

# 3. Monitor rollout
kubectl rollout status deployment/api-server -n production

# For volume-mounted secrets: pods pick up changes automatically within ~1 min
# No restart needed if the app watches for file changes
```

## Further Reading

- [ConfigMaps — Kubernetes Docs](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Secrets — Kubernetes Docs](https://kubernetes.io/docs/concepts/configuration/secret/)
- [External Secrets Operator](https://external-secrets.io/)
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
- [Encrypting Secret Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)

## Summary

ConfigMaps store non-sensitive config; Secrets store sensitive data (base64-encoded, encrypted at rest).

Injection methods:
- `env[].valueFrom.configMapKeyRef` / `secretKeyRef` — individual variables
- `envFrom[].configMapRef` / `secretRef` — all keys from a resource
- `volumes` + `volumeMounts` — files (update automatically when ConfigMap changes; env vars don't)

Production patterns:
- **External Secrets Operator** — sync from AWS Secrets Manager, GCP Secret Manager, Vault. Best for cloud-native deployments.
- **Sealed Secrets** — encrypt secrets for git storage. Best for GitOps with kubeseal.
- Always set `readOnly: true` on config volume mounts
- Use explicit `secretKeyRef` (not `envFrom secretRef`) to limit secret surface area
- Never store plaintext secrets in git — use Sealed Secrets or External Secrets

Debugging:
- `kubectl get secret <name> -o jsonpath='{.data.KEY}' | base64 -d` to inspect values
- `kubectl exec <pod> -- printenv` to verify injection
- `kubectl rollout restart deployment/<name>` to pick up updated Secrets in env vars
