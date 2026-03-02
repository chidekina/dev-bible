# Helm Cheatsheet

> Quick reference — use Ctrl+F to find what you need.

---

## Repo Management

```bash
# Add repos
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add cert-manager https://charts.jetstack.io

# Update & list
helm repo update                             # refresh all repo indices
helm repo list                               # show configured repos
helm repo remove bitnami                     # remove repo

# Search
helm search repo nginx                       # search in added repos
helm search repo nginx --versions            # all versions
helm search hub nginx                        # search Artifact Hub (public)
helm show chart bitnami/nginx                # Chart.yaml metadata
helm show values bitnami/nginx               # default values.yaml
helm show readme bitnami/nginx               # README
helm show all bitnami/nginx                  # everything
```

---

## Install & Upgrade

```bash
# Install
helm install my-release bitnami/nginx
helm install my-release bitnami/nginx -n myns --create-namespace
helm install my-release ./mychart            # from local chart dir
helm install my-release mychart-1.0.0.tgz   # from archive

# Key flags
--set key=value                              # override single value
--set-string key=value                       # force string type
--values custom.yaml                         # override from file (can repeat)
--namespace myns                             # target namespace
--create-namespace                           # create ns if missing
--version 15.3.1                             # pin chart version
--atomic                                     # rollback on failure + wait
--wait                                       # wait for resources to be ready
--timeout 5m0s                               # wait timeout (default: 5m)
--dry-run                                    # simulate, render manifests
--dry-run=client                             # client-side only (no server validation)
--debug                                      # verbose output
--generate-name                              # auto-generate release name

# Upgrade
helm upgrade my-release bitnami/nginx
helm upgrade --install my-release bitnami/nginx   # install if not exists (idempotent)
helm upgrade my-release bitnami/nginx \
  --reuse-values \                           # keep existing values
  --set image.tag=1.25

# Force upgrade (delete + recreate resources that can't be patched)
helm upgrade my-release bitnami/nginx --force
```

---

## Release Management

```bash
helm list                                    # list releases in current namespace
helm list -A                                 # all namespaces
helm list -n myns
helm list --failed                           # filter by status
helm list --superseded

helm status my-release                       # release status + notes
helm history my-release                      # revision history
helm history my-release --max 10

helm rollback my-release                     # rollback to previous revision
helm rollback my-release 2                   # rollback to revision 2

helm uninstall my-release
helm uninstall my-release --keep-history     # keep history for rollback
helm uninstall my-release -n myns
```

---

## Chart Development

```bash
# Scaffold
helm create mychart                          # generate chart skeleton

# Lint
helm lint mychart/
helm lint mychart/ --strict                  # fail on warnings too

# Template (render without installing)
helm template my-release mychart/
helm template my-release mychart/ --debug    # include failed templates
helm template my-release mychart/ \
  --values custom.yaml \
  --set image.tag=1.0 \
  > rendered.yaml

# Package & publish
helm package mychart/                        # creates mychart-x.y.z.tgz
helm package mychart/ --destination ./dist

# OCI registry push (Helm 3.8+)
helm registry login registry.example.com
helm push mychart-1.0.0.tgz oci://registry.example.com/charts
helm install my-release oci://registry.example.com/charts/mychart --version 1.0.0

# Classic chart museum
helm repo index ./dist --url https://charts.example.com
```

---

## Values & Overrides

```
# Precedence (highest to lowest):
# 1. --set flags (last one wins)
# 2. --values / -f files (last file wins)
# 3. Parent chart values.yaml
# 4. Subchart values.yaml defaults
```

```bash
# --set syntax
helm install r chart \
  --set name=value \                         # string
  --set list={a,b,c} \                       # array
  --set map.key=value \                      # nested key
  --set "annotations.kubernetes\.io/name=v"  # escape dots in key
  --set servers[0].port=80 \                 # array index
  --set servers[0].host=example.com

# Multiple value files
helm upgrade r chart -f base.yaml -f prod.yaml -f secrets.yaml
```

