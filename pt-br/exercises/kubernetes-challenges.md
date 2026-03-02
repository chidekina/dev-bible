# Desafios de Kubernetes

> Exercícios práticos de Kubernetes cobrindo manifests de workload, rede, escalonamento, segurança e depuração. Cada exercício reflete uma tarefa real de operações em produção. Ferramentas assumidas: kubectl, Helm 3, k9s (opcional), cluster local (minikube, kind ou k3d). Nível alvo: iniciante a intermediário.

---

## Exercício 1 — Manifest de Deployment + Service para um App Node.js (Fácil)

**Cenário:** Escreva manifests Kubernetes para fazer o deploy de uma API Node.js e expô-la dentro do cluster.

**Requisitos:**
- `Deployment` com 3 réplicas de `seu-registry/node-api:1.0.0`.
- Porta do container: `3000`. Resource requests: `cpu: 100m, memory: 128Mi`. Sem limits ainda.
- `Service` do tipo `ClusterIP` expondo a porta `80` direcionando para a porta `3000` do container.
- Label do app: `app: node-api`. Use `matchLabels` no seletor do Deployment.
- Health check: uma probe `exec` simples que roda `wget -qO- http://localhost:3000/health`.

**Critérios de Aceite:**
- [ ] `kubectl apply -f deployment.yaml` cria 3 pods em execução.
- [ ] `kubectl get pods -l app=node-api` mostra 3 pods com status `Running`.
- [ ] `kubectl exec -it <pod> -- wget -qO- http://node-api/health` retorna `{"status":"ok"}` (via o Service ClusterIP).
- [ ] O spec do pod inclui `imagePullPolicy: Always`.
- [ ] Os manifests usam `apiVersion: apps/v1` e `apiVersion: v1` corretamente.

**Dicas:**
1. O seletor do Deployment deve corresponder aos labels do template do pod:
   ```yaml
   selector:
     matchLabels:
       app: node-api
   template:
     metadata:
       labels:
         app: node-api
   ```
2. O `selector` do Service também deve corresponder a `app: node-api`.
3. Service ClusterIP: `type: ClusterIP` é o padrão — você pode omitir o campo `type`.
4. Verificar conectividade: `kubectl run tmp --image=busybox --restart=Never -it --rm -- wget -qO- http://node-api`.

---

## Exercício 2 — Probes de Liveness e Readiness (Fácil)

**Cenário:** Adicione probes de health check HTTP ao Deployment do Exercício 1. O app expõe `GET /health` (liveness) e `GET /ready` (readiness).

**Requisitos:**
- `livenessProbe`: HTTP GET `/health` na porta `3000`. Initial delay: 10s, period: 15s, failure threshold: 3.
- `readinessProbe`: HTTP GET `/ready` na porta `3000`. Initial delay: 5s, period: 10s, failure threshold: 2.
- O endpoint `/ready` retorna `503` quando o app ainda está carregando dados do banco de dados.
- A probe de liveness não deve reiniciar o pod durante uma inicialização lenta — use `startupProbe` para isso.
- `startupProbe`: HTTP GET `/health`, failure threshold: 30, period: 5s (permite até 150s para inicialização).

**Critérios de Aceite:**
- [ ] Um pod com probe de liveness com falha é reiniciado (simule desabilitando o endpoint de health temporariamente).
- [ ] Tráfego não é roteado para um pod com probe de readiness com falha (`kubectl get endpoints node-api` mostra o pod removido).
- [ ] A startup probe dá ao container 150 segundos para iniciar antes da liveness entrar em ação.
- [ ] `kubectl describe pod <nome>` mostra as três probes configuradas corretamente.
- [ ] Probes não disparam antes do container ter tempo para iniciar (initial delays definidos adequadamente).

