# Helm Básico

## Visão Geral

Helm é o gerenciador de pacotes para Kubernetes. Em vez de gerenciar dezenas de arquivos YAML individuais para cada aplicação, o Helm os empacota em um **chart** — um bundle versionado, parametrizado e reutilizável. Você instala um chart com configurações customizadas em `values.yaml`, e o Helm renderiza os templates e os aplica ao seu cluster.

O Helm resolve três problemas com os quais o kubectl puro luta:
1. **Parametrização** — mesmos templates, valores diferentes por ambiente
2. **Gerenciamento de releases** — instalar, atualizar, fazer rollback como operações atômicas
3. **Gerenciamento de dependências** — declarar e instalar dependências de chart

## Pré-requisitos

- kubectl instalado e configurado
- Acesso a um cluster Kubernetes
- Compreensão de Pods, Deployments, Services, ConfigMaps

## Conceitos Principais

### Estrutura de um Chart

Um Helm chart é um diretório com um layout específico:

```
my-app/
├── Chart.yaml           # Metadados do chart (nome, versão, descrição)
├── values.yaml          # Valores de configuração padrão
├── values-staging.yaml  # (opcional) Sobrescritas específicas por ambiente
├── charts/              # (opcional) Charts de dependência (subcharts)
├── templates/
│   ├── _helpers.tpl     # Helpers de template (templates nomeados, snippets reutilizáveis)
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── hpa.yaml
│   ├── serviceaccount.yaml
│   └── NOTES.txt        # Notas pós-instalação exibidas ao usuário
└── .helmignore          # Arquivos a excluir do empacotamento
```

### Chart.yaml

```yaml
apiVersion: v2           # Helm 3 (use v1 apenas para compatibilidade com Helm 2)
name: my-app
description: Uma API Node.js pronta para produção
type: application        # ou 'library' para templates de chart reutilizáveis
version: 1.4.2           # versão do chart (versionamento semântico)
appVersion: "2.4.1"      # versão da aplicação (informativa)
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

Valores padrão que os usuários sobrescrevem no momento da instalação:

```yaml
# values.yaml
replicaCount: 3

image:
  repository: my-org/api-server
  tag: ""                   # se vazio, usa o appVersion do Chart.yaml
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
  databaseUrl: ""         # deve ser definido no momento da instalação
  jwtSecret: ""

postgresql:
  enabled: false          # desabilitado por padrão (banco de dados externo assumido)
```

### Sintaxe de Template

O Helm usa templates Go com a biblioteca de funções Sprig.

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
        # Acionar rollout na mudança do configmap
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

Templates nomeados (snippets reutilizáveis) que outros templates chamam:

```
{{/* templates/_helpers.tpl */}}

