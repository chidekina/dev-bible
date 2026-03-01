# Kubernetes em Produção

## Visão Geral

Rodar Kubernetes em produção significa adicionar controles de segurança, confiabilidade e operação sobre os Pods e Services básicos vistos anteriormente. Um cluster que funciona em staging pode falhar em produção por causa de RBAC ausente (escalada de privilégios), NetworkPolicies ausentes (movimentação lateral irrestrita), Pod Disruption Budgets ausentes (rolling updates derrubam o service) ou resource quotas ausentes (o deployment descontrolado de um time afoga as cargas de trabalho de outro).

Este capítulo cobre os controles que separam um cluster pronto para produção de um cluster de desenvolvimento: RBAC para controle de acesso, NetworkPolicy para segmentação de rede, PodDisruptionBudget para confiabilidade, HPA para autoescalamento, resource quotas para equidade em ambientes multi-tenant e uma estratégia de múltiplos namespaces.

## Pré-requisitos

- Pods, Deployments, Services (capítulos anteriores)
- kubectl básico
- Helm básico (para padrões de implantação de recursos)

## Conceitos Principais

### RBAC — Controle de Acesso Baseado em Funções

O RBAC do Kubernetes controla quem pode fazer o quê com quais recursos. Toda requisição à API é autenticada (quem é você?) e então autorizada (você tem permissão?).

Os quatro objetos de RBAC:
- **Role** — conjunto de permissões com escopo de namespace
- **ClusterRole** — conjunto de permissões em todo o cluster (ou aplicável em todos os namespaces)
- **RoleBinding** — vincula um Role (ou ClusterRole) a um usuário/grupo/ServiceAccount dentro de um namespace
- **ClusterRoleBinding** — vincula um ClusterRole a um usuário/grupo/ServiceAccount em todo o cluster

**Subjects** — a quem o binding se aplica:
- `User` — usuário humano (gerenciado pelo provedor de autenticação)
- `Group` — grupo de usuários
- `ServiceAccount` — identidade para um Pod ou processo

```yaml
# Role: permissões com escopo de namespace
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
# RoleBinding: vincular o role a um usuário em um namespace
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
# RBAC de ServiceAccount — Pods usam ServiceAccounts para acesso à API
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
# Permitir apenas leitura de ConfigMaps (para configuração dinâmica)
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

**ClusterRoles comuns** embutidos no Kubernetes:
| ClusterRole | Acesso |
|-------------|--------|
| `cluster-admin` | Controle total de tudo |
| `admin` | Controle total em um namespace |
| `edit` | Leitura/escrita na maioria dos recursos com escopo de namespace |
| `view` | Somente leitura na maioria dos recursos com escopo de namespace |

### NetworkPolicy

Por padrão, todos os Pods podem se comunicar com todos os outros Pods no cluster. NetworkPolicy restringe isso — pense nela como um firewall para tráfego pod a pod.

NetworkPolicy requer um plugin CNI que a suporte (Calico, Cilium, Weave Net). Flannel não suporta NetworkPolicy.

```yaml
# Negar todo ingress e egress por padrão em um namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}         # aplica a todos os pods
  policyTypes:
  - Ingress
  - Egress
  # Sem regras de ingress/egress = negar tudo
```

```yaml
# Permitir apenas api-server acessar postgres
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
# Permitir egress do api-server para postgres e Redis, e para a internet (APIs externas)
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
  # Para postgres
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - port: 5432
  # Para Redis
  - to:
    - podSelector:
        matchLabels:
          app: redis
    ports:
    - port: 6379
  # Para DNS (sempre necessário)
  - to:
    - namespaceSelector: {}
    ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
  # Para serviços externos (internet)
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8      # excluir IPs internos do cluster
        - 172.16.0.0/12
        - 192.168.0.0/16
    ports:
    - port: 443
```

### PodDisruptionBudget (PDB)

Um PDB limita quantos Pods de um Deployment/StatefulSet podem estar simultaneamente indisponíveis durante disrupções voluntárias (rolling updates, drenagem de nodes, upgrades de cluster). Ele NÃO protege contra disrupções involuntárias (falhas de node, OOMKill).

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-server-pdb
  namespace: production
spec:
  # Pelo menos 2 pods sempre disponíveis
  minAvailable: 2

  # OU: no máximo 1 pod indisponível por vez
  # maxUnavailable: 1

  # OU: porcentagem
  # minAvailable: "80%"

  selector:
    matchLabels:
      app.kubernetes.io/name: api-server
```