**Dicas:**
1. A startup probe desabilita a liveness até ter sucesso. Uma vez que tem sucesso (primeiro sucesso), a liveness assume.
2. Readiness vs. liveness: readiness controla o roteamento de tráfego; liveness controla o reinício do pod.
3. `failureThreshold * periodSeconds` = tempo máximo de inicialização antes de um reinício. `30 * 5 = 150s`.
4. Teste readiness: `kubectl exec <pod> -- kill -STOP 1` pausa o processo → readiness falha → pod removido dos endpoints.

---

## Exercício 3 — Resource Requests e Limits (Fácil)

**Cenário:** Defina resource requests e limits no Deployment Node.js para garantir agendamento justo e prevenir um vizinho barulhento de consumir todos os recursos do nó.

**Requisitos:**
- `requests`: `cpu: 100m, memory: 128Mi`.
- `limits`: `cpu: 500m, memory: 256Mi`.
- Adicione um `LimitRange` ao namespace `default` que impõe um máximo de `cpu: 1, memory: 512Mi` por container.
- Adicione um `ResourceQuota` ao namespace: máx 10 pods, máx total `cpu: 4, memory: 2Gi`.
- Verifique que tentar criar um pod excedendo o `LimitRange` é rejeitado.

**Critérios de Aceite:**
- [ ] `kubectl describe pod <nome>` mostra os requests e limits corretos.
- [ ] Tentar criar um pod com `limits.memory: 1Gi` é rejeitado com um erro de `LimitRange`.
- [ ] `kubectl describe resourcequota` mostra uso atual vs. limites hard.
- [ ] A classe de QoS dos pods é `Burstable` (requests < limits).
- [ ] Um pod com requests e limits iguais teria QoS `Guaranteed` — explique a diferença em um comentário.

**Dicas:**
1. Classes de QoS: `Guaranteed` (requests == limits), `Burstable` (requests < limits), `BestEffort` (sem requests ou limits). Pods `Guaranteed` são os últimos a serem despejados sob pressão de memória.
2. `LimitRange` aplica-se por container. `ResourceQuota` aplica-se por namespace.
3. `cpu: 100m` = 0,1 cores de CPU. `cpu: 500m` = 0,5 cores de CPU.
4. Teste a quota: crie pods até a quota ser esgotada — o 11º pod deve ser rejeitado.

---

## Exercício 4 — Deploy com Helm: Crie um Chart do Zero (Médio)

**Cenário:** Empacote a API Node.js como um chart Helm para que possa ser deployada em múltiplos ambientes (dev, staging, prod) com diferentes configurações.

**Requisitos:**
- Estrutura do chart: `Chart.yaml`, `values.yaml`, `templates/deployment.yaml`, `templates/service.yaml`, `templates/ingress.yaml`.
- `values.yaml` deve incluir: `replicaCount`, `image.repository`, `image.tag`, `service.port`, `ingress.enabled`, `ingress.host`, `resources`.
- O Ingress só é criado se `ingress.enabled: true`.
- Overrides de prod: `values-prod.yaml` com `replicaCount: 5` e `resources.limits.memory: 512Mi`.
- O chart deve passar `helm lint` sem warnings.

**Critérios de Aceite:**
- [ ] `helm install node-api ./chart` deploya o app com os valores padrão.
- [ ] `helm install node-api ./chart -f values-prod.yaml` deploya com 5 réplicas.
- [ ] `helm template ./chart | kubectl apply --dry-run=client -f -` funciona sem erros.
- [ ] `helm lint ./chart` retorna "1 chart(s) linted, 0 chart(s) failed".
- [ ] Mudar `image.tag` em `values.yaml` é a única mudança necessária para fazer o deploy de uma nova versão de imagem.

**Dicas:**
1. `Chart.yaml` mínimo: `apiVersion: v2`, `name: node-api`, `version: 0.1.0`, `appVersion: "1.0.0"`.
2. Ingress condicional: `{{- if .Values.ingress.enabled }}` ... `{{- end }}`.
3. Imagem: `image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"`.
4. `helm upgrade --install` é idempotente — use-o no CI em vez de comandos separados de install/upgrade.

