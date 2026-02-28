# Helm Basics

## Overview

Helm is the package manager for Kubernetes. Instead of managing dozens of individual YAML files for each application, Helm packages them into a **chart** — a versioned, parameterized, reusable bundle. You install a chart with custom `values.yaml` settings, and Helm renders the templates and applies them to your cluster.

Helm solves three problems that raw kubectl struggles with:
1. **Parameterization** — same templates, different values per environment
2. **Release management** — install, upgrade, rollback as atomic operations
3. **Dependency management** — declare and install chart dependencies

## Prerequisites

- kubectl installed and configured
- Kubernetes cluster access
- Understanding of Pods, Deployments, Services, ConfigMaps

## Core Concepts

### Chart Structure

A Helm chart is a directory with a specific layout:

```
my-app/
├── Chart.yaml           # Chart metadata (name, version, description)
├── values.yaml          # Default configuration values
├── values-staging.yaml  # (optional) Environment-specific overrides
├── charts/              # (optional) Dependency charts (subcharts)
├── templates/
│   ├── _helpers.tpl     # Template helpers (named templates, reusable snippets)
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── hpa.yaml
│   ├── serviceaccount.yaml
│   └── NOTES.txt        # Post-install notes displayed to user
└── .helmignore          # Files to exclude from packaging
```

### Chart.yaml

```yaml
apiVersion: v2           # Helm 3 (use v1 only for Helm 2 compatibility)
name: my-app
description: A production-ready Node.js API
type: application        # or 'library' for reusable chart templates
version: 1.4.2           # Chart version (semantic versioning)
appVersion: "2.4.1"      # Application version (informational)
maintainers:
- name: Platform Team
  email: platform@example.com
dependencies:
- name: postgresql
  version: "14.x.x"
  repository: https://charts.bitnami.com/bitnami
  condition: postgresql.enabled
```

### values.yaml

Default values that users override at install time:

```yaml
# values.yaml
replicaCount: 3

image:
  repository: my-org/api-server
  tag: ""                   # if empty, uses Chart.yaml appVersion
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 3000

ingress:
  enabled: false
  className: nginx
  host: ""
  tls: true
  annotations: {}

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

env:
  LOG_LEVEL: info
  NODE_ENV: production

secrets:
  databaseUrl: ""         # must be set at install time
  jwtSecret: ""

postgresql:
  enabled: false          # disable by default (external DB assumed)
```

### Template Syntax

