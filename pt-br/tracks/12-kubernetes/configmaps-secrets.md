# ConfigMaps & Secrets

## Visão Geral

Aplicações precisam de configuração: URLs de banco de dados, feature flags, níveis de log, endpoints de API, certificados TLS e credenciais. O Kubernetes fornece dois objetos para essa finalidade:

- **ConfigMap** — configuração não sensível (strings, arquivos de configuração, dados JSON)
- **Secret** — dados sensíveis (senhas, tokens, chaves privadas) — armazenados codificados em base64 e criptografados em repouso com a configuração adequada do cluster

Ambos podem ser injetados em Pods como variáveis de ambiente ou montados como arquivos. Este capítulo cobre criação, padrões de injeção, o external-secrets-operator para sincronização a partir do AWS Secrets Manager / GCP Secret Manager / HashiCorp Vault, e Sealed Secrets para armazenar valores criptografados no git.

## Pré-requisitos

- Conceitos básicos de Pods e Deployments
- kubectl básico
- Compreensão básica de YAML e codificação base64

## Conceitos Principais

### ConfigMap

Um ConfigMap armazena pares chave-valor. As chaves são strings; os valores podem ser qualquer string, incluindo multi-linha (arquivos de configuração, JSON).

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
  namespace: production
data:
  # Pares chave-valor simples
  LOG_LEVEL: "info"
  PORT: "3000"
  FEATURE_NEW_DASHBOARD: "true"

  # Entradas do tipo arquivo (valores multi-linha)
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

Secrets armazenam dados sensíveis. Os valores devem ser codificados em base64 no YAML (o Kubernetes os decodifica antes da injeção).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
type: Opaque       # genérico — tipo mais comum
data:
  # codificado em base64: echo -n "postgresql://..." | base64
  DATABASE_URL: cG9zdGdyZXNxbDovL2FwcDpzZWNyZXRAcG9zdGdyZXM6NTQzMi9teWRi
  API_KEY: c2VjcmV0LWFwaS1rZXktMTIz
stringData:
  # valores em stringData NÃO estão codificados em base64 — o Kubernetes os codifica
  # Use isso para legibilidade em manifests criados manualmente
  REDIS_PASSWORD: "my-redis-password"
```

Tipos de Secret:
| Tipo | Finalidade |
|------|------------|
| `Opaque` | Dados chave-valor arbitrários (padrão) |
| `kubernetes.io/tls` | Certificado TLS e chave (`tls.crt`, `tls.key`) |
| `kubernetes.io/dockerconfigjson` | Credenciais de registry Docker |
| `kubernetes.io/service-account-token` | Token de ServiceAccount (criado automaticamente) |
| `kubernetes.io/ssh-auth` | Chave privada SSH |

### Métodos de Injeção

**Método 1: Variáveis de ambiente individuais**

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

**Método 2: Todas as chaves de um ConfigMap/Secret como variáveis de ambiente**

```yaml
envFrom:
- configMapRef:
    name: api-config
- secretRef:
    name: db-credentials
# Todas as chaves se tornam variáveis de ambiente com o mesmo nome
```

**Método 3: Volume mount (arquivos)**

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
    items:                    # opcional: selecione chaves específicas
    - key: app.json
      path: app.json          # nome do arquivo no diretório montado
- name: tls-certs
  secret:
    secretName: api-tls-cert
    defaultMode: 0400         # permissões restritivas para arquivos de chave
```

**Volume mounts se atualizam automaticamente** — quando um ConfigMap é atualizado, os arquivos no volume são atualizados (com atraso de ~1 minuto). Variáveis de ambiente NÃO se atualizam — elas exigem reinicialização do pod.

## Exemplos Práticos

### Criando Recursos com kubectl

