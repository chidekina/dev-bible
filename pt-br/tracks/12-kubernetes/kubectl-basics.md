# kubectl Básico

## Visão Geral

`kubectl` é a interface de linha de comando para o Kubernetes. Toda interação com um cluster — leitura de estado, aplicação de configuração, depuração de pods, redirecionamento de portas — passa pelo kubectl. Dominá-lo faz a diferença entre gastar 30 segundos depurando e gastar 30 minutos.

Este capítulo cobre gerenciamento de contexto, os comandos essenciais de leitura e depuração (`get`, `describe`, `logs`, `exec`), aplicação e gerenciamento de configuração, port-forwarding para depuração local e o fluxo de dry-run para deploys mais seguros.

## Pré-requisitos

- kubectl instalado (`brew install kubectl` ou download em kubernetes.io)
- Acesso a um cluster (local: kind, minikube, k3s; ou nuvem: EKS, GKE, AKS)
- Arquivo KUBECONFIG apontando para o seu cluster (fornecido pelo cluster ou `~/.kube/config`)

## Conceitos Principais

### Contextos e Clusters

O kubectl pode gerenciar conexões com múltiplos clusters usando **contextos**. Um contexto agrupa um cluster, um usuário (credenciais) e um namespace padrão.

```bash
# Ver o contexto atual e todos os contextos disponíveis
kubectl config current-context
kubectl config get-contexts

# Trocar de contexto
kubectl config use-context my-production-cluster

# Ver o kubeconfig completo
kubectl config view

# Definir um namespace padrão para um contexto (para não precisar de -n em todo lugar)
kubectl config set-context --current --namespace=my-namespace
```

Gerenciando múltiplos contextos com segurança:
```bash
# Instalar kubectx para troca rápida de contexto
brew install kubectx

kubectx                          # listar contextos
kubectx my-staging-cluster       # trocar para staging
kubectx -                        # voltar ao contexto anterior
kubens                           # listar namespaces no contexto atual
kubens my-namespace              # trocar o namespace padrão
```

### A Estrutura do Comando kubectl

```
kubectl [command] [TYPE] [NAME] [flags]

command: get, describe, apply, delete, logs, exec, port-forward, ...
TYPE:    pod, deployment, service, configmap, secret, node, ...
NAME:    nome específico do recurso (opcional — omita para listar todos)
flags:   -n <namespace>, -o <formato-de-saída>, -l <seletor-de-label>, ...
```

### Formatos de Saída

```bash
# Padrão: tabela legível por humanos
kubectl get pods

# Wide: mais colunas (node, IP, etc.)
kubectl get pods -o wide

# YAML: spec completa como armazenada no etcd
kubectl get deployment my-app -o yaml

# JSON: o mesmo que YAML mas em JSON
kubectl get pod my-app-xyz -o json

# Colunas customizadas
kubectl get pods -o custom-columns='NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName'

# JSONPath: extrair campos específicos
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}'

# Go template
kubectl get pods -o go-template='{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}'
```

## Exemplos Práticos

### Lendo Recursos

```bash
# Listar pods no namespace atual
kubectl get pods

# Listar pods em todos os namespaces
kubectl get pods --all-namespaces
kubectl get pods -A  # atalho

# Listar pods com seletor de label
kubectl get pods -l app=api-server
kubectl get pods -l environment=production,tier=backend

# Obter um pod específico
kubectl get pod my-app-7d9f8-xyz

# Observar recursos (atualizações em tempo real)
kubectl get pods -w
kubectl get pods --watch

# Listar múltiplos tipos de recursos
kubectl get pods,services,deployments

# Listar com informações adicionais (node, IP, nominated node, readiness gates)
kubectl get pods -o wide

# Ordenar por um campo
kubectl get pods --sort-by=.metadata.creationTimestamp
kubectl get pods --sort-by=.status.startTime
```

### Descrevendo Recursos

`describe` fornece o quadro completo: spec, status, eventos e condições. Use quando algo não está funcionando.

```bash
# Descrever um pod
kubectl describe pod my-app-7d9f8-xyz

# Descrever um node
kubectl describe node worker-node-1

# Descrever um deployment
kubectl describe deployment my-app

# Descrever um service
kubectl describe service my-app-svc

# Descrever sem saber o nome exato (lista os que correspondem)
kubectl describe pods -l app=my-app
```

Seções da saída do describe em que focar:
- **Conditions** — condições de saúde do Node ou Pod
- **Events** — atividade recente, avisos, erros (mais útil para depuração)
- **Containers** → **State**, **Last State** — estado atual e anterior do container

### Lendo Logs

