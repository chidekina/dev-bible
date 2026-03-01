# Pods & Deployments

## Visão Geral

Pods e Deployments são os objetos de carga de trabalho fundamentais no Kubernetes. Um Pod é a menor unidade agendável — ele envolve um ou mais containers que compartilham um contexto de rede e armazenamento. Um Deployment gerencia um conjunto de Pods idênticos, fornecendo atualizações declarativas, histórico de rollout e rollback automático.

Na prática, você nunca cria Pods diretamente. Você define um Deployment (ou StatefulSet, DaemonSet, etc.) e o deixa gerenciar os Pods. Entender a spec de Pod em profundidade — resource requests/limits, liveness/readiness probes, variáveis de ambiente, volume mounts — é o que separa "funciona" de "funciona de forma confiável em produção".

## Pré-requisitos

- Conceitos básicos de Docker (images, portas, variáveis de ambiente)
- kubectl instalado e configurado
- Um cluster Kubernetes (kind ou minikube para uso local)

## Conceitos Principais

### Anatomia de um Pod

Uma spec de Pod define os containers a executar e suas configurações:

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

    # Declaração de porta (informativa — não controla a rede)
    ports:
    - name: http
      containerPort: 3000
      protocol: TCP

    # Variáveis de ambiente
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

    # Orçamento de recursos
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

  # Volumes disponíveis para todos os containers
  volumes:
  - name: config
    configMap:
      name: app-config

  # Quanto tempo esperar pelo graceful shutdown (padrão: 30s)
  terminationGracePeriodSeconds: 60

  # Política de reinicialização (Always, OnFailure, Never — padrão: Always)
  restartPolicy: Always

  # Service account para RBAC
  serviceAccountName: api-server
```

### Resource Requests vs Limits

Este é um dos aspectos mais mal compreendidos do Kubernetes.

**Requests** — o que o scheduler reserva. Um pod com `memory: 256Mi` solicitado não será colocado em um node com menos de 256Mi disponíveis. É o que o scheduler enxerga.

**Limits** — o teto. Se um container exceder seu memory limit, ele é OOMKilled. Se exceder seu CPU limit, ele é throttled (não morto).

```
Node tem 4 CPU, 8Gi de memória
Pods agendados:
  Pod A: request 1CPU/2Gi  limit 2CPU/4Gi
  Pod B: request 1CPU/2Gi  limit 2CPU/4Gi
  Pod C: request 1CPU/2Gi  limit 2CPU/4Gi  ← scheduler: 3CPU/6Gi reservados, OK
  Pod D: request 1CPU/2Gi              ← scheduler: 4CPU/8Gi reservados — CHEIO
  Pod E: request 0.5CPU/1Gi            ← Pending: capacidade reservada insuficiente
```

Unidades de CPU:
- `1` = 1 vCPU
- `0.5` = meio vCPU
- `500m` = 500 millicores = 0,5 vCPU
- `100m` = 100 millicores = 0,1 vCPU

Unidades de memória:
- `Mi` = mebibytes (1Mi = 1.048.576 bytes)
- `Gi` = gibibytes
- `M` = megabytes (1M = 1.000.000 bytes)
- Use `Mi`/`Gi`, não `M`/`G`

**Classes de QoS** (derivadas de requests/limits):
- **Guaranteed** — requests == limits (ambos definidos). Primeiro a ser agendado, último a ser despejado.
- **Burstable** — requests < limits (ou apenas um definido). Prioridade intermediária.
- **BestEffort** — sem requests ou limits. Despejado primeiro sob pressão.

Para produção: defina tanto requests quanto limits. Para bancos de dados e serviços com estado: use QoS Guaranteed.

### Liveness vs Readiness Probes

Duas probes, dois propósitos diferentes:

**Liveness probe** — "Este container está vivo?"
- Falhar = container está travado, não saudável
- Kubernetes **reinicia** o container
- Usada para detectar deadlocks e processos zumbi
- Não configure de forma muito agressiva — você vai causar loops de reinicialização

**Readiness probe** — "Este container está pronto para servir tráfego?"
- Falhar = remover temporariamente este Pod do load balancing do Service
- Kubernetes **não reinicia** o container
- Usada para tempo de inicialização, sobrecarga temporária, serviço dependente fora do ar
- O Pod continua rodando mas não recebe tráfego até a probe passar novamente

**Startup probe** (Kubernetes 1.16+) — "O container terminou de inicializar?"
- Atrasa as liveness probes durante inicialização lenta
- Evita que a liveness mate o container prematuramente durante a inicialização

```yaml
# Startup probe — desabilita liveness/readiness até esta passar
startupProbe:
  httpGet:
    path: /health
    port: 3000
  failureThreshold: 30    # 30 × 10s = 5 minutos para inicializar
  periodSeconds: 10

