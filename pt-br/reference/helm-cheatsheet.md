# Helm Cheatsheet

> Referência rápida — use Ctrl+F para encontrar o que precisa.

---

## Gerenciamento de Repositórios

```bash
# Adicionar repositórios
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add cert-manager https://charts.jetstack.io

# Atualizar e listar
helm repo update                             # atualizar índices de todos os repos
helm repo list                               # mostrar repos configurados
helm repo remove bitnami                     # remover repositório

# Buscar
helm search repo nginx                       # buscar nos repos adicionados
helm search repo nginx --versions            # todas as versões
helm search hub nginx                        # buscar no Artifact Hub (público)
helm show chart bitnami/nginx                # metadados do Chart.yaml
helm show values bitnami/nginx               # values.yaml padrão
helm show readme bitnami/nginx               # README
helm show all bitnami/nginx                  # tudo junto
```

---

## Instalar e Atualizar

```bash
# Instalar
helm install meu-release bitnami/nginx
helm install meu-release bitnami/nginx -n meuns --create-namespace
helm install meu-release ./meuchart            # a partir de diretório local
helm install meu-release mychart-1.0.0.tgz    # a partir de arquivo

# Flags principais
--set chave=valor                              # sobrescrever um valor
--set-string chave=valor                       # forçar tipo string
--values custom.yaml                           # sobrescrever via arquivo (pode repetir)
--namespace meuns                              # namespace de destino
--create-namespace                             # criar namespace se não existir
--version 15.3.1                               # fixar versão do chart
--atomic                                       # rollback em falha + aguardar
--wait                                         # aguardar recursos ficarem prontos
--timeout 5m0s                                 # timeout de espera (padrão: 5m)
--dry-run                                      # simular, renderizar manifests
--dry-run=client                               # somente client-side (sem validação no servidor)
--debug                                        # saída detalhada
--generate-name                                # gerar nome de release automaticamente

# Atualizar
helm upgrade meu-release bitnami/nginx
helm upgrade --install meu-release bitnami/nginx   # instala se não existir (idempotente)
helm upgrade meu-release bitnami/nginx \
  --reuse-values \                              # manter valores existentes
  --set image.tag=1.25

# Forçar upgrade (deletar + recriar recursos que não podem ser aplicados via patch)
helm upgrade meu-release bitnami/nginx --force
```

---

## Gerenciamento de Releases

```bash
helm list                                    # listar releases no namespace atual
helm list -A                                 # todos os namespaces
helm list -n meuns
helm list --failed                           # filtrar por status
helm list --superseded

helm status meu-release                      # status + notas do release
helm history meu-release                     # histórico de revisões
helm history meu-release --max 10

helm rollback meu-release                    # voltar para revisão anterior
helm rollback meu-release 2                  # voltar para revisão 2

helm uninstall meu-release
helm uninstall meu-release --keep-history    # manter histórico para rollback
helm uninstall meu-release -n meuns
```

---

## Desenvolvimento de Charts

```bash
# Scaffold
helm create meuchart                         # gerar estrutura de chart

# Lint
helm lint meuchart/
helm lint meuchart/ --strict                 # falhar em warnings também

# Template (renderizar sem instalar)
helm template meu-release meuchart/
helm template meu-release meuchart/ --debug  # incluir templates com falha
helm template meu-release meuchart/ \
  --values custom.yaml \
  --set image.tag=1.0 \
  > rendered.yaml

# Empacotar e publicar
helm package meuchart/                       # cria meuchart-x.y.z.tgz
helm package meuchart/ --destination ./dist

# Push para registro OCI (Helm 3.8+)
helm registry login registry.exemplo.com
helm push meuchart-1.0.0.tgz oci://registry.exemplo.com/charts
helm install meu-release oci://registry.exemplo.com/charts/meuchart --version 1.0.0

# Chart museum clássico
helm repo index ./dist --url https://charts.exemplo.com
```

---

## Values e Sobrescritas

```
# Precedência (maior para menor):
# 1. Flags --set (o último vence)
# 2. Arquivos --values / -f (o último arquivo vence)
# 3. values.yaml do chart pai
# 4. Padrões do values.yaml do subchart
```