Um PDB com `minAvailable: 2` e `replicas: 3` significa:
- Rolling update: substitui um pod por vez (bloqueia em 1 indisponível até que o substituto esteja pronto)
- `kubectl drain node`: a drenagem do node aguarda o pod substituto estar Running antes de despejar

```bash
# Verificar status do PDB
kubectl get pdb -n production
# NAME           MIN AVAILABLE  MAX UNAVAILABLE  ALLOWED DISRUPTIONS  AGE
# api-server-pdb  2              N/A              1                    5h
# ALLOWED DISRUPTIONS = réplicas atuais - minAvailable = 3 - 2 = 1
```

### HPA — Horizontal Pod Autoscaler

O HPA escala automaticamente o número de réplicas de Pod com base em métricas observadas.

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
  # Escalar com base na utilização de CPU
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70   # alvo de 70% de CPU em todos os pods
  # Escalar com base na utilização de memória
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  # Escalar com base em métrica customizada (ex.: requisições por segundo do Prometheus)
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # aguardar 5 min antes de reduzir
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60    # máx. 2 pods removidos por minuto
    scaleUp:
      stabilizationWindowSeconds: 0     # escalar para cima imediatamente
      policies:
      - type: Pods
        value: 4
        periodSeconds: 60    # máx. 4 pods adicionados por minuto
```

O HPA requer que o `metrics-server` esteja instalado:
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Para métricas customizadas, use o Prometheus Adapter com o Prometheus instalado no cluster.

### Resource Quotas

ResourceQuota limita o total de recursos consumidos em um namespace — evita que um time consuma toda a capacidade do cluster.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    # Limites de contagem de pods
    pods: "50"
    # Limites de recursos de computação
    requests.cpu: "10"       # total de CPU requests no namespace
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    # Limites de contagem de objetos
    configmaps: "20"
    secrets: "20"
    services: "10"
    services.loadbalancers: "2"
    persistentvolumeclaims: "10"
```

```yaml
# LimitRange: define requests/limits padrão para pods que não os especificam
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

### Estratégia de Múltiplos Namespaces

Estruture namespaces em torno de times e ambientes:

```
Cluster
├── kube-system          (componentes do cluster — não mexa)
├── ingress-nginx        (ingress controller)
├── cert-manager         (automação de TLS)
├── monitoring           (Prometheus, Grafana)
├── platform-production  (time de plataforma — produção)
├── platform-staging     (time de plataforma — staging)
├── api-production       (time de API — produção)
├── api-staging          (time de API — staging)
└── data-production      (time de dados — produção)
```

Cada namespace recebe:
- Bindings de RBAC (time tem `edit` nos seus namespaces, `view` nos demais)
- ResourceQuota (orçamento de CPU/memória por namespace)
- LimitRange (requests/limits padrão)
- NetworkPolicy (negar por padrão, regras explícitas de permissão)

## Exemplos Práticos

### Verificando o Que Você Pode Fazer

```bash
# O que posso fazer neste namespace?
kubectl auth can-i --list -n production

# Posso deletar deployments?
kubectl auth can-i delete deployments -n production

# O que o ServiceAccount api-server pode fazer?
kubectl auth can-i --list \
  --as=system:serviceaccount:production:api-server \
  -n production
```

### Configurando um Novo Namespace de Time

```bash
#!/bin/bash
# scripts/setup-namespace.sh
TEAM=$1
ENV=$2
NS="${TEAM}-${ENV}"

# Criar namespace
kubectl create namespace "$NS"

# Aplicar NetworkPolicy de negação padrão
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

# Aplicar ResourceQuota
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

# Aplicar LimitRange
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

# Vincular o grupo do time ao role de edit
kubectl create rolebinding "${TEAM}-editors" \
  --clusterrole=edit \
  --group="${TEAM}-team" \
  --namespace="$NS"

echo "Namespace $NS configurado"
```

### Implementando Boas Práticas de Security Context

```yaml
# Spec de pod segura
spec:
  # Desabilitar montagem automática do token de service account (menor privilégio)
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
    # Diretório temp gravável quando readOnlyRootFilesystem: true
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
```

### Monitorando o Comportamento do HPA

```bash
# Observar o HPA escalar em tempo real
kubectl get hpa api-server-hpa -n production -w