# Liveness — rodando mas travado?
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 0   # startup probe já tratou o atraso
  periodSeconds: 30
  failureThreshold: 3      # 3 falhas → reiniciar

# Readiness — pronto para tráfego?
readinessProbe:
  httpGet:
    path: /ready
    port: 3000
  initialDelaySeconds: 0
  periodSeconds: 10
  failureThreshold: 3      # 3 falhas → remover do LB
  successThreshold: 1      # 1 sucesso → adicionar de volta ao LB
```

Tipos de probe:
```yaml
# HTTP GET — mais comum para aplicações web
httpGet:
  path: /healthz
  port: 8080
  httpHeaders:
  - name: Custom-Header
    value: Awesome

# TCP socket — para servidores não-HTTP
tcpSocket:
  port: 3306

# Exec — rodar um comando
exec:
  command:
  - sh
  - -c
  - "redis-cli ping | grep PONG"

# gRPC — para serviços gRPC (Kubernetes 1.24+)
grpc:
  port: 9090
```

### Deployments

Um Deployment gerencia um ReplicaSet que gerencia Pods. Você interage com o Deployment — ele cuida do resto.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
spec:
  replicas: 3

  # Quais pods este deployment gerencia
  selector:
    matchLabels:
      app: api-server

  # Estratégia de rolling update (padrão)
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1    # no máximo 1 pod indisponível durante o rollout
      maxSurge: 1          # no máximo 1 pod extra durante o rollout

  # Template do Pod
  template:
    metadata:
      labels:
        app: api-server
        version: "2.4"
    spec:
      # ... (mesmo que a spec do Pod)
```

Comportamento do rolling update com `replicas: 3`, `maxUnavailable: 1`, `maxSurge: 1`:
```
Início:  [v1] [v1] [v1]
Passo 1: [v1] [v1] [v1] [v2]   ← surge: +1 novo
Passo 2: [v1] [v2] [v2]         ← mata um antigo, sobe um novo
Passo 3: [v2] [v2] [v2]         ← concluído
```

**Estratégia Recreate** — mata todos os pods antigos antes de criar os novos:
```yaml
strategy:
  type: Recreate
```
Use quando não for possível rodar duas versões simultaneamente (ex.: migrations de schema de banco de dados não retrocompatíveis). Causa downtime.

### Operações de Rollout do Deployment

```bash
# Verificar status do rollout
kubectl rollout status deployment/api-server

# Ver histórico de rollout
kubectl rollout history deployment/api-server

# Ver detalhes de uma revisão específica
kubectl rollout history deployment/api-server --revision=3

# Rollback para a versão anterior
kubectl rollout undo deployment/api-server

# Rollback para uma revisão específica
kubectl rollout undo deployment/api-server --to-revision=2

# Pausar um rollout (para fazer rollout em etapas manualmente)
kubectl rollout pause deployment/api-server

# Retomar um rollout pausado
kubectl rollout resume deployment/api-server

# Acionar um rollout sem alterar a spec (ex.: forçar novos pods)
kubectl rollout restart deployment/api-server
```

## Exemplos Práticos

### Deployment Completo com Todas as Boas Práticas

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
        # Força rollout quando o configmap muda (com helm --set checksum)
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
      # Distribuir entre zonas para maior resiliência
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app.kubernetes.io/name: api-server
      # Garantir que os pods não se acumulem no mesmo node
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

### Endpoint de Health Check em Node.js

```typescript
// src/health.ts
import { FastifyInstance } from 'fastify';

export async function healthRoutes(app: FastifyInstance) {
  // Liveness: o processo está vivo?
  app.get('/health', async () => {
    return { status: 'ok', timestamp: new Date().toISOString() };
  });

  // Readiness: conseguimos servir tráfego?
  app.get('/ready', async (req, reply) => {
    try {
      // Verificar conectividade com o banco de dados
      await app.db.raw('SELECT 1');

      // Verificar conectividade com o Redis
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

O Kubernetes envia SIGTERM antes de matar um container. Trate esse sinal:

```typescript
// src/server.ts
const app = Fastify({ logger: true });

