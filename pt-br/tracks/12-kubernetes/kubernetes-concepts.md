# Conceitos de Kubernetes

## Visão Geral

Kubernetes (K8s) é uma plataforma de orquestração de containers que automatiza o deploy, o escalonamento e o gerenciamento de aplicações containerizadas. Em vez de dizer ao Kubernetes o que fazer ("rode este container nesta máquina"), você declara o estado desejado ("quero 3 réplicas desta aplicação") e o Kubernetes trabalha continuamente para fazer a realidade corresponder à sua declaração.

Criado originalmente pelo Google e agora mantido pela CNCF, o Kubernetes tornou-se o padrão de fato para rodar containers em escala. Entender sua arquitetura — control plane versus worker nodes, o papel do etcd, como o scheduler posiciona cargas de trabalho — é essencial para troubleshooting, ajuste de performance e decisões de infraestrutura bem embasadas.

## Pré-requisitos

- Conceitos básicos de Docker (images, containers, portas, volumes)
- Redes básicas (endereços IP, DNS, portas, TCP/UDP)
- Sintaxe YAML
- Fundamentos de linha de comando Linux

## Conceitos Principais

### O Modelo Declarativo

A mudança fundamental que o Kubernetes introduz é a **configuração declarativa**: você descreve o que quer, não os passos para chegar lá.

```yaml
# Imperativo: "inicie um container rodando nginx"
docker run -d nginx

# Declarativo: "garanta que 3 pods de nginx estejam sempre rodando"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

O Kubernetes armazena esse estado desejado no etcd. Seus control loops comparam continuamente o estado desejado com o estado real e reconciliam as diferenças. Se um node falha e dois pods morrem, o Deployment controller percebe que a contagem de réplicas está errada e agenda dois novos pods.

### Control Plane vs Worker Nodes

Um cluster Kubernetes é composto por um **control plane** e **worker nodes**.

**Control Plane** (gerencia o cluster):

```
┌─────────────────────────────────────────────┐
│                Control Plane                 │
│                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │  kube-   │  │  kube-   │  │  cloud-  │  │
│  │  apiserver│  │ scheduler│  │controller│  │
│  └──────────┘  └──────────┘  └──────────┘  │
│                                             │
│  ┌──────────────────┐  ┌──────────────────┐ │
│  │ kube-controller- │  │      etcd        │ │
│  │     manager      │  │  (state store)   │ │
│  └──────────────────┘  └──────────────────┘ │
└─────────────────────────────────────────────┘
```

- **kube-apiserver** — A porta de entrada. Toda comunicação do cluster passa por ele (kubectl, outros componentes do control plane, kubelets). Valida, autentica e persiste objetos da API.
- **etcd** — Armazenamento chave-valor distribuído. A fonte única da verdade para todo o estado do cluster. Fazer backup do etcd equivale a fazer backup do seu cluster.
- **kube-scheduler** — Observa Pods sem agendamento e os atribui a nodes com base em requisitos de recursos, regras de affinity, taints/tolerations e topologia.
- **kube-controller-manager** — Executa uma coleção de controllers (Deployment controller, Node controller, ReplicaSet controller, etc.). Cada controller observa o API server e age quando o estado real diverge do estado desejado.
- **cloud-controller-manager** — Integra com APIs do provedor de nuvem (AWS, GCP, Azure) para provisionar load balancers, volumes persistentes e ciclo de vida de nodes.

**Worker Nodes** (executam suas cargas de trabalho):

```
┌───────────────────────────────┐
│          Worker Node          │
│                               │
│  ┌─────────┐  ┌────────────┐  │
│  │ kubelet │  │ kube-proxy │  │
│  └─────────┘  └────────────┘  │
│                               │
│  ┌────────────────────────┐   │
│  │   Container Runtime    │   │
│  │  (containerd / CRI-O)  │   │
│  └────────────────────────┘   │
│                               │
│  ┌──────┐ ┌──────┐ ┌──────┐  │
│  │ Pod  │ │ Pod  │ │ Pod  │  │
│  └──────┘ └──────┘ └──────┘  │
└───────────────────────────────┘
```

- **kubelet** — Agente rodando em cada node. Recebe especificações de Pod do API server e garante que os containers especificados estejam rodando e saudáveis. Reporta o status do node e dos Pods de volta ao API server.
- **kube-proxy** — Mantém regras de rede nos nodes. Implementa a abstração Service roteando tráfego para os Pods corretos. Usa iptables ou IPVS por baixo dos panos.
- **Container runtime** — O software que efetivamente roda os containers. O Kubernetes usa a Container Runtime Interface (CRI); os runtimes mais comuns são containerd e CRI-O.

### Objetos Principais da API

Tudo no Kubernetes é um objeto de API com:
- `apiVersion` — qual grupo e versão da API
- `kind` — tipo do objeto
- `metadata` — nome, namespace, labels, annotations
- `spec` — estado desejado
- `status` — estado atual (escrito pelos controllers)

**Pod** — A menor unidade implantável. Um Pod envolve um ou mais containers que compartilham:
- Mesmo namespace de rede (mesmo IP, mesmo `localhost`)
- Mesmos volumes de armazenamento
- Mesmo ciclo de vida (agendados juntos, morrem juntos)

**ReplicaSet** — Garante que N cópias de um Pod estejam sempre rodando. Raramente usado diretamente — use Deployments.

**Deployment** — Gerencia ReplicaSets e fornece atualizações declarativas com histórico de rollout e rollback.

**Service** — Endpoint de rede estável para um conjunto de Pods. Pods são efêmeros (seus IPs mudam); Services fornecem um IP fixo e um nome DNS.

**ConfigMap** — Dados de configuração não sensíveis (variáveis de ambiente, arquivos de configuração).

**Secret** — Dados sensíveis (senhas, tokens, certificados TLS) — codificados em base64 no etcd, criptografados em repouso com configuração adequada do cluster.

**Namespace** — Cluster virtual dentro de um cluster. Fornece isolamento de recursos e fronteiras de controle de acesso.

**Node** — Uma máquina worker (VM ou física).

**PersistentVolume / PersistentVolumeClaim** — Abstração de provisionamento de armazenamento.

**ServiceAccount** — Identidade para processos rodando em Pods.

### Labels e Selectors

Labels são pares chave/valor anexados a objetos. Selectors filtram objetos por labels. É assim que Services encontram seus Pods, como Deployments gerenciam seus ReplicaSets e como você organiza os recursos.

```yaml
metadata:
  labels:
    app: api-server
    version: "2.4"
    environment: production
    team: platform