# Gerar carga para acionar o escalonamento (de dentro do cluster)
kubectl run load-test --image=busybox --rm -it --restart=Never \
  -- sh -c "while true; do wget -qO- http://api-server.production/; done"

# Verificar eventos para decisões de escalonamento
kubectl describe hpa api-server-hpa -n production
```

## Padrões Comuns e Boas Práticas

### Princípio do Menor Privilégio para ServiceAccounts

```yaml
# Cada carga de trabalho recebe seu próprio ServiceAccount
# Vincule apenas as permissões exatas que ela precisa

# api-server precisa ler configmaps
kind: Role
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "watch", "list"]
  resourceNames: ["api-config"]  # restringir ao ConfigMap específico

# worker precisa escrever em um ConfigMap de fila
kind: Role
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "update", "patch"]
  resourceNames: ["job-queue"]
```

### NetworkPolicy Padrão por Namespace

Aplique isto a cada novo namespace como parte da configuração:

```yaml
# Permitir DNS em todo lugar (kube-dns), negar todo o resto por padrão
# Depois adicione permissões explícitas para cada service

---
# Permitir egress de DNS de tudo
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
# Negar todo o outro ingress e egress
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

### Taints e Tolerations para Node Pools

Separe cargas de trabalho entre node pools dedicados:

```yaml
# Aplicar taint a um node pool (feito pelo provedor de nuvem ou manualmente)
kubectl taint nodes node-pool-gpu gpu=true:NoSchedule

# Adicionar toleration aos pods que devem rodar nos nodes GPU
spec:
  tolerations:
  - key: gpu
    operator: Equal
    value: "true"
    effect: NoSchedule
  nodeSelector:
    node-pool: gpu          # ou use nodeAffinity para mais controle
```

## Anti-Padrões a Evitar

**Usar `cluster-admin` para tudo**
```bash
# Ruim: todos os ServiceAccounts são cluster admins
kubectl create clusterrolebinding default-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=default:default

# Bom: atribua permissões mínimas necessárias por ServiceAccount
```

**Sem NetworkPolicy (rede plana)**
Sem NetworkPolicy, qualquer pod comprometido pode alcançar qualquer outro pod, incluindo bancos de dados e armazenamentos de secrets. Sempre aplique negar-por-padrão e permissões explícitas.

**Sem PodDisruptionBudget para serviços com estado**
```bash
# Um upgrade de cluster ou drenagem de node sem PDB pode despejar todas as réplicas simultaneamente
# Sempre adicione PDB para services com replicas > 1
```

**HPA sem requests definidos**
```yaml
# Ruim: HPA não pode calcular % de CPU/memória sem requests
containers:
- name: api
  image: my-app:1.0.0
  # Sem resources.requests → HPA não consegue funcionar

# Bom: sempre defina requests para alvos de HPA
resources:
  requests:
    cpu: 100m
    memory: 256Mi
```

**Permissões RBAC com wildcard**
```yaml
# Ruim: concede acesso total a todos os recursos
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]

# Bom: recursos e verbos específicos
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "update", "patch"]
```

## Depuração e Troubleshooting

### Depuração de RBAC

```bash
# Por que uma requisição está sendo proibida?
kubectl auth can-i create pods -n production \
  --as=system:serviceaccount:production:api-server
# Retorna yes/no

# Obter visão detalhada de RBAC para um ServiceAccount
kubectl get rolebindings,clusterrolebindings -A \
  -o json | jq '.items[] | select(
    .subjects[]? |
    (.kind=="ServiceAccount" and .name=="api-server" and .namespace=="production")
  ) | .metadata.name'

# Usar kubectl-who-can (plugin krew)
kubectl krew install who-can
kubectl who-can create pods -n production
```

### Testando NetworkPolicy

```bash
# Testar conectividade entre dois pods
kubectl run nettest --image=busybox --rm -it --restart=Never \
  --overrides='{"spec":{"serviceAccountName":"default"}}' \
  -- sh

# Dentro do pod:
wget -qO- http://postgres.production:5432   # deve falhar com NetworkPolicy
wget -qO- http://api-server.production:80   # deve ter sucesso
nc -zv redis.production 6379               # teste TCP

# Ou com um pod nomeado:
kubectl exec api-server-pod -- nc -zv postgres.production 5432
```

### HPA Não Escalando