// Registrar rotas
await app.register(routes);

// Iniciar servidor
await app.listen({ port: 3000, host: '0.0.0.0' });

// Graceful shutdown
async function shutdown(signal: string) {
  app.log.info(`Sinal ${signal} recebido, encerrando graciosamente`);

  try {
    // Parar de aceitar novas requisições
    await app.close();
    app.log.info('Servidor fechado');

    // Fechar conexões com o banco de dados
    await db.destroy();
    app.log.info('Conexões com o banco de dados fechadas');

    process.exit(0);
  } catch (err) {
    app.log.error('Erro durante o encerramento:', err);
    process.exit(1);
  }
}

// terminationGracePeriodSeconds dá tempo para finalizar requisições em andamento
process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));
```

## Padrões Comuns e Boas Práticas

### Init Containers para Tarefas de Configuração

```yaml
spec:
  initContainers:
  # Rodar migrations de banco de dados antes de a aplicação iniciar
  - name: run-migrations
    image: my-org/api-server:2.4.1
    command: ['node', 'dist/migrate.js']
    env:
    - name: DATABASE_URL
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: url

  # Aguardar uma dependência estar pronta
  - name: wait-for-redis
    image: busybox
    command:
    - sh
    - -c
    - |
      until nc -z redis-service 6379; do
        echo "Aguardando Redis..."
        sleep 2
      done
      echo "Redis está pronto"

  containers:
  - name: api
    image: my-org/api-server:2.4.1
    # executa apenas após ambos os init containers terem sucesso
```

### Pod Disruption Budget

Proteja contra a queda de muitos pods ao mesmo tempo (durante rolling updates, drenagem de nodes):

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-server-pdb
spec:
  minAvailable: 2          # Pelo menos 2 pods sempre disponíveis
  # OU: maxUnavailable: 1  # No máximo 1 pod indisponível por vez
  selector:
    matchLabels:
      app.kubernetes.io/name: api-server
```

### Annotations para Rastreamento de Mudanças

```yaml
metadata:
  annotations:
    # Quem fez o deploy
    deployment.kubernetes.io/deployed-by: "github-actions"
    # Quando
    deployment.kubernetes.io/deployed-at: "2024-01-15T14:30:00Z"
    # De qual commit git
    deployment.kubernetes.io/git-commit: "abc123def"
    # Qual execução do pipeline
    deployment.kubernetes.io/pipeline-run: "https://github.com/org/repo/actions/runs/123"
```

## Anti-Padrões a Evitar

**Sem resource requests ou limits**
```yaml
# Ruim: o scheduler fica cego, os nodes podem ser OOMKilled
containers:
- name: api
  image: my-app:latest

# Bom
resources:
  requests: { cpu: 100m, memory: 256Mi }
  limits: { cpu: 500m, memory: 512Mi }
```

**Sem health probes**
```yaml
# Ruim: Kubernetes roteia tráfego para pods que não estão prontos
# e não reinicia pods com deadlock
containers:
- name: api
  image: my-app:latest

# Bom: adicione ao mínimo uma readiness probe
readinessProbe:
  httpGet:
    path: /health
    port: 3000
```

**Usar a tag de image `latest`**
```yaml
# Ruim: imprevisível, rollback impossível
image: my-app:latest

# Bom
image: my-app:2.4.1
```

**Rodar como root**
```yaml
# Ruim: o container roda como root por padrão
containers:
- name: api
  image: my-app:2.4.1

# Bom
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
```

**Réplica única em produção**
```yaml
# Ruim: sem resiliência
replicas: 1

# Bom: mínimo 2, prefira 3+
replicas: 3
```

## Depuração e Troubleshooting

### Depuração do Ciclo de Vida do Pod