---

## Exercício 5 — Configurar Horizontal Pod Autoscaling (Médio)

**Cenário:** Configure o Deployment Node.js para escalonar automaticamente entre 2 e 10 réplicas com base na utilização de CPU.

**Requisitos:**
- `HorizontalPodAutoscaler` direcionando ao Deployment `node-api`.
- Escalone para cima quando a utilização média de CPU exceder 70% dos requests.
- Escalone para baixo quando a CPU cair abaixo de 50% por mais de 5 minutos (use `stabilizationWindowSeconds`).
- Réplicas mínimas: 2 (disponibilidade). Máximas: 10.
- O servidor de métricas deve estar em execução no cluster.

**Critérios de Aceite:**
- [ ] `kubectl get hpa` mostra o HPA com réplicas mín/máx corretas e métricas atuais.
- [ ] Simular carga de CPU dispara scale-up.
- [ ] Após remover a carga, a contagem de réplicas diminui de volta para 2 dentro de 10 minutos.
- [ ] `kubectl describe hpa node-api` mostra a configuração da janela de estabilização.
- [ ] `requests.cpu` está definido no container (HPA requer requests para calcular % de utilização).

**Dicas:**
1. HPA v2 YAML:
   ```yaml
   spec:
     scaleTargetRef:
       apiVersion: apps/v1
       kind: Deployment
       name: node-api
     minReplicas: 2
     maxReplicas: 10
     metrics:
       - type: Resource
         resource:
           name: cpu
           target:
             type: Utilization
             averageUtilization: 70
   ```
2. Estabilização no scale-down: `behavior.scaleDown.stabilizationWindowSeconds: 300` (5 minutos).
3. Servidor de métricas no minikube: `minikube addons enable metrics-server`.
4. Verifique métricas atuais: `kubectl top pods -l app=node-api`.

---

## Exercício 6 — Configurar uma NetworkPolicy (Médio)

**Cenário:** Por padrão, todos os pods em um cluster Kubernetes podem se comunicar entre si. Implemente uma política de negar tudo por padrão e depois permita tráfego seletivamente.

**Requisitos:**
- Crie uma `NetworkPolicy` que nega todo tráfego ingress e egress para todos os pods no namespace `default` por padrão.
- Permita: os pods `node-api` receberem tráfego do namespace `ingress-nginx` na porta 3000.
- Permita: os pods `node-api` se conectarem aos pods `postgres` na porta 5432.
- Permita: todos os pods alcançarem o serviço DNS do Kubernetes (UDP/TCP porta 53).
- Bloqueie: `node-api` conectando a qualquer IP externo (egress para a internet bloqueado).

**Critérios de Aceite:**
- [ ] Após aplicar a política de negar tudo, `kubectl exec` de um pod para outro falha.
- [ ] Após a política de permissão, `kubectl exec <pod-node-api> -- wget http://postgres:5432` conecta.
- [ ] `kubectl exec <pod-node-api> -- wget http://example.com` expira (egress externo bloqueado).
- [ ] Resolução DNS ainda funciona dentro dos pods: `nslookup kubernetes.default` tem sucesso.
- [ ] `kubectl describe networkpolicy` mostra os seletores de pod e namespace corretos.

**Dicas:**
1. Negar ingress por padrão: `spec.podSelector: {}` (corresponde a todos os pods) + `spec.ingress: []` (sem ingress permitido).
2. Negar egress por padrão: mesma estrutura com `spec.egress: []`.
3. Permitir DNS: adicione uma regra de egress para permitir UDP/TCP porta 53 para o namespace `kube-system`.
4. NetworkPolicy requer um plugin CNI que o suporte (Calico, Cilium, WeaveNet). Flannel não suporta.

---

## Exercício 7 — Injeção de ConfigMap e Secret (Fácil)