```bash
# Sintaxe do --set
helm install r chart \
  --set nome=valor \                           # string
  --set lista={a,b,c} \                        # array
  --set mapa.chave=valor \                     # chave aninhada
  --set "annotations.kubernetes\.io/nome=v"   # escapar pontos na chave
  --set servers[0].port=80 \                   # índice de array
  --set servers[0].host=exemplo.com

# Múltiplos arquivos de values
helm upgrade r chart -f base.yaml -f prod.yaml -f secrets.yaml
```

```yaml
# values.yaml — boas práticas
# Use camelCase nas chaves
# Agrupe valores relacionados em objetos
# Documente cada chave com comentário
# Forneça padrões sensatos

image:
  repository: nginx
  tag: "1.25"        # use aspas para versões
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

# Padrões vazios para configuração opcional
config: {}
extraEnv: []
extraVolumes: []
```

---

## Hooks

```yaml
# Anotações de hook
metadata:
  annotations:
    "helm.sh/hook": pre-install          # executar antes da instalação
    "helm.sh/hook-weight": "-5"          # ordem de execução (menor = primeiro)
    "helm.sh/hook-delete-policy": hook-succeeded  # política de limpeza

# Eventos de hook disponíveis
# pre-install      post-install
# pre-upgrade      post-upgrade
# pre-rollback     post-rollback
# pre-delete       post-delete
# test             (gatilho do helm test)

# Políticas de deleção
# hook-succeeded          — deletar após sucesso
# hook-failed             — deletar após falha
# before-hook-creation    — deletar antes de nova execução do hook (padrão)
```

```yaml
# Exemplo de job de migração de DB
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "meuchart.fullname" . }}-migracao
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migracao
          image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
          command: ["node", "migrate.js"]
```

---

## Campos Principais do Chart.yaml

```yaml
apiVersion: v2          # v2 para Helm 3 (v1 = Helm 2)
name: meuchart
description: Um Helm chart para minha aplicação
type: application       # application | library
version: 1.2.3          # versão do chart (SemVer) — incrementar ao mudar o chart
appVersion: "2.0.0"     # versão da aplicação (informativo) — exibido no helm list

dependencies:
  - name: postgresql
    version: "~13.0"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled   # ignorar se values.postgresql.enabled=false
    alias: db

maintainers:
  - name: Seu Nome
    email: voce@exemplo.com

keywords:
  - web
  - api
```

```bash
# Após adicionar dependências
helm dependency update meuchart/     # baixar para charts/
helm dependency list meuchart/       # verificar status
```

---

## Padrões Comuns em _helpers.tpl

```yaml
{{/*
Expandir o nome do chart.
*/}}
{{- define "meuchart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Criar nome totalmente qualificado padrão.
*/}}
{{- define "meuchart.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name (include "meuchart.name" .) | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}

{{/*
Labels comuns
*/}}
{{- define "meuchart.labels" -}}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
app.kubernetes.io/name: {{ include "meuchart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Labels de seletor
*/}}
{{- define "meuchart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "meuchart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

---

## Referência Rápida do Helmfile

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
      - web/nginx                   # implantar após nginx
```

```bash
helmfile sync                       # aplicar todos os releases
helmfile apply                      # sync + limpar releases removidos
helmfile diff                       # mostrar diff sem aplicar
helmfile destroy                    # desinstalar todos os releases
helmfile -e staging sync            # selecionar ambiente
helmfile -l app=nginx sync          # filtrar por label
```

---

## Comandos de Debug

```bash
# Inspecionar um release
helm get all meu-release             # tudo: values + manifest + notas + hooks
helm get manifest meu-release        # manifests k8s renderizados
helm get values meu-release          # values fornecidos pelo usuário
helm get values meu-release --all    # todos os values (usuário + padrões)
helm get notes meu-release           # notas pós-instalação
helm get hooks meu-release           # manifests dos hooks

# Debug de renderização de template
helm template meu-release meuchart/ --debug 2>&1 | head -50

# Ver o que vai mudar antes de atualizar
helm diff upgrade meu-release bitnami/nginx  # requer plugin helm-diff
helm plugin install https://github.com/databus23/helm-diff

# Testar um release (executa pods com anotação helm.sh/hook: test)
helm test meu-release
helm test meu-release --logs

# Variáveis de ambiente úteis
export HELM_NAMESPACE=meuns
export HELM_DEBUG=true
```