```bash
# Logs de um pod (container único)
kubectl logs my-app-7d9f8-xyz

# Seguir (stream) os logs
kubectl logs -f my-app-7d9f8-xyz

# Últimas N linhas
kubectl logs my-app-7d9f8-xyz --tail=100

# Logs desde um período de tempo
kubectl logs my-app-7d9f8-xyz --since=1h
kubectl logs my-app-7d9f8-xyz --since=30m

# Logs do container anterior (após crash/reinicialização)
kubectl logs my-app-7d9f8-xyz --previous
kubectl logs my-app-7d9f8-xyz -p

# Logs de um container específico em um pod multi-container
kubectl logs my-app-7d9f8-xyz -c sidecar-container

# Logs de todos os pods com um label (agregados)
kubectl logs -l app=my-app --all-containers
kubectl logs -l app=my-app --tail=50

# Timestamps
kubectl logs my-app-7d9f8-xyz --timestamps
```

### Executando Comandos em Pods

```bash
# Shell interativo
kubectl exec -it my-app-7d9f8-xyz -- /bin/sh
kubectl exec -it my-app-7d9f8-xyz -- /bin/bash

# Comando único
kubectl exec my-app-7d9f8-xyz -- ls /app
kubectl exec my-app-7d9f8-xyz -- env
kubectl exec my-app-7d9f8-xyz -- cat /etc/hosts

# Em um container específico
kubectl exec -it my-app-7d9f8-xyz -c api -- /bin/sh

# Verificar resolução DNS de dentro de um pod
kubectl exec my-app-7d9f8-xyz -- nslookup my-service
kubectl exec my-app-7d9f8-xyz -- curl http://my-service:8080/health

# Verificar variáveis de ambiente
kubectl exec my-app-7d9f8-xyz -- printenv | grep DATABASE
```

### Port Forwarding

Port-forward cria um túnel da sua máquina local para um pod ou service — inestimável para depuração.

```bash
# Redirecionar a porta local 8080 para a porta 3000 do pod
kubectl port-forward pod/my-app-7d9f8-xyz 8080:3000

# Redirecionar para um service (seleciona automaticamente um pod saudável)
kubectl port-forward service/my-app-svc 8080:80

# Redirecionar para um deployment
kubectl port-forward deployment/my-app 8080:3000

# Ouvir em todas as interfaces (para que outras máquinas na rede possam acessar)
kubectl port-forward service/my-app-svc 8080:80 --address 0.0.0.0

# Port-forward para um banco de dados para acesso local
kubectl port-forward service/postgres 5432:5432 -n database
# Agora: psql -h localhost -p 5432 -U app mydb

# Port-forward em background
kubectl port-forward service/my-app-svc 8080:80 &
# Para encerrar:
kill %1
```

### Aplicando Configuração

```bash
# Aplicar um manifest (criar ou atualizar)
kubectl apply -f deployment.yaml

# Aplicar todos os manifests em um diretório
kubectl apply -f ./k8s/

# Aplicar recursivamente
kubectl apply -f ./k8s/ -R

# Aplicar a partir de uma URL
kubectl apply -f https://raw.githubusercontent.com/org/repo/main/manifests/app.yaml

# Mostrar diff antes de aplicar (dry run do lado do servidor)
kubectl diff -f deployment.yaml

# Dry run — validar sem aplicar
kubectl apply -f deployment.yaml --dry-run=client
kubectl apply -f deployment.yaml --dry-run=server

# create vs apply
kubectl create -f deployment.yaml   # falha se já existe
kubectl apply -f deployment.yaml    # criar ou atualizar (idempotente) ← sempre prefira este
```

### Deletando Recursos

```bash
# Deletar um recurso específico
kubectl delete pod my-app-7d9f8-xyz
kubectl delete deployment my-app

# Deletar a partir de um manifest (deleta o que o arquivo descreve)
kubectl delete -f deployment.yaml

# Deletar por label
kubectl delete pods -l app=my-app

# Deletar todos os recursos em um namespace
kubectl delete all --all -n my-namespace

# Forçar deleção de um pod travado (último recurso)
kubectl delete pod my-app-xyz --force --grace-period=0
```

## Padrões Comuns e Boas Práticas

### O Fluxo de Depuração

Sequência padrão de depuração quando algo está quebrado:

```bash
# 1. Qual é o estado geral?
kubectl get pods -n my-namespace

# 2. Qual pod está com problema?
kubectl get pods -n my-namespace | grep -v Running

# 3. O que aconteceu?
kubectl describe pod <pod-com-problema> -n my-namespace
# Leia a seção Events com atenção

# 4. O que a aplicação disse?
kubectl logs <pod> -n my-namespace
kubectl logs <pod> -n my-namespace --previous  # se travou

# 5. Consigo entrar?
kubectl exec -it <pod> -n my-namespace -- /bin/sh

# 6. O service está roteando corretamente?
kubectl get endpoints my-service -n my-namespace
# Se endpoints está vazio: o selector do service não corresponde a nenhum label de pod
```

### Aliases para o Dia a Dia