Helm uses Go templates with the Sprig function library.

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
  annotations:
    helm.sh/chart: {{ include "my-app.chart" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
      annotations:
        # Trigger rollout on configmap change
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: {{ .Values.service.targetPort }}
        env:
        {{- range $key, $value := .Values.env }}
        - name: {{ $key }}
          value: {{ $value | quote }}
        {{- end }}
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: {{ include "my-app.fullname" . }}-secrets
              key: databaseUrl
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
```

### _helpers.tpl

Named templates (reusable snippets) that other templates call:

```
{{/* templates/_helpers.tpl */}}

{{/*
Expand the name of the chart.
*/}}
{{- define "my-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "my-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "my-app.labels" -}}
helm.sh/chart: {{ include "my-app.chart" . }}
{{ include "my-app.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "my-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "my-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### Built-in Objects

Available in every template:

| Object | Description |
|--------|-------------|
| `.Release.Name` | Release name (e.g., `my-app-production`) |
| `.Release.Namespace` | Target namespace |
| `.Release.IsInstall` | `true` if first install |
| `.Release.IsUpgrade` | `true` if upgrade |
| `.Chart.Name` | Chart name |
| `.Chart.Version` | Chart version |
| `.Chart.AppVersion` | App version from Chart.yaml |
| `.Values` | Contents of values.yaml (merged with overrides) |
| `.Files` | Access to non-template files in the chart |

## Hands-On Examples

### Installing and Managing Releases

```bash
# Add a chart repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Search for charts
helm search repo postgres
helm search hub nginx-ingress

# Install a chart
helm install my-release bitnami/postgresql \
  --namespace database \
  --create-namespace \
  --set auth.password=mypassword \
  --set primary.persistence.size=20Gi

# Install from a local chart
helm install my-app ./my-app-chart \
  --namespace production \
  --create-namespace \
  -f values.yaml \
  -f values-production.yaml

# Install with set overrides (highest priority)
helm install my-app ./my-app \
  --namespace production \
  --set image.tag=2.4.1 \
  --set replicaCount=5 \
  --set "ingress.host=api.example.com"

# Upgrade an existing release
helm upgrade my-app ./my-app \
  --namespace production \
  -f values-production.yaml \
  --set image.tag=2.5.0

# Install or upgrade (idempotent — use in CI/CD)
helm upgrade --install my-app ./my-app \
  --namespace production \
  --create-namespace \
  -f values-production.yaml \
  --set image.tag=${IMAGE_TAG}

# List releases
helm list -n production
helm list --all-namespaces

# Get release status
helm status my-app -n production

# Show history
helm history my-app -n production

# Rollback to previous version
helm rollback my-app -n production

# Rollback to specific revision
helm rollback my-app 3 -n production

# Uninstall a release (removes all resources)
helm uninstall my-app -n production

# Uninstall but keep history
helm uninstall my-app -n production --keep-history
```

### Debugging Templates

```bash
# Dry-run: render templates but don't apply
helm install my-app ./my-app \
  --dry-run \
  --debug \
  -f values-production.yaml

# Render templates to stdout (no cluster needed)
helm template my-app ./my-app \
  -f values-production.yaml \
  --set image.tag=2.4.1

# Render only specific templates
helm template my-app ./my-app \
  -s templates/deployment.yaml

# Show computed values (merged from all sources)
helm get values my-app -n production

# Show all values including defaults
helm get values my-app -n production --all

# Show rendered manifests for a deployed release
helm get manifest my-app -n production

# Lint the chart
helm lint ./my-app
helm lint ./my-app -f values-production.yaml
```

### Conditional Templates

```yaml
# templates/ingress.yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "my-app.fullname" . }}
  annotations:
    {{- if .Values.ingress.tls }}
    cert-manager.io/cluster-issuer: letsencrypt-prod
    {{- end }}
    {{- with .Values.ingress.annotations }}
    {{- toYaml . | nindent 4 }}
    {{- end }}
spec:
  ingressClassName: {{ .Values.ingress.className }}
  {{- if .Values.ingress.tls }}
  tls:
  - hosts:
    - {{ .Values.ingress.host }}
    secretName: {{ include "my-app.fullname" . }}-tls
  {{- end }}
  rules:
  - host: {{ .Values.ingress.host | quote }}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: {{ include "my-app.fullname" . }}
            port:
              number: {{ .Values.service.port }}
{{- end }}
```

### Conditional HPA

```yaml
# templates/hpa.yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "my-app.fullname" . }}
  namespace: {{ .Release.Namespace }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "my-app.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
  {{- if .Values.autoscaling.targetMemoryUtilizationPercentage }}
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: {{ .Values.autoscaling.targetMemoryUtilizationPercentage }}
  {{- end }}
{{- end }}
```

### Helmfile for Multi-Release Management

Helmfile manages multiple Helm releases as code:

```bash
brew install helmfile
```

```yaml
# helmfile.yaml
repositories:
- name: ingress-nginx
  url: https://kubernetes.github.io/ingress-nginx
- name: jetstack
  url: https://charts.jetstack.io
- name: bitnami
  url: https://charts.bitnami.com/bitnami

releases:
- name: ingress-nginx
  namespace: ingress-nginx
  chart: ingress-nginx/ingress-nginx
  version: 4.8.3
  values:
  - controller:
      replicaCount: 2
      metrics:
        enabled: true

- name: cert-manager
  namespace: cert-manager
  chart: jetstack/cert-manager
  version: 1.13.3
  set:
  - name: installCRDs
    value: true

- name: api-server
  namespace: production
  chart: ./charts/my-app
  values:
  - values.yaml
  - values-production.yaml
  set:
  - name: image.tag
    value: {{ requiredEnv "IMAGE_TAG" }}

environments:
  staging:
    values:
    - env: staging
  production:
    values:
    - env: production
```

```bash
# Apply all releases
helmfile apply

# Diff before applying
helmfile diff

# Apply a specific release
helmfile apply --selector name=api-server

# Sync (like apply but always shows diff)
helmfile sync
```

## Common Patterns & Best Practices

### Versioning Strategy

```yaml
# Chart.yaml
version: 1.4.2      # bump this on EVERY chart change
appVersion: "2.4.1" # update this to match the application release
```

Use semantic versioning for charts:
- Patch (`1.4.2` → `1.4.3`) — bug fixes, minor template changes
- Minor (`1.4.2` → `1.5.0`) — new optional features, backward-compatible
- Major (`1.4.2` → `2.0.0`) — breaking changes to values.yaml structure

### Values File Hierarchy

```bash
# Layered values — later files override earlier ones
helm install my-app ./my-app \
  -f values.yaml              # defaults
  -f values-production.yaml   # environment overrides
  --set image.tag=${CI_TAG}   # CI overrides (highest priority)
```

### Testing Charts

```bash
# Add chart-testing
helm plugin install https://github.com/helm-unittest/helm-unittest

# Write unit tests for templates
# tests/deployment_test.yaml
suite: deployment tests
templates:
  - templates/deployment.yaml
tests:
  - it: should have 3 replicas by default
    asserts:
    - equal:
        path: spec.replicas
        value: 3
  - it: should use custom replica count
    set:
      replicaCount: 5
    asserts:
    - equal:
        path: spec.replicas
        value: 5

# Run tests
helm unittest ./my-app
```

```bash
# Integration test: install to a kind cluster
helm install my-app ./my-app \
  --namespace testing \
  --create-namespace \
  --wait \
  --timeout 5m

# Run helm test (tests defined in templates/tests/)
helm test my-app -n testing
```

### NOTES.txt

Always include post-install guidance:

```
# templates/NOTES.txt
1. Get the application URL by running:

{{- if .Values.ingress.enabled }}
  https://{{ .Values.ingress.host }}
{{- else }}
  kubectl port-forward -n {{ .Release.Namespace }} \
    service/{{ include "my-app.fullname" . }} 8080:{{ .Values.service.port }}
  Then visit: http://localhost:8080
{{- end }}

2. Check the deployment status:
  kubectl rollout status deployment/{{ include "my-app.fullname" . }} -n {{ .Release.Namespace }}

3. View logs:
  kubectl logs -n {{ .Release.Namespace }} \
    -l "app.kubernetes.io/instance={{ .Release.Name }}" \
    --tail=50 -f
```

## Anti-Patterns to Avoid

**Hardcoding values in templates**
```yaml
# Bad: can't be overridden
image: my-org/api:2.4.1

# Good: parameterized
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

**Not quoting string values that might be interpreted as numbers**
```yaml
# Bad: YAML may parse "1.10" as 1.1, "true" as boolean
version: {{ .Values.appVersion }}

# Good
version: {{ .Values.appVersion | quote }}
```

**Giant values.yaml with no documentation**
```yaml
# Bad: no comments, unclear purpose
x: 100
y: true
z: ""

# Good: document every value
# Maximum number of pods to schedule
replicaCount: 3

# Whether to enable Horizontal Pod Autoscaler
autoscaling:
  # Set to true to enable HPA (disables replicaCount)
  enabled: false
```

**Not using `helm upgrade --install`** in CI/CD — use idempotent operations.

**Not pinning chart versions** — always pin `version` in `helmfile.yaml` or `Chart.yaml` dependencies.

## Debugging & Troubleshooting

### Template Rendering Errors

```bash
# See the full error with context
helm template my-app ./my-app -f values.yaml 2>&1 | head -50

# Lint catches many common issues
helm lint ./my-app

# Validate against cluster (requires cluster access)
helm lint ./my-app --with-subcharts

# Check specific template
helm template my-app ./my-app \
  -s templates/deployment.yaml \
  -f values.yaml
```

### Release in Bad State

```bash
# See release history
helm history my-app -n production

# If stuck in pending-upgrade state
helm rollback my-app -n production

# Force delete a broken release (removes all resources)
helm uninstall my-app -n production

# If uninstall hangs, force delete with kubectl
kubectl delete all -l "app.kubernetes.io/instance=my-app" -n production
```

## Real-World Scenarios

### Scenario 1: CI/CD Pipeline with Helm

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: azure/setup-helm@v3

    - name: Build and push image
      run: |
        docker build -t my-org/api:${{ github.sha }} .
        docker push my-org/api:${{ github.sha }}

    - name: Deploy to staging
      run: |
        helm upgrade --install api-server ./charts/my-app \
          --namespace staging \
          --create-namespace \
          -f charts/my-app/values.yaml \
          -f charts/my-app/values-staging.yaml \
          --set image.tag=${{ github.sha }} \
          --wait \
          --timeout 5m

    - name: Run smoke tests
      run: |
        kubectl wait --for=condition=ready pod \
          -l app.kubernetes.io/instance=api-server \
          -n staging --timeout=120s
        kubectl run smoke-test --image=curlimages/curl --rm -it --restart=Never \
          -- curl -f http://api-server.staging.svc.cluster.local/health

    - name: Deploy to production
      run: |
        helm upgrade --install api-server ./charts/my-app \
          --namespace production \
          --create-namespace \
          -f charts/my-app/values.yaml \
          -f charts/my-app/values-production.yaml \
          --set image.tag=${{ github.sha }} \
          --wait \
          --timeout 10m
```

## Further Reading

- [Helm Documentation](https://helm.sh/docs/)
- [Helm Best Practices](https://helm.sh/docs/chart_best_practices/)
- [Helmfile](https://helmfile.readthedocs.io/)
- [Helm Unittest](https://github.com/helm-unittest/helm-unittest)
- [Artifact Hub](https://artifacthub.io/) — search for public Helm charts

## Summary

Helm is the package manager for Kubernetes — it packages manifests into versioned, parameterized charts.

Chart structure:
- `Chart.yaml` — metadata, version, appVersion, dependencies
- `values.yaml` — default configuration values
- `templates/` — Go template YAML files
- `templates/_helpers.tpl` — reusable named templates

Essential commands:
- `helm upgrade --install <name> <chart>` — idempotent install/upgrade (use in CI/CD)
- `helm template <name> <chart>` — render templates without applying (dry-run)
- `helm diff` — see what would change
- `helm rollback <name>` — revert to previous release
- `helm history <name>` — see release history

Best practices:
- Layer values: `values.yaml` (defaults) → `values-<env>.yaml` (environment) → `--set` (CI overrides)
- Pin chart and dependency versions
- Use `helm lint` before committing chart changes
- Include a meaningful `NOTES.txt` with post-install instructions
- Use helmfile to manage many releases declaratively