```yaml
# values.yaml — best practices
# Use camelCase for keys
# Group related values under objects
# Document each key with a comment
# Provide sensible defaults

image:
  repository: nginx
  tag: "1.25"        # use string quotes for versions
  pullPolicy: IfNotPresent

replicaCount: 2

service:
  type: ClusterIP
  port: 80

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi

# Empty defaults for optional config
config: {}
extraEnv: []
extraVolumes: []
```

---

## Hooks

```yaml
# Hook annotations
metadata:
  annotations:
    "helm.sh/hook": pre-install          # run before install
    "helm.sh/hook-weight": "-5"          # execution order (lower = first)
    "helm.sh/hook-delete-policy": hook-succeeded  # cleanup policy

# Available hook events
# pre-install      post-install
# pre-upgrade      post-upgrade
# pre-rollback     post-rollback
# pre-delete       post-delete
# test             (helm test trigger)

# Delete policies
# hook-succeeded   — delete after success
# hook-failed      — delete after failure
# before-hook-creation — delete before new hook run (default)
```

```yaml
# DB migration job example
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "mychart.fullname" . }}-migration
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migration
          image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
          command: ["node", "migrate.js"]
```

---

## Chart.yaml Key Fields

```yaml
apiVersion: v2          # v2 for Helm 3 (v1 = Helm 2)
name: mychart
description: A Helm chart for my application
type: application       # application | library
version: 1.2.3          # chart version (SemVer) — bump on chart changes
appVersion: "2.0.0"     # app version (informational) — displayed in helm list

dependencies:
  - name: postgresql
    version: "~13.0"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled   # skip if values.postgresql.enabled=false
    alias: db

maintainers:
  - name: Your Name
    email: you@example.com

keywords:
  - web
  - api
```

```bash
# After adding dependencies
helm dependency update mychart/     # download to charts/
helm dependency list mychart/       # check status
```

---

## _helpers.tpl Common Patterns

```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "mychart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "mychart.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name (include "mychart.name" .) | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "mychart.labels" -}}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "mychart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

---

## Helmfile Quick Reference

```yaml
# helmfile.yaml
repositories:
  - name: bitnami
    url: https://charts.bitnami.com/bitnami

environments:
  staging:
    values:
      - environments/staging.yaml
  production:
    values:
      - environments/production.yaml

releases:
  - name: nginx
    namespace: web
    chart: bitnami/nginx
    version: ~15.0
    values:
      - values/nginx.yaml
    set:
      - name: replicaCount
        value: {{ .Values.nginx.replicas }}

  - name: postgres
    namespace: data
    chart: bitnami/postgresql
    version: ~13.0
    needs:
      - web/nginx                   # deploy after nginx
```

```bash
helmfile sync                       # apply all releases
helmfile apply                      # sync + cleanup removed releases
helmfile diff                       # show diff without applying
helmfile destroy                    # uninstall all releases
helmfile -e staging sync            # target environment
helmfile -l app=nginx sync          # filter by label
```

---

## Debugging Commands

```bash
# Inspect a release
helm get all my-release             # everything: values + manifest + notes + hooks
helm get manifest my-release        # rendered k8s manifests
helm get values my-release          # applied user-supplied values
helm get values my-release --all    # all values (user + defaults)
helm get notes my-release           # post-install notes
helm get hooks my-release           # hook manifests

# Template rendering debug
helm template my-release mychart/ --debug 2>&1 | head -50

# Check what will change before upgrade
helm diff upgrade my-release bitnami/nginx  # requires helm-diff plugin
helm plugin install https://github.com/databus23/helm-diff

# Test a release (runs pods with helm.sh/hook: test annotation)
helm test my-release
helm test my-release --logs

# Useful env
export HELM_NAMESPACE=myns
export HELM_DEBUG=true
```