```bash
# Verificar se o metrics-server está funcionando
kubectl top pods -n production

# Verificar status do HPA
kubectl describe hpa api-server-hpa -n production
# Procure: "Warning: FailedGetResourceMetric" → problema no metrics-server
# Procure: "DesiredReplicas: X, CurrentReplicas: X" → pode já estar no máximo

# Verificar se requests estão definidos (requisito do HPA)
kubectl get pods -n production -o jsonpath='{.items[*].spec.containers[*].resources}'
```

### ResourceQuota Bloqueando Deployments

```bash
# Ver uso da quota
kubectl describe resourcequota production-quota -n production

# Ver o que está consumindo a quota
kubectl get pods -n production -o custom-columns=\
  'NAME:.metadata.name,CPU-REQ:.spec.containers[*].resources.requests.cpu,MEM-REQ:.spec.containers[*].resources.requests.memory'
```

## Cenários do Mundo Real

### Cenário 1: RBAC de Produção para um Time de Desenvolvimento

```bash
# Criar roles para diferentes responsabilidades
kubectl apply -f - <<EOF
# Desenvolvedores: somente leitura em produção
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
# Desenvolvedores: edit completo em staging
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
# On-call: pode reiniciar deployments em produção
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

### Cenário 2: Hardening Completo de Namespace de Produção

```bash
# Aplicar a pilha completa de hardening de produção
kubectl apply -f k8s/production/namespace-config/

# k8s/production/namespace-config/ contém:
# - resourcequota.yaml
# - limitrange.yaml
# - networkpolicy-default-deny.yaml
# - networkpolicy-allow-dns.yaml
# - networkpolicy-allow-ingress-controller.yaml
# NetworkPolicies por serviço são aplicadas junto com cada deploy de serviço
```

### Cenário 3: Drenagem Graciosa de Node (Rolling Cluster Upgrade)

```bash
# Cordon: parar de agendar novos pods neste node
kubectl cordon node-1

# Drain: despejar todos os pods (respeita PDBs — não violará minAvailable)
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data

# O node agora está seguro para upgrade de OS, versão do Kubernetes, etc.
# ...fazer upgrade do node...

# Uncordon: permitir agendamento novamente
kubectl uncordon node-1

# Se o drain travar: verificar qual PDB está bloqueando
kubectl get pdb -A
# Se o PDB estiver mal configurado (minAvailable > replicas), corrija-o:
kubectl patch pdb api-server-pdb -n production \
  -p '{"spec":{"minAvailable":1}}'
```

## Leitura Adicional

- [RBAC Authorization — Kubernetes Docs](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Network Policies — Kubernetes Docs](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Pod Disruption Budgets](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
- [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [NSA/CISA Kubernetes Hardening Guide](https://media.defense.gov/2022/Aug/29/2003066362/-1/-1/0/CTR_KUBERNETES_HARDENING_GUIDANCE_1.2_20220829.PDF)

## Resumo

Kubernetes em produção requer controles de segurança e confiabilidade adicionados sobre as cargas de trabalho básicas.

**RBAC** — quem pode fazer o quê:
- Princípio do menor privilégio: cada ServiceAccount recebe apenas as permissões que precisa
- Use ClusterRoles embutidos (`view`, `edit`, `admin`) para acesso com escopo de namespace
- Nunca use `cluster-admin` para service accounts de aplicação
- `kubectl auth can-i` para verificar permissões

**NetworkPolicy** — o que pode falar com o quê:
- Negar por padrão todo ingress e egress em cada namespace
- Permitir explicitamente: ingress-controller → web/api, api → banco de dados, api → cache
- Sempre permitir egress de DNS (porta 53)

**PodDisruptionBudget** — continuidade do serviço durante disrupções:
- `minAvailable: 2` ou `maxUnavailable: 1` para todos os Deployments de produção com replicas > 1
- Sem PDB: upgrade de cluster ou drenagem de node pode despejar todos os pods de uma vez

**HPA** — dimensionamento correto sob carga:
- Requer que `resources.requests` esteja definido
- Configure `scaleDown.stabilizationWindowSeconds` (300s) para evitar oscilações
- Teste o comportamento de escalonamento antes da liberação para produção

**ResourceQuota + LimitRange** — equidade em ambientes multi-tenant:
- Quota limita o consumo total do namespace
- LimitRange define padrões para pods sem requests/limits explícitos

**Estratégia de namespace** — isolamento e organização:
- Namespaces separados por time × ambiente
- Aplique RBAC, NetworkPolicy e ResourceQuota a cada novo namespace
- Use um script de configuração para inicializar novos namespaces de forma consistente