```bash
# ~/.zshrc ou ~/.bashrc
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgpa='kubectl get pods -A'
alias kgs='kubectl get services'
alias kgd='kubectl get deployments'
alias kgn='kubectl get nodes'
alias kdp='kubectl describe pod'
alias kdd='kubectl describe deployment'
alias kl='kubectl logs'
alias klf='kubectl logs -f'
alias kex='kubectl exec -it'
alias kaf='kubectl apply -f'
alias kdelf='kubectl delete -f'

# Habilitar autocompletion do kubectl
source <(kubectl completion zsh)
complete -F __start_kubectl k
```

### Server-Side Apply

Prefira server-side apply em CI/CD — ele rastreia a propriedade dos campos e lida melhor com conflitos:

```bash
kubectl apply --server-side -f deployment.yaml
kubectl apply --server-side -f ./k8s/
```

### Gerando Manifests a Partir de Comandos

Use `--dry-run=client -o yaml` para gerar templates de manifests:

```bash
# Gerar um manifest de deployment
kubectl create deployment my-app \
  --image=my-app:1.2.3 \
  --replicas=3 \
  --dry-run=client \
  -o yaml > deployment.yaml

# Gerar um manifest de service
kubectl expose deployment my-app \
  --port=80 \
  --target-port=3000 \
  --dry-run=client \
  -o yaml > service.yaml

# Gerar um ConfigMap a partir de um arquivo
kubectl create configmap app-config \
  --from-file=config.json \
  --dry-run=client \
  -o yaml > configmap.yaml
```

### Pods Temporários para Depuração

Quando você precisa de um pod para depuração (curl, dig, psql) sem modificar sua aplicação:

```bash
# Rodar um pod temporário de busybox
kubectl run debug --image=busybox --rm -it --restart=Never -- sh

# Rodar um pod com suporte a curl
kubectl run debug --image=curlimages/curl --rm -it --restart=Never -- sh

# Depurar a rede de um node específico
kubectl run debug --image=busybox --rm -it --restart=Never \
  --overrides='{"spec":{"nodeName":"worker-node-1"}}' -- sh

# Rodar com um service account específico
kubectl run debug --image=busybox --rm -it --restart=Never \
  --serviceaccount=my-service-account -- sh
```

## Anti-Padrões a Evitar

**Usar `kubectl create` em vez de `kubectl apply`**

`create` falha na segunda execução; `apply` é idempotente. Use `apply` em todo lugar.

**Editar recursos diretamente com `kubectl edit`**

```bash
# Evite em produção
kubectl edit deployment my-app

# Use em vez disso: atualize o arquivo YAML, aplique, faça commit no git
vim deployment.yaml && kubectl apply -f deployment.yaml
```

`kubectl edit` cria mudanças que não estão no controle de versão e contorna revisões.

**Forçar deleção de pods como primeira resposta**

```bash
# Não faça isso primeiro
kubectl delete pod --force --grace-period=0

# Faça isso primeiro: entenda o porquê
kubectl describe pod <name>
kubectl logs <name> --previous
```

**Acessar produção com o contexto errado**

```bash
# Adicione uma confirmação para comandos de produção
function kprod() {
  echo "AVISO: Você está prestes a executar: kubectl $@"
  echo "Contexto: $(kubectl config current-context)"
  read -p "Continuar? (y/N) " confirm
  [[ $confirm == "y" ]] && kubectl "$@"
}
```

## Depuração e Troubleshooting

### Pod em CrashLoopBackOff

```bash
kubectl describe pod <name>    # verificar Events
kubectl logs <name> --previous # verificar saída do último crash
kubectl logs <name>            # verificar saída atual
# Causas comuns: variáveis de ambiente ruins, comando de inicialização errado, dependência ausente, OOMKill
```

### ImagePullBackOff

```bash
kubectl describe pod <name>
# Events: Failed to pull image — verifique:
# 1. Erro de digitação no nome da image
# 2. A tag da image não existe
# 3. Registry é privado — falta o imagePullSecret
# 4. Registry está fora do ar ou com rate limit (Docker Hub)
```

### Pod Travado em Pending

```bash
kubectl describe pod <name>
# A seção Events mostra o motivo:
# "0/3 nodes available: 3 Insufficient memory" → adicione nodes ou reduza os requests
# "0/3 nodes have the required toleration" → verifique taints/tolerations
# "0/3 nodes available: persistentvolumeclaim not bound" → problema no PVC
```

### Service Não Roteando Tráfego

```bash
# Verificar que endpoints existem (o selector deve corresponder aos labels dos pods)
kubectl get endpoints my-service

# Se endpoints estiver vazio:
kubectl get pods -l app=my-app     # os pods devem existir
kubectl describe service my-service  # verificar selector
kubectl get pods --show-labels       # verificar labels

# Testar conectividade de dentro do cluster
kubectl run test --image=busybox --rm -it --restart=Never \
  -- wget -qO- http://my-service:80/health
```