```bash
# ConfigMap a partir de valores literais
kubectl create configmap api-config \
  --from-literal=LOG_LEVEL=info \
  --from-literal=PORT=3000 \
  --namespace=production

# ConfigMap a partir de um arquivo
kubectl create configmap nginx-config \
  --from-file=nginx.conf \
  --namespace=production

# ConfigMap a partir de um diretório (cada arquivo vira uma chave)
kubectl create configmap app-configs \
  --from-file=./config/ \
  --namespace=production

# Secret a partir de literais
kubectl create secret generic db-credentials \
  --from-literal=DATABASE_URL='postgresql://app:secret@postgres:5432/mydb' \
  --from-literal=REDIS_PASSWORD='redis-secret' \
  --namespace=production

# Secret a partir de um arquivo (ex.: chave privada)
kubectl create secret generic ssh-key \
  --from-file=id_rsa=~/.ssh/id_rsa \
  --namespace=production

# Secret TLS a partir de arquivos de certificado e chave
kubectl create secret tls api-tls-cert \
  --cert=server.crt \
  --key=server.key \
  --namespace=production

# Credenciais de registry Docker
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password='mypassword' \
  --namespace=production
```

### Pod Completo com Injeção de ConfigMap e Secret

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
        # Carregar toda a configuração não sensível como variáveis de ambiente
        - configMapRef:
            name: api-config
        env:
        # Carregar secrets individuais
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
        # Montar um arquivo de configuração
        - name: app-json
          mountPath: /app/config
          readOnly: true
        # Montar certificado TLS para HTTPS
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

### Lendo a Configuração Montada em Node.js

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

// Ler arquivo de configuração montado
function readAppConfig() {
  try {
    const raw = readFileSync('/app/config/app.json', 'utf-8');
    return JSON.parse(raw);
  } catch {
    return {}; // fallback para os padrões
  }
}

export const appConfig = readAppConfig();
```

### Observando Atualizações de Arquivo de Configuração

Quando ConfigMaps são montados como volumes, o Kubernetes atualiza os arquivos no lugar (~1 min). Implemente um watcher para recarregar sem reiniciar:

```typescript
import { watch } from 'fs';

const CONFIG_PATH = '/app/config/app.json';
let config = readConfig();

watch(CONFIG_PATH, (event) => {
  if (event === 'change') {
    try {
      config = readConfig();
      console.log('Configuração recarregada');
    } catch (err) {
      console.error('Falha ao recarregar configuração:', err);
    }
  }
});

function readConfig() {
  return JSON.parse(require('fs').readFileSync(CONFIG_PATH, 'utf-8'));
}
```

## Padrões Comuns e Boas Práticas

### External Secrets Operator

O External Secrets Operator sincroniza secrets de armazenamentos externos (AWS Secrets Manager, GCP Secret Manager, Azure Key Vault, HashiCorp Vault) para Secrets do Kubernetes automaticamente.

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace
```

```yaml
# SecretStore — define a conexão com o armazenamento externo
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
        # Usar IRSA (IAM Roles for Service Accounts) no EKS
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets

---
# ExternalSecret — mapeia o secret externo para um Secret do K8s
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
    name: db-credentials         # nome do Secret K8s a criar/atualizar
    creationPolicy: Owner
  data:
  - secretKey: DATABASE_URL      # chave no Secret K8s
    remoteRef:
      key: production/api/database  # caminho no AWS Secrets Manager
      property: url               # propriedade JSON dentro do secret
  - secretKey: DATABASE_PASSWORD
    remoteRef:
      key: production/api/database
      property: password
```

### Sealed Secrets (compatível com GitOps)

Sealed Secrets permitem armazenar secrets criptografados no git. Apenas o controller no cluster consegue descriptografá-los.

```bash
# Instalar o controller
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system

# Instalar a CLI kubeseal
brew install kubeseal

# Selar um secret (criptografar para o seu cluster)
kubectl create secret generic db-credentials \
  --from-literal=DATABASE_URL='postgresql://...' \
  --dry-run=client \
  -o yaml | \
  kubeseal \
    --controller-name=sealed-secrets \
    --controller-namespace=kube-system \
    --format yaml \
  > sealed-db-credentials.yaml

# Fazer commit no git (seguro — apenas seu cluster consegue descriptografar)
git add sealed-db-credentials.yaml
git commit -m "chore: add sealed database credentials"
```