```bash
# Estado completo do pod
kubectl describe pod <name> -n <ns>

# Condições para verificar:
# PodScheduled: False → o scheduler não conseguiu posicioná-lo (restrições de recursos, taints)
# Initialized: False → init container falhou
# ContainersReady: False → readiness probe falhando ou container travou
# Ready: False → pod não está pronto para servir tráfego

# Estado do container:
# Waiting (Reason: CrashLoopBackOff) → container continua travando, verifique logs --previous
# Waiting (Reason: ImagePullBackOff) → não consegue fazer pull da image
# Running → OK
# Terminated (Reason: OOMKilled) → excedeu o memory limit, aumente-o
# Terminated (Exit Code: 1) → aplicação saiu com erro, verifique os logs
```

### Visualizando Eventos de OOMKill

```bash
# Ver eventos de OOMKill
kubectl get events --field-selector reason=OOMKilling -A

# Ou na saída do describe
kubectl describe pod <name> | grep -A5 "Last State"
# Last State: Terminated
#   Reason: OOMKilled
#   Exit Code: 137
#   Started: ...
#   Finished: ...
```

### Acompanhando um Rollout

```bash
# Monitorar rolling update em tempo real
kubectl rollout status deployment/api-server -n production --watch

# Se o rollout estiver travado, verificar eventos do pod
kubectl get pods -n production -l app=api-server
kubectl describe pod <novo-pod-pendente>

# Fazer rollback se algo estiver errado
kubectl rollout undo deployment/api-server -n production
```

## Cenários do Mundo Real

### Cenário 1: Deploy com Zero Downtime

Garanta zero downtime com configuração correta de probe + estratégia:

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0    # nunca derrube um pod antes que o substituto esteja pronto
      maxSurge: 1          # crie um pod extra primeiro
  template:
    spec:
      containers:
      - name: api
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          # O novo pod deve passar na readiness antes que o antigo seja encerrado
          initialDelaySeconds: 5
          periodSeconds: 5
          successThreshold: 2    # passar duas vezes antes de ser considerado pronto
        lifecycle:
          preStop:
            exec:
              # Aguardar para permitir que requisições em andamento terminem
              # após o SIGTERM ser enviado mas antes de o pod ser removido dos endpoints
              command: ["/bin/sh", "-c", "sleep 5"]
      terminationGracePeriodSeconds: 30
```

### Cenário 2: Deploy Canário com Labels

```bash
# Fazer deploy do canário (10% dos pods)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server-canary
spec:
  replicas: 1          # 1 de 10 no total = 10% do tráfego
  selector:
    matchLabels:
      app: api-server
      track: canary
  template:
    metadata:
      labels:
        app: api-server    # mesmo label → Service roteia aqui também
        track: canary
    spec:
      containers:
      - name: api
        image: my-org/api-server:2.5.0-rc1   # nova versão
        # ... restante da spec
EOF

# Monitorar o canário
kubectl logs -l track=canary -f
kubectl top pods -l track=canary

# Promover: atualizar o deployment estável para a nova versão
kubectl set image deployment/api-server api=my-org/api-server:2.5.0

# Remover o canário
kubectl delete deployment api-server-canary
```

## Leitura Adicional

- [Kubernetes Deployments — Official Docs](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Pod Disruption Budgets](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
- [Security Context for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)

## Resumo

Pods são a menor unidade executável; Deployments gerenciam conjuntos de Pods com rolling updates e rollback.

Campos principais da spec do Pod:
- **resources** — sempre defina tanto `requests` (para agendamento) quanto `limits` (para isolamento). Violações de memory limit causam OOMKill.
- **livenessProbe** — o container está travado? Kubernetes reinicia em caso de falha.
- **readinessProbe** — o container está pronto para tráfego? Kubernetes remove do load balancer em caso de falha.
- **startupProbe** — aplicações de inicialização lenta; atrasa as verificações de liveness durante a inicialização.
- **terminationGracePeriodSeconds** — tempo para graceful shutdown; trate SIGTERM na sua aplicação.
- **securityContext** — rode como não-root, filesystem somente leitura, elimine todas as capabilities.

Boas práticas de Deployment:
- `replicas: 3+` em produção — réplica única significa que qualquer interrupção = downtime
- `RollingUpdate` com `maxUnavailable: 0, maxSurge: 1` para zero downtime
- Nunca use a tag `latest` — use digests de image ou tags de versão explícitas
- Sempre adicione um `PodDisruptionBudget` para serviços críticos
- Use `kubectl rollout undo` para rollback rápido — não enfrente um deploy ruim