### Verificando Uso de Recursos

```bash
# Uso de recursos do node (requer metrics-server)
kubectl top nodes

# Uso de recursos do pod
kubectl top pods
kubectl top pods --sort-by=memory
kubectl top pods -n my-namespace
```

## Cenários do Mundo Real

### Cenário 1: Investigando um Problema em Produção

```bash
# Passo 1: Ter uma visão geral
kubectl get pods -n production
# SAÍDA: my-api-7d8f9 está em estado Error, 2/3 disponíveis

# Passo 2: Obter detalhes
kubectl describe pod my-api-7d8f9 -n production
# Last State: Terminated, Exit Code 1, OOMKilled

# Passo 3: Aumentar o memory limit no deployment
kubectl get deployment my-api -n production -o yaml > /tmp/my-api.yaml
vim /tmp/my-api.yaml  # aumentar memory limit de 256Mi para 512Mi
kubectl apply -f /tmp/my-api.yaml

# Passo 4: Acompanhar o rollout
kubectl rollout status deployment/my-api -n production

# Passo 5: Verificar
kubectl top pods -n production
```

### Cenário 2: Script Automatizado de Health Check

```bash
#!/bin/bash
# scripts/k8s-health-check.sh

NAMESPACE="${1:-default}"
ERRORS=0

echo "=== Kubernetes Health Check: $NAMESPACE ==="

# Verificar pods que não estão Running
NOT_RUNNING=$(kubectl get pods -n "$NAMESPACE" --no-headers | grep -v "Running\|Completed" | wc -l)
if [ "$NOT_RUNNING" -gt 0 ]; then
  echo "WARN: $NOT_RUNNING pods não estão em estado Running"
  kubectl get pods -n "$NAMESPACE" | grep -v "Running\|Completed"
  ERRORS=$((ERRORS + 1))
fi

# Verificar pods com alta contagem de reinicializações
HIGH_RESTARTS=$(kubectl get pods -n "$NAMESPACE" -o jsonpath='{.items[*].status.containerStatuses[*].restartCount}' | tr ' ' '\n' | awk '$1 > 5' | wc -l)
if [ "$HIGH_RESTARTS" -gt 0 ]; then
  echo "WARN: $HIGH_RESTARTS containers reiniciaram mais de 5 vezes"
  ERRORS=$((ERRORS + 1))
fi

# Verificar services sem endpoints
for svc in $(kubectl get services -n "$NAMESPACE" -o jsonpath='{.items[*].metadata.name}'); do
  ENDPOINTS=$(kubectl get endpoints "$svc" -n "$NAMESPACE" -o jsonpath='{.subsets[*].addresses[*].ip}' 2>/dev/null)
  if [ -z "$ENDPOINTS" ] && [ "$svc" != "kubernetes" ]; then
    echo "WARN: Service $svc não tem endpoints"
    ERRORS=$((ERRORS + 1))
  fi
done

if [ "$ERRORS" -eq 0 ]; then
  echo "OK: Todas as verificações passaram"
  exit 0
else
  echo "FAIL: $ERRORS problema(s) encontrado(s)"
  exit 1
fi
```

## Leitura Adicional

- [kubectl Cheat Sheet — Kubernetes Docs](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [kubectl Reference Documentation](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands)
- [kubectx e kubens](https://github.com/ahmetb/kubectx) — troca rápida de contexto/namespace
- [stern](https://github.com/stern/stern) — tail de logs de múltiplos pods com regex
- [k9s](https://k9scli.io/) — interface de terminal para Kubernetes

## Resumo

`kubectl` é sua interface principal com o Kubernetes. Comandos essenciais:

- `kubectl config use-context` / `kubectx` — trocar de cluster
- `kubectl get <tipo>` — listar recursos; `-o wide`, `-o yaml`, `-l label` para filtrar
- `kubectl describe <tipo> <nome>` — detalhes completos incluindo Events (use isso ao depurar)
- `kubectl logs <pod>` — saída da aplicação; `--previous` para containers que travaram; `-f` para fazer stream
- `kubectl exec -it <pod> -- /bin/sh` — shell interativo dentro de um container
- `kubectl port-forward service/<nome> <local>:<remoto>` — túnel para um pod para depuração local
- `kubectl apply -f <arquivo>` — criar ou atualizar recursos (sempre prefira em relação a `create`)
- `kubectl diff -f <arquivo>` — ver o que vai mudar antes de aplicar
- `kubectl rollout status deployment/<nome>` — acompanhar o rollout de um deployment

Sempre use `apply`, não `create`. Mantenha mudanças no controle de versão. Use dry-run antes de operações destrutivas. Instale `kubectx`, `stern` e `k9s` para ganhar velocidade.