```yaml
# sealed-db-credentials.yaml — seguro para armazenar no git
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  encryptedData:
    DATABASE_URL: AgBy8hBl...  # texto cifrado criptografado
```

### Secrets Escopados por Namespace vs Cluster

```yaml
# Padrão: escopado por namespace — apenas pods em 'production' podem usá-lo
metadata:
  namespace: production

# Para secrets compartilhados (ex.: credenciais de registry): replique para cada namespace
# Ou use ClusterSecretStore com External Secrets Operator
```

### Padrão de Separação de Ambientes

```
secrets/
  production/
    db-credentials.yaml       # sealed ou referência externa
    app-secrets.yaml
  staging/
    db-credentials.yaml
    app-secrets.yaml

configmaps/
  base/
    api-config.yaml           # padrões compartilhados
  production/
    api-config-patch.yaml     # sobrescritas de produção
  staging/
    api-config-patch.yaml     # sobrescritas de staging
```

## Anti-Padrões a Evitar

**Armazenar secrets em texto simples no git**
```yaml
# NUNCA faça isso
data:
  DATABASE_PASSWORD: mypassword123   # texto simples no git = violação esperando para acontecer

# Use Sealed Secrets ou External Secrets Operator
```

**Usar ConfigMaps para secrets**
```yaml
# Ruim: ConfigMaps não têm criptografia, auditoria nem diferenciação RBAC
kind: ConfigMap
data:
  JWT_SECRET: "my-jwt-secret"

# Bom
kind: Secret
data:
  JWT_SECRET: <base64>
```

**Montar todos os secrets como variáveis de ambiente (`envFrom`)**
```yaml
# Arriscado: expõe TODAS as chaves no secret, incluindo as futuras
envFrom:
- secretRef:
    name: all-the-secrets

# Melhor: mapeamento explícito apenas do que o pod precisa
env:
- name: DATABASE_URL
  valueFrom:
    secretKeyRef:
      name: db-credentials
      key: DATABASE_URL
```

**Não definir `readOnly: true` nos volume mounts**
```yaml
# Ruim: o container pode modificar a configuração montada
volumeMounts:
- name: app-config
  mountPath: /app/config

# Bom
volumeMounts:
- name: app-config
  mountPath: /app/config
  readOnly: true
```

**Usar ConfigMaps para dados binários grandes**
ConfigMaps têm um limite de tamanho de 1 MiB. Para arquivos grandes, use um PersistentVolume ou um armazenamento de objetos (S3).

## Depuração e Troubleshooting

### Visualizando Valores de Secrets

```bash
# Visualizar um secret (os valores estão codificados em base64)
kubectl get secret db-credentials -n production -o yaml

# Decodificar uma chave específica
kubectl get secret db-credentials -n production \
  -o jsonpath='{.data.DATABASE_URL}' | base64 -d

# Decodificar todas as chaves
kubectl get secret db-credentials -n production -o json | \
  jq -r '.data | to_entries[] | "\(.key): \(.value | @base64d)"'
```

### Verificando se o Pod Recebeu a Configuração

```bash
# Verificar variáveis de ambiente em um pod em execução
kubectl exec -n production my-api-pod -- printenv | sort

# Verificar variável específica
kubectl exec -n production my-api-pod -- printenv DATABASE_URL

# Verificar arquivos montados
kubectl exec -n production my-api-pod -- ls /app/config
kubectl exec -n production my-api-pod -- cat /app/config/app.json
```

### ConfigMap Não Atualizando

```bash
# Verificar se o ConfigMap foi atualizado
kubectl get configmap api-config -n production -o yaml

# Arquivos montados em volume atualizam dentro de ~1 minuto
# Verificar o timestamp de inode dentro do pod
kubectl exec -n production my-api-pod -- ls -la /app/config

# Para variáveis de ambiente: reiniciar os pods
kubectl rollout restart deployment/api-server -n production
```

### External Secret Não Sincronizando