{{/*
Expande o nome do chart.
*/}}
{{- define "my-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Cria um nome completo qualificado padrão para a aplicação.
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
Labels comuns
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
Labels de seletor
*/}}
{{- define "my-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "my-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### Objetos Embutidos

Disponíveis em todos os templates:

| Objeto | Descrição |
|--------|-----------|
| `.Release.Name` | Nome da release (ex.: `my-app-production`) |
| `.Release.Namespace` | Namespace de destino |
| `.Release.IsInstall` | `true` se for a primeira instalação |
| `.Release.IsUpgrade` | `true` se for um upgrade |
| `.Chart.Name` | Nome do chart |
| `.Chart.Version` | Versão do chart |
| `.Chart.AppVersion` | Versão da aplicação definida no Chart.yaml |
| `.Values` | Conteúdo do values.yaml (mesclado com sobrescritas) |
| `.Files` | Acesso a arquivos não-template no chart |

## Exemplos Práticos

### Instalando e Gerenciando Releases

```bash
# Adicionar um repositório de charts
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Buscar charts
helm search repo postgres
helm search hub nginx-ingress

# Instalar um chart
helm install my-release bitnami/postgresql \
  --namespace database \
  --create-namespace \
  --set auth.password=mypassword \
  --set primary.persistence.size=20Gi

# Instalar a partir de um chart local
helm install my-app ./my-app-chart \
  --namespace production \
  --create-namespace \
  -f values.yaml \
  -f values-production.yaml

# Instalar com sobrescritas via --set (maior prioridade)
helm install my-app ./my-app \
  --namespace production \
  --set image.tag=2.4.1 \
  --set replicaCount=5 \
  --set "ingress.host=api.example.com"

# Atualizar uma release existente
helm upgrade my-app ./my-app \
  --namespace production \
  -f values-production.yaml \
  --set image.tag=2.5.0

# Instalar ou atualizar (idempotente — use em CI/CD)
helm upgrade --install my-app ./my-app \
  --namespace production \
  --create-namespace \
  -f values-production.yaml \
  --set image.tag=${IMAGE_TAG}

# Listar releases
helm list -n production
helm list --all-namespaces

# Obter status da release
helm status my-app -n production

# Ver histórico
helm history my-app -n production

# Fazer rollback para a versão anterior
helm rollback my-app -n production

# Fazer rollback para uma revisão específica
helm rollback my-app 3 -n production

# Desinstalar uma release (remove todos os recursos)
helm uninstall my-app -n production

# Desinstalar mas manter o histórico
helm uninstall my-app -n production --keep-history
```

### Depurando Templates

```bash
# Dry-run: renderizar templates sem aplicar
helm install my-app ./my-app \
  --dry-run \
  --debug \
  -f values-production.yaml

# Renderizar templates para stdout (sem necessidade de cluster)
helm template my-app ./my-app \
  -f values-production.yaml \
  --set image.tag=2.4.1

# Renderizar apenas templates específicos
helm template my-app ./my-app \
  -s templates/deployment.yaml

# Mostrar os valores computados (mesclados de todas as fontes)
helm get values my-app -n production

# Mostrar todos os valores incluindo os padrões
helm get values my-app -n production --all

# Mostrar os manifests renderizados de uma release implantada
helm get manifest my-app -n production

# Fazer lint do chart
helm lint ./my-app
helm lint ./my-app -f values-production.yaml
```

### Templates Condicionais

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

### HPA Condicional

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

### Helmfile para Gerenciamento de Múltiplas Releases

O Helmfile gerencia múltiplas releases do Helm como código:

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
# Aplicar todas as releases
helmfile apply

# Ver diff antes de aplicar
helmfile diff

# Aplicar uma release específica
helmfile apply --selector name=api-server

# Sync (como apply, mas sempre mostra o diff)
helmfile sync
```

## Padrões Comuns e Boas Práticas

### Estratégia de Versionamento

```yaml
# Chart.yaml
version: 1.4.2      # incremente a CADA mudança no chart
appVersion: "2.4.1" # atualize para corresponder ao release da aplicação
```

Use versionamento semântico para charts:
- Patch (`1.4.2` → `1.4.3`) — correções de bug, pequenas mudanças de template
- Minor (`1.4.2` → `1.5.0`) — novos recursos opcionais, retrocompatível
- Major (`1.4.2` → `2.0.0`) — mudanças que quebram a estrutura do values.yaml

### Hierarquia de Arquivos de Values

```bash
# Values em camadas — arquivos posteriores sobrescrevem os anteriores
helm install my-app ./my-app \
  -f values.yaml              # padrões
  -f values-production.yaml   # sobrescritas de ambiente
  --set image.tag=${CI_TAG}   # sobrescritas do CI (maior prioridade)
```

### Testando Charts

```bash
# Adicionar chart-testing
helm plugin install https://github.com/helm-unittest/helm-unittest

# Escrever testes unitários para templates
# tests/deployment_test.yaml
suite: testes do deployment
templates:
  - templates/deployment.yaml
tests:
  - it: deve ter 3 réplicas por padrão
    asserts:
    - equal:
        path: spec.replicas
        value: 3
  - it: deve usar contagem de réplicas customizada
    set:
      replicaCount: 5
    asserts:
    - equal:
        path: spec.replicas
        value: 5

# Rodar os testes
helm unittest ./my-app
```

```bash
# Teste de integração: instalar em um cluster kind
helm install my-app ./my-app \
  --namespace testing \
  --create-namespace \
  --wait \
  --timeout 5m

# Rodar helm test (testes definidos em templates/tests/)
helm test my-app -n testing
```

### NOTES.txt

Sempre inclua orientações pós-instalação:

```
# templates/NOTES.txt
1. Obtenha a URL da aplicação executando:

{{- if .Values.ingress.enabled }}
  https://{{ .Values.ingress.host }}
{{- else }}
  kubectl port-forward -n {{ .Release.Namespace }} \
    service/{{ include "my-app.fullname" . }} 8080:{{ .Values.service.port }}
  Então acesse: http://localhost:8080
{{- end }}

2. Verifique o status do deployment:
  kubectl rollout status deployment/{{ include "my-app.fullname" . }} -n {{ .Release.Namespace }}

3. Ver logs:
  kubectl logs -n {{ .Release.Namespace }} \
    -l "app.kubernetes.io/instance={{ .Release.Name }}" \
    --tail=50 -f
```

## Anti-Padrões a Evitar

**Fixar valores nos templates**
```yaml
# Ruim: não pode ser sobrescrito
image: my-org/api:2.4.1

# Bom: parametrizado
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

**Não colocar entre aspas valores string que possam ser interpretados como números**
```yaml
# Ruim: o YAML pode interpretar "1.10" como 1.1, "true" como booleano
version: {{ .Values.appVersion }}

# Bom
version: {{ .Values.appVersion | quote }}
```

**values.yaml gigante sem documentação**
```yaml
# Ruim: sem comentários, finalidade obscura
x: 100
y: true
z: ""

# Bom: documente cada valor
# Número máximo de pods a agendar
replicaCount: 3

# Se deve habilitar o Horizontal Pod Autoscaler
autoscaling:
  # Defina como true para habilitar o HPA (desabilita replicaCount)
  enabled: false
```

**Não usar `helm upgrade --install`** em CI/CD — use operações idempotentes.

**Não fixar versões de chart** — sempre fixe `version` no `helmfile.yaml` ou nas dependências do `Chart.yaml`.

## Depuração e Troubleshooting

### Erros de Renderização de Template

```bash
# Ver o erro completo com contexto
helm template my-app ./my-app -f values.yaml 2>&1 | head -50

# O lint captura muitos problemas comuns
helm lint ./my-app

# Validar contra o cluster (requer acesso ao cluster)
helm lint ./my-app --with-subcharts

# Verificar template específico
helm template my-app ./my-app \
  -s templates/deployment.yaml \
  -f values.yaml
```

### Release em Estado Ruim

```bash
# Ver histórico da release
helm history my-app -n production

# Se estiver travada em estado pending-upgrade
helm rollback my-app -n production

# Forçar deleção de uma release quebrada (remove todos os recursos)
helm uninstall my-app -n production

# Se a desinstalação travar, forçar deleção com kubectl
kubectl delete all -l "app.kubernetes.io/instance=my-app" -n production
```

## Cenários do Mundo Real

### Cenário 1: Pipeline CI/CD com Helm

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

## Leitura Adicional

- [Helm Documentation](https://helm.sh/docs/)
- [Helm Best Practices](https://helm.sh/docs/chart_best_practices/)
- [Helmfile](https://helmfile.readthedocs.io/)
- [Helm Unittest](https://github.com/helm-unittest/helm-unittest)
- [Artifact Hub](https://artifacthub.io/) — buscar Helm charts públicos

## Resumo

Helm é o gerenciador de pacotes para Kubernetes — ele empacota manifests em charts versionados e parametrizados.

Estrutura do chart:
- `Chart.yaml` — metadados, versão, appVersion, dependências
- `values.yaml` — valores de configuração padrão
- `templates/` — arquivos YAML com templates Go
- `templates/_helpers.tpl` — templates nomeados reutilizáveis

Comandos essenciais:
- `helm upgrade --install <nome> <chart>` — instalar/atualizar de forma idempotente (use em CI/CD)
- `helm template <nome> <chart>` — renderizar templates sem aplicar (dry-run)
- `helm diff` — ver o que vai mudar
- `helm rollback <nome>` — reverter para a release anterior
- `helm history <nome>` — ver o histórico de releases

Boas práticas:
- Camadas de values: `values.yaml` (padrões) → `values-<env>.yaml` (ambiente) → `--set` (sobrescritas do CI)
- Fixe versões de chart e de dependências
- Use `helm lint` antes de fazer commit de mudanças no chart
- Inclua um `NOTES.txt` significativo com instruções pós-instalação
- Use helmfile para gerenciar muitas releases de forma declarativa