```

Exemplos de selector:
```bash
# filtro com kubectl
kubectl get pods -l app=api-server
kubectl get pods -l environment=production,team=platform

# Na spec de um Service
selector:
  app: api-server
  environment: production
```

### O Loop de Reconciliação

Todo controller do Kubernetes executa um loop:

```
observar mudanças no API server
  → comparar estado desejado (spec) com estado real (status)
  → executar ação para aproximar o real do desejado
  → atualizar status
  → repetir
```

É por isso que o Kubernetes é **eventualmente consistente** e **auto-recuperável**. Deletou um Pod que pertence a um Deployment? O Deployment controller vê 2 réplicas em vez de 3 e cria uma nova em segundos.

### etcd — O Cérebro

O etcd é um armazenamento chave-valor distribuído construído sobre o algoritmo de consenso Raft. Todo o estado do cluster vive aqui. Propriedades principais:
- **Fortemente consistente** — leituras sempre retornam o último valor confirmado
- **Mecanismo de watch** — clientes podem observar chaves por mudanças (como os controllers são notificados)
- **Distribuído** — tipicamente 3 ou 5 nodes para HA (deve ser ímpar para quórum do Raft)

```bash
# Fazendo backup do etcd (faça isso regularmente em produção)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

## Exemplos Práticos

### Explorando a Arquitetura de um Cluster

```bash
# Ver todos os nodes (control plane + workers)
kubectl get nodes -o wide

# Descrever um node para ver recursos disponíveis, pods, condições
kubectl describe node worker-node-1

# Ver componentes do control plane rodando como pods (clusters com kubeadm)
kubectl get pods -n kube-system

# Observar as decisões do scheduler em tempo real
kubectl get events --sort-by=.metadata.creationTimestamp -n default -w
```

### Entendendo Grupos de API

```bash
# Listar todos os recursos de API disponíveis e seus grupos
kubectl api-resources

# Verificar versões de API para um recurso
kubectl explain deployment --api-version apps/v1
kubectl explain deployment.spec.strategy
```

### Observando a Reconciliação em Ação

```bash
# Terminal 1: observar pods
kubectl get pods -w

# Terminal 2: deletar um pod
kubectl delete pod my-app-7d9f8-xyz

# Observe: o Deployment controller imediatamente cria um substituto
```

### Verificando a Saúde do etcd (no node do control plane)

```bash
# Verificar saúde do cluster etcd
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key

# Listar todas as chaves (mostra o que está armazenado)
ETCDCTL_API=3 etcdctl get / --prefix --keys-only | head -30
```

## Padrões Comuns e Boas Práticas

### Estratégia de Namespace

```yaml
# Separar por ambiente
namespaces:
  - production
  - staging
  - development

# Ou por time + ambiente
namespaces:
  - platform-production
  - platform-staging
  - api-production
  - api-staging
```

Namespaces fornecem:
- Fronteiras de RBAC (times diferentes, acessos diferentes)
- Aplicação de resource quota
- Escopo de NetworkPolicy
- Isolamento de DNS (`service.namespace.svc.cluster.local`)

### Use Labels de Forma Consistente

Adote uma convenção de labels e aplique em todo lugar:

```yaml
metadata:
  labels:
    app.kubernetes.io/name: api-server
    app.kubernetes.io/version: "2.4.1"
    app.kubernetes.io/component: backend
    app.kubernetes.io/part-of: commerce-platform
    app.kubernetes.io/managed-by: helm
    environment: production
    team: platform
```

O prefixo `app.kubernetes.io/` é uma convenção do Kubernetes suportada por muitas ferramentas.