**Cenário:** Externalize a configuração do app Node.js usando um `ConfigMap` para configurações não-sensíveis e um `Secret` para credenciais de banco de dados.

**Requisitos:**
- `ConfigMap`: `LOG_LEVEL=info`, `PORT=3000`, `NODE_ENV=production`.
- `Secret`: `DATABASE_URL=postgres://user:pass@db:5432/myapp`, `JWT_SECRET=supersecret`.
- Injete o `ConfigMap` como variáveis de ambiente usando `envFrom`.
- Injete valores do `Secret` como env vars individuais usando `valueFrom.secretKeyRef`.
- Monte o `ConfigMap` como volume em `/app/config/app.conf` para um formato de arquivo de configuração secundário.

**Critérios de Aceite:**
- [ ] `kubectl exec <pod> -- env | grep LOG_LEVEL` retorna `LOG_LEVEL=info`.
- [ ] `kubectl exec <pod> -- env | grep DATABASE_URL` retorna a URL do banco de dados.
- [ ] `kubectl get secret node-api-secrets -o jsonpath='{.data.JWT_SECRET}' | base64 -d` retorna o valor do segredo.
- [ ] O volume do `ConfigMap` está montado em `/app/config/app.conf` e é legível dentro do pod.
- [ ] Segredos são codificados em base64 no manifest — não armazenados como texto puro no YAML commitado no git.

**Dicas:**
1. Crie secret a partir de literal: `kubectl create secret generic node-api-secrets --from-literal=DATABASE_URL=... --from-literal=JWT_SECRET=...`.
2. Nunca commite segredos em texto puro. Use `kubectl create secret` a partir de um pipeline de CI com valores de um vault.
3. `envFrom: [configMapRef: {name: node-api-config}]` injeta todas as chaves do ConfigMap como env vars.
4. Volume mount: `volumes: [{name: config, configMap: {name: node-api-config}}]` + `volumeMounts: [{name: config, mountPath: /app/config}]`.

---

## Exercício 8 — Deploy com Atualização Zero-Downtime (Médio)

**Cenário:** Faça o deploy de uma nova versão do app Node.js (`image.tag: 1.1.0`) com zero downtime. Garanta que o rollback seja rápido se a nova versão estiver quebrada.

**Requisitos:**
- Configure a estratégia `RollingUpdate` com `maxUnavailable: 0` e `maxSurge: 1`.
- A probe de readiness deve estar passando antes de o rollout considerar um pod pronto.
- Defina `minReadySeconds: 30` — novos pods devem estar estáveis por 30 segundos antes de o próximo pod ser substituído.
- Verifique o rollout com `kubectl rollout status`.
- Simule um deploy ruim (imagem que falha na readiness) e faça rollback com `kubectl rollout undo`.

**Critérios de Aceite:**
- [ ] `kubectl rollout status deployment/node-api` mostra "successfully rolled out" após a atualização.
- [ ] Durante o rollout, pelo menos 3 pods estão sempre `Running` e servindo tráfego.
- [ ] Uma imagem quebrada (`image.tag: quebrada`) faz o rollout travar — `kubectl rollout status` mostra pendente.
- [ ] `kubectl rollout undo deployment/node-api` restaura a versão anterior dentro de 60 segundos.
- [ ] `kubectl rollout history deployment/node-api` mostra pelo menos 2 revisões.

**Dicas:**
1. `maxUnavailable: 0` significa que nenhum pod é derrubado até que um novo esteja pronto. `maxSurge: 1` permite um pod extra durante o rollout.
2. `minReadySeconds: 30` previne um pod instável de avançar o rollout.
3. Para simular um deploy ruim: defina a imagem para uma tag inexistente. O pod fica em `ImagePullBackOff` e o rollout trava.
4. Anote o rollout: `kubectl annotate deployment node-api kubernetes.io/change-cause="Atualização para v1.1.0"` — isso aparece no `rollout history`.

---

## Exercício 9 — Configurar RBAC para uma Service Account de CI/CD (Médio)