```bash
# Verificar o status do ExternalSecret
kubectl describe externalsecret db-credentials -n production

# Verificar os logs do External Secrets Operator
kubectl logs -n external-secrets deployment/external-secrets

# Verificar permissões IAM (para AWS)
# A role IRSA deve ter a permissão secretsmanager:GetSecretValue
# no ARN específico do secret
```

## Cenários do Mundo Real

### Cenário 1: Configuração Completa para um Microsserviço Node.js

```bash
# 1. Criar namespace
kubectl create namespace production

# 2. Criar ConfigMap a partir de um arquivo de configuração
kubectl create configmap api-config \
  --from-literal=LOG_LEVEL=info \
  --from-literal=NODE_ENV=production \
  --from-literal=PORT=3000 \
  --from-literal=CORS_ORIGINS=https://app.example.com \
  -n production

# 3. Criar secrets (ou aplicar a partir de Sealed Secrets em GitOps)
kubectl create secret generic api-secrets \
  --from-literal=DATABASE_URL="$(cat .env.production | grep DATABASE_URL | cut -d= -f2-)" \
  --from-literal=JWT_SECRET="$(openssl rand -hex 32)" \
  --from-literal=REDIS_URL="$(cat .env.production | grep REDIS_URL | cut -d= -f2-)" \
  -n production

# 4. Aplicar o deployment (referencia o acima)
kubectl apply -f k8s/production/deployment.yaml

# 5. Verificar
kubectl exec -n production deploy/api-server -- printenv | grep -E "DATABASE_URL|LOG_LEVEL|NODE_ENV"
```

### Cenário 2: Rotação de Configuração Sem Downtime

Quando um secret precisa ser rotacionado (ex.: mudança de senha do banco de dados):

```bash
# 1. Atualizar o Secret com o novo valor
kubectl create secret generic db-credentials \
  --from-literal=DATABASE_URL="postgresql://app:newpassword@postgres:5432/mydb" \
  -n production \
  --dry-run=client \
  -o yaml | kubectl apply -f -

# 2. Acionar um rolling restart (os pods carregam a nova variável de ambiente)
kubectl rollout restart deployment/api-server -n production

# 3. Monitorar o rollout
kubectl rollout status deployment/api-server -n production

# Para secrets montados em volume: os pods recebem as mudanças automaticamente em ~1 min
# Nenhum restart é necessário se a aplicação observa mudanças nos arquivos
```

## Leitura Adicional

- [ConfigMaps — Kubernetes Docs](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Secrets — Kubernetes Docs](https://kubernetes.io/docs/concepts/configuration/secret/)
- [External Secrets Operator](https://external-secrets.io/)
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
- [Encrypting Secret Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)

## Resumo

ConfigMaps armazenam configuração não sensível; Secrets armazenam dados sensíveis (codificados em base64, criptografados em repouso).

Métodos de injeção:
- `env[].valueFrom.configMapKeyRef` / `secretKeyRef` — variáveis individuais
- `envFrom[].configMapRef` / `secretRef` — todas as chaves de um recurso
- `volumes` + `volumeMounts` — arquivos (se atualizam automaticamente quando o ConfigMap muda; variáveis de ambiente não se atualizam)

Padrões de produção:
- **External Secrets Operator** — sincronize a partir do AWS Secrets Manager, GCP Secret Manager, Vault. Melhor para deployments nativos de nuvem.
- **Sealed Secrets** — criptografe secrets para armazenamento no git. Melhor para GitOps com kubeseal.
- Sempre defina `readOnly: true` nos volume mounts de configuração
- Use `secretKeyRef` explícito (não `envFrom secretRef`) para limitar a superfície de exposição de secrets
- Nunca armazene secrets em texto simples no git — use Sealed Secrets ou External Secrets

Depuração:
- `kubectl get secret <nome> -o jsonpath='{.data.CHAVE}' | base64 -d` para inspecionar valores
- `kubectl exec <pod> -- printenv` para verificar a injeção
- `kubectl rollout restart deployment/<nome>` para carregar Secrets atualizados em variáveis de ambiente