### Resource Requests e Limits (Sempre Defina)

O scheduler usa requests para decidir o posicionamento. Limits evitam que containers descontrolados prejudiquem os vizinhos.

```yaml
resources:
  requests:
    cpu: 100m      # 0,1 núcleos de CPU — o que o scheduler reserva
    memory: 128Mi  # o que o scheduler reserva
  limits:
    cpu: 500m      # 0,5 núcleos — limite rígido
    memory: 512Mi  # limite rígido — exceder isso → OOMKilled
```

Sem requests, o scheduler não tem dados e posiciona pods aleatoriamente. Sem limits, um único pod com bug pode afogar um node inteiro.

## Anti-Padrões a Evitar

**Rodar cargas de trabalho nos nodes do control plane** — taints evitam isso por padrão; não os remova.

**Não fazer backup do etcd** — perder o etcd equivale a perder o estado do cluster. Automatize snapshots diários.

**Usar o namespace `default` para tudo** — torna controle de acesso, resource quotas e limpeza impossíveis.

**Ignorar resource requests e limits** — leva a problemas de vizinho barulhento e agendamento imprevisível.

**Usar a tag de image `latest`** — torna rollbacks impossíveis e puxa versões de image imprevisíveis. Sempre use tags de versão específicas.

**Acesso direto ao control plane para cargas de trabalho** — Pods devem usar ServiceAccounts, não arquivos kubeconfig montados como Secrets.

## Depuração e Troubleshooting

### Verificação de Saúde do Cluster

```bash
# Verificar status dos componentes
kubectl get componentstatuses

# Verificar condições dos nodes
kubectl get nodes
kubectl describe node <node-name> | grep -A5 Conditions

# Verificar pods do sistema
kubectl get pods -n kube-system

# Ver eventos do cluster (ordenados por mais recente)
kubectl get events --sort-by=.metadata.creationTimestamp --all-namespaces | tail -30
```

### Problemas no API Server

```bash
# Verificar logs do API server (no node do control plane)
sudo journalctl -u kube-apiserver --since "10 min ago"

# Ou se estiver rodando como pod:
kubectl logs -n kube-system kube-apiserver-<node-name>
```

### Node Not Ready

```bash
kubectl describe node <node-name>
# Procure: Conditions, Events, Allocated resources

# Verificar kubelet no node
sudo systemctl status kubelet
sudo journalctl -u kubelet --since "5 min ago"
```

## Cenários do Mundo Real

### Cenário 1: Pod Fica Reiniciando

```bash
# Ver contagem de reinicializações
kubectl get pod my-app-7d9f8-xyz

# Verificar logs atuais
kubectl logs my-app-7d9f8-xyz

# Verificar logs do container anterior (após reinicialização)
kubectl logs my-app-7d9f8-xyz --previous

# Descrever para ver eventos e estado
kubectl describe pod my-app-7d9f8-xyz
# Procure: OOMKilled → aumente o memory limit
# Procure: CrashLoopBackOff → verifique os logs da aplicação
# Procure: ImagePullBackOff → verifique nome/tag da image e pull secret
```

### Cenário 2: Entendendo Decisões de Agendamento

```bash
# Pod está Pending — por quê?
kubectl describe pod my-app-pending
# Seção Events: "0/3 nodes are available: 3 Insufficient memory"
# Solução: aumente os memory requests ou adicione mais nodes

# Ou: "0/3 nodes have the required toleration"
# Solução: verifique taints nos nodes e tolerations na spec do pod
```

## Leitura Adicional

- [Kubernetes Documentation — Concepts](https://kubernetes.io/docs/concepts/)
- [Kubernetes the Hard Way — Kelsey Hightower](https://github.com/kelseyhightower/kubernetes-the-hard-way) — entenda construindo do zero
- [The Kubernetes Book — Nigel Poulton](https://nigelpoulton.com/books/) — introdução concisa e prática
- [etcd Documentation](https://etcd.io/docs/) — clustering, operações, backup
- [Kubernetes API Reference](https://kubernetes.io/docs/reference/kubernetes-api/)

## Resumo

Kubernetes é um orquestrador declarativo de containers. Você declara o estado desejado; os controllers o reconciliam com a realidade continuamente.

Arquitetura:
- **Control plane**: API server (porta de entrada), etcd (estado), scheduler (posicionamento de pods), controller-manager (loops de reconciliação)
- **Worker nodes**: kubelet (roda pods), kube-proxy (rede), container runtime (containerd/CRI-O)

Conceitos-chave:
- **Modelo declarativo** — descreva o que quer, não como chegar lá
- **Labels e selectors** — como recursos se encontram e se agrupam
- **Loop de reconciliação** — cada controller observa desvios e os corrige
- **etcd** — a fonte única da verdade do cluster; faça backup
- **Namespaces** — fronteiras de isolamento e organização; não use apenas o `default`
- **Sempre defina resource requests/limits** — essencial para agendamento e estabilidade
- **Nunca use a tag de image `latest`** — use tags de versão explícitas para rollouts e rollbacks previsíveis