**Cenário:** Seu pipeline de CI/CD precisa fazer deploy de atualizações no namespace `staging`. Crie uma service account com as permissões mínimas necessárias — nada além disso.

**Requisitos:**
- Crie uma `ServiceAccount` chamada `cicd-deployer` no namespace `staging`.
- Crie um `Role` que permite: `get`, `list`, `create`, `update`, `patch` em `Deployments`, `Services` e `ConfigMaps`.
- Crie um `RoleBinding` que vincula o role à service account.
- A service account não deve ser capaz de deletar recursos ou acessar outros namespaces.
- Gere um kubeconfig para a service account usar no CI.

**Critérios de Aceite:**
- [ ] `kubectl auth can-i update deployments --as=system:serviceaccount:staging:cicd-deployer -n staging` retorna `yes`.
- [ ] `kubectl auth can-i delete deployments --as=system:serviceaccount:staging:cicd-deployer -n staging` retorna `no`.
- [ ] `kubectl auth can-i get pods --as=system:serviceaccount:staging:cicd-deployer -n production` retorna `no` (sem acesso cross-namespace).
- [ ] O kubeconfig usa um token de um `Secret` do tipo `kubernetes.io/service-account-token`.
- [ ] O `Role` usa o princípio do menor privilégio — sem wildcard (`*`) em resource ou verb.

**Dicas:**
1. Token de service account (Kubernetes 1.24+): crie manualmente um `Secret` do tipo `kubernetes.io/service-account-token` referenciando a service account.
2. `kubectl auth can-i` é seu melhor amigo para testar regras RBAC antes de aplicá-las em produção.
3. `Role` vs. `ClusterRole`: use `Role` para permissões com escopo de namespace. `ClusterRole` é para permissões em todo o cluster.
4. Geração de kubeconfig: extraia o token do secret, o cert CA do cluster e construa o YAML do kubeconfig com comandos `kubectl config set-*`.

---

## Exercício 10 — Depurar um Pod em CrashLoopBackOff (Médio)

**Cenário:** Um pod está preso em `CrashLoopBackOff`. Percorra os passos sistemáticos de depuração para identificar e corrigir a causa raiz.

**Requisitos:**
- Faça o deploy do seguinte manifest intencionalmente quebrado e depure-o:
  ```yaml
  containers:
    - name: api
      image: node:20-alpine
      command: ["node", "dist/arquivo-inexistente.js"]  # arquivo não existe
      env:
        - name: DATABASE_URL
          value: ""  # vazio — app trava na inicialização
  ```
- Documente cada passo de depuração que você executar.
- Corrija a causa raiz — não apenas mascare o problema mudando a política de restart.
- Após a correção, o pod deve atingir o estado `Running` e permanecer assim por 5+ minutos.

**Critérios de Aceite:**
- [ ] Passo 1: `kubectl get pods` identifica o status `CrashLoopBackOff`.
- [ ] Passo 2: `kubectl describe pod <nome>` revela o código de saída e o último estado.
- [ ] Passo 3: `kubectl logs <nome> --previous` mostra o erro do container anterior (crashado).
- [ ] Passo 4: a causa raiz é identificada a partir dos logs (arquivo faltando + DATABASE_URL vazio).
- [ ] Passo 5: manifest corrigido é aplicado e o pod atinge `Running` com `0` restarts.

**Dicas:**
1. Código de saída 1 = erro da aplicação (verifique os logs). Código de saída 137 = OOMKilled (aumente o limit de memória). Código de saída 139 = segfault.
2. `kubectl logs <pod> --previous` mostra logs da última execução do container antes de crashar.
3. `kubectl describe pod <nome>` → seção `Last State` mostra o código de saída e motivo.
4. Para um pod que crasha antes mesmo do seu código rodar: use `command: ["sleep", "infinity"]` temporariamente para obter um shell dentro do container e depurar: `kubectl exec -it <pod> -- sh`.
