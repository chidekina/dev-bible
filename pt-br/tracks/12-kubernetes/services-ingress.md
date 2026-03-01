# Services & Ingress

## Visão Geral

Pods são efêmeros — são criados e destruídos conforme seu cluster escala para cima e para baixo. Cada novo Pod recebe um novo endereço IP. Se você fixar IPs de Pods no código, sua aplicação quebra toda vez que um Pod reinicia. Os Services do Kubernetes resolvem isso fornecendo um IP virtual (ClusterIP) e um nome DNS estáveis que roteiam tráfego para Pods saudáveis.

O Ingress vai um nível acima: ele expõe rotas HTTP e HTTPS de fora do cluster para Services dentro dele, com roteamento baseado em host, roteamento por caminho e TLS termination — tudo configurado declarativamente.

Este capítulo cobre todos os tipos de Service, descoberta de serviço baseada em DNS, Ingress com nginx-ingress e automação de TLS com cert-manager.

## Pré-requisitos

- Cluster Kubernetes em execução
- Conceitos básicos de kubectl (ver `kubectl-basics.md`)
- Pods e Deployments (ver `pods-deployments.md`)
- Redes básicas (DNS, TCP/IP, portas, TLS)

## Conceitos Principais

### Tipos de Service

**ClusterIP** (padrão) — Apenas interno. Cria um IP virtual acessível de dentro do cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-server
  namespace: production
spec:
  type: ClusterIP          # padrão — pode omitir
  selector:
    app.kubernetes.io/name: api-server
  ports:
  - name: http
    port: 80               # porta que os clientes usam
    targetPort: 3000       # porta no Pod
    protocol: TCP
```

**NodePort** — Expõe o service em cada IP de Node em uma porta estática (30000-32767).

```yaml
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 3000
    nodePort: 30080      # opcional: se omitido, o Kubernetes atribui um
```

Acesso: `<qualquer-ip-de-node>:30080`. Usado para desenvolvimento ou quando você gerencia seu próprio load balancer externamente. Não recomendado para produção — expõe uma porta em cada node e contorna o Ingress.

**LoadBalancer** — Provisiona um cloud load balancer (AWS ELB, GCP LB, Azure LB). Cada Service recebe seu próprio IP externo.

```yaml
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
  - port: 443
    targetPort: 3000
```

Caro em escala — um cloud LB por Service. Use Ingress em vez disso para compartilhar um LB entre muitos services.

**ExternalName** — Mapeia um Service para um nome DNS externo (CNAME). Sem proxy — puramente DNS.

```yaml
spec:
  type: ExternalName
  externalName: database.us-east-1.rds.amazonaws.com
```

Acesso: `my-db-service.namespace.svc.cluster.local` resolve para o hostname externo. Útil para migrar gradualmente de serviços internos para externos.

**Headless Service** — Sem ClusterIP. Retorna IPs dos Pods diretamente via DNS. Usado por StatefulSets para fornecer nomes DNS estáveis por pod.

```yaml
spec:
  clusterIP: None     # torna o service headless
  selector:
    app: cassandra
```

### Como Services Roteiam Tráfego

Um Service usa um **selector** para encontrar Pods. O Kubernetes mantém um objeto **Endpoints** (ou EndpointSlice) — uma lista em tempo real de IPs e portas dos Pods saudáveis e prontos que correspondem ao selector.

```bash
# Ver os Endpoints de um service
kubectl get endpoints api-server -n production

# NAME        ENDPOINTS                                         AGE
# api-server  10.0.1.5:3000,10.0.1.6:3000,10.0.1.7:3000      5h
```

O `kube-proxy` em cada node observa os Endpoints e programa regras de iptables/IPVS para rotear tráfego para qualquer um dos IPs dos Pods. Quando um Pod fica não pronto (readiness probe falha), ele é removido dos Endpoints e para de receber tráfego.

### Descoberta de Serviço por DNS

Cada Service recebe automaticamente um nome DNS:

```
<nome-do-service>.<namespace>.svc.cluster.local
```

Dentro do mesmo namespace, você pode usar apenas o nome do service:

```typescript
// De um pod no namespace 'production':
await fetch('http://api-server/health');          // forma curta
await fetch('http://api-server.production/health'); // qualificado pelo namespace
await fetch('http://api-server.production.svc.cluster.local/health'); // FQDN
```

Entre namespaces:
```typescript
// De um pod no namespace 'frontend' acessando o backend em 'production':
await fetch('http://api-server.production.svc.cluster.local/v1/users');
```

Para headless services (StatefulSets), cada Pod também recebe:
```
<nome-do-pod>.<nome-do-service>.<namespace>.svc.cluster.local
```

Isso permite que membros de um StatefulSet se descubram:
```
cassandra-0.cassandra.production.svc.cluster.local
cassandra-1.cassandra.production.svc.cluster.local
cassandra-2.cassandra.production.svc.cluster.local
```

### Ingress

Ingress é um objeto de API que define regras de roteamento HTTP/HTTPS. Ele requer um **Ingress Controller** (não embutido no Kubernetes) para implementar essas regras.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls-cert   # Secret com certificado TLS + chave
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-server
            port:
              number: 80
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-server-v2
            port:
              number: 80
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-ui
            port:
              number: 80
```

**Tipos de path:**
- `Prefix` — corresponde a qualquer caminho que comece com o prefixo dado (`/v1` corresponde a `/v1`, `/v1/users`, `/v1/orders`)
- `Exact` — corresponde apenas ao caminho exato (`/v1` corresponde apenas a `/v1`, não a `/v1/users`)
- `ImplementationSpecific` — comportamento definido pelo Ingress controller

### Instalando o nginx-ingress

```bash
# Instalação com Helm (pronto para produção)
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.replicaCount=2 \
  --set controller.metrics.enabled=true \
  --set controller.podAnnotations."prometheus\.io/scrape"=true

# Obter o IP externo (cloud LoadBalancer)
kubectl get service ingress-nginx-controller -n ingress-nginx
# Aponte seus registros DNS para este IP
```

### TLS com cert-manager

O cert-manager automatiza o provisionamento de certificados TLS do Let's Encrypt (ou qualquer CA ACME).

```bash
# Instalar o cert-manager
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

```yaml
# ClusterIssuer — Let's Encrypt produção
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
    - http01:
        ingress:
          class: nginx
```

```yaml
# Ingress com TLS automático via cert-manager
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls-cert   # cert-manager cria este Secret
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-server
            port:
              number: 80
```

O cert-manager observa objetos Ingress com sua annotation, provisiona o certificado via desafio HTTP-01 e o armazena no Secret referenciado. Ele também renova os certificados automaticamente antes da expiração.

## Exemplos Práticos

### Service para uma API Node.js

```yaml
# k8s/api-server/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: api-server
  namespace: production
  labels:
    app.kubernetes.io/name: api-server
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: api-server
  ports:
  - name: http
    port: 80
    targetPort: 3000
```

```bash
# Aplicar
kubectl apply -f k8s/api-server/service.yaml

# Verificar endpoints (deve listar os IPs dos pods)
kubectl get endpoints api-server -n production

# Testar de dentro do cluster
kubectl run test --image=curlimages/curl --rm -it --restart=Never \
  -- curl http://api-server.production/health
```

### Ingress Multi-Service

```yaml
# k8s/ingress.yaml — roteia múltiplos services por um único load balancer
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  namespace: production
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    - app.example.com
    - admin.example.com
    secretName: example-com-tls
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-server
            port:
              number: 80
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-ui
            port:
              number: 80
```

### Testando a Descoberta de Serviço

```typescript
// src/services/user.service.ts
// Descoberta de serviço dentro do cluster via DNS
const USER_SERVICE_URL = process.env.USER_SERVICE_URL
  ?? 'http://user-service.services.svc.cluster.local';

export async function getUserById(id: string) {
  const response = await fetch(`${USER_SERVICE_URL}/users/${id}`);
  if (!response.ok) {
    throw new Error(`Erro no user service: ${response.statusText}`);
  }
  return response.json();
}
```

### Rate Limiting com nginx-ingress

```yaml
metadata:
  annotations:
    # Rate limiting: 10 requisições por segundo por IP
    nginx.ingress.kubernetes.io/limit-rps: "10"
    nginx.ingress.kubernetes.io/limit-connections: "20"
    # Resposta customizada para requisições com rate limit
    nginx.ingress.kubernetes.io/limit-req-status-code: "429"
    # Whitelist de IPs internos
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8"
```

### Autenticação com nginx-ingress (Basic Auth)

```bash
# Criar arquivo htpasswd
htpasswd -c auth admin
kubectl create secret generic basic-auth \
  --from-file=auth \
  -n monitoring
```

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Autenticação Necessária"
```

## Padrões Comuns e Boas Práticas

### Convenções de Nomenclatura de Service

```yaml
# Nomeie o service igual ao deployment
# Isso torna o DNS previsível: my-api.production.svc.cluster.local
metadata:
  name: my-api   # igual ao nome do deployment

# Use portas nomeadas — permite mudar a porta real sem atualizar todos os consumidores
ports:
- name: http      # consumidores usam o nome da porta, não o número
  port: 80
  targetPort: http   # corresponde ao nome de containerPort na spec do pod
```

### Session Affinity

Para aplicações que não suportam sessões distribuídas, habilite sticky sessions:

```yaml
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600    # 1 hora
```

Atenção: sticky sessions prejudicam a distribuição de carga. Melhor: use um armazenamento de sessão compartilhado (Redis).

### Services Internos vs Externos

```yaml
# API pública: exposta via Ingress
kind: Service
metadata:
  name: api-server
  annotations:
    # Evita provisionar cloud load balancer externo
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"

---
# Service de banco de dados interno apenas
kind: Service
metadata:
  name: postgres
spec:
  type: ClusterIP    # não acessível de fora do cluster
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
```

### NetworkPolicy para Restringir Acesso a Services

Restrinja quais services podem acessar seu banco de dados:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-access
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
  - Ingress
  ingress:
  - from:
    # Permitir apenas api-server e worker pods
    - podSelector:
        matchLabels:
          app.kubernetes.io/name: api-server
    - podSelector:
        matchLabels:
          app.kubernetes.io/name: worker
    ports:
    - protocol: TCP
      port: 5432
```

## Anti-Padrões a Evitar

**Usar NodePort em produção**
NodePort expõe uma porta em cada node, contorna o Ingress e não tem TLS termination. Use ClusterIP + Ingress em vez disso.

**Um LoadBalancer Service por aplicação**
Cada LoadBalancer provisiona um cloud load balancer ($$$). Use um Ingress com um LoadBalancer Service para o nginx-ingress em vez disso.

**Fixar IPs de Pod no código**
IPs de Pod mudam a cada reinicialização. Sempre use o nome DNS do Service.

**Ignorar TLS em services internos**
Tráfego entre services dentro de um cluster não é criptografado por padrão. Para dados sensíveis (tokens de autenticação, PII), use mutual TLS com um service mesh (Istio, Linkerd) ou, ao menos, TLS no nível do Ingress.

**Annotations de timeout ausentes no Ingress**
```yaml
# Ruim: o timeout padrão de 60s do proxy quebra requisições de longa duração
# Bom: defina timeouts explicitamente
nginx.ingress.kubernetes.io/proxy-read-timeout: "300"   # para uploads de arquivo
nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
nginx.ingress.kubernetes.io/proxy-connect-timeout: "5"
```

## Depuração e Troubleshooting

### Service Não Roteando Tráfego

```bash
# Passo 1: Verificar endpoints — se vazio, o selector não corresponde a nenhum pod
kubectl get endpoints my-service -n production
# Se a coluna ENDPOINTS mostrar <none>:

# Passo 2: Verificar o selector do service
kubectl describe service my-service -n production | grep Selector

# Passo 3: Verificar os labels dos pods
kubectl get pods -n production --show-labels | grep app=my-app
# Se os labels do pod não correspondem ao selector do service → corrija a discrepância

# Passo 4: Verificar a prontidão do pod (apenas pods Ready aparecem nos Endpoints)
kubectl get pods -n production -l app=my-app
# Se os pods mostram 0/1 READY → a readiness probe está falhando
# kubectl describe pod <name> para ver o motivo
```

### Ingress Não Funcionando

```bash
# Verificar se o Ingress controller está rodando
kubectl get pods -n ingress-nginx

# Verificar o recurso Ingress
kubectl describe ingress my-ingress -n production

# Verificar logs do nginx-ingress
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

# Testar com port-forward para contornar problemas de DNS/LB
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8443:443
curl -k -H "Host: api.example.com" https://localhost:8443/health
```

### Certificado Não Provisionado

```bash
# Verificar o recurso Certificate do cert-manager
kubectl get certificate -n production
kubectl describe certificate api-tls-cert -n production

# Verificar o CertificateRequest
kubectl get certificaterequests -n production

# Verificar logs do cert-manager
kubectl logs -n cert-manager deployment/cert-manager

# Problema comum: desafio HTTP-01 falhando
# O Ingress controller deve ser acessível a partir dos servidores do Let's Encrypt
# em http://<seu-dominio>/.well-known/acme-challenge/<token>
```

### Testando Resolução DNS

```bash
# De dentro de um pod
kubectl run dns-test --image=busybox --rm -it --restart=Never \
  -- nslookup api-server.production.svc.cluster.local

# Testar entre namespaces
kubectl run dns-test --image=busybox --rm -it --restart=Never \
  -- nslookup postgres.database.svc.cluster.local
```

## Cenários do Mundo Real

### Cenário 1: Microsserviços com Ingress Compartilhado

Três microsserviços atrás de um único Ingress:

```
api.acme.com/v1/*      → api-v1-service:80
api.acme.com/v2/*      → api-v2-service:80
app.acme.com/*         → frontend-service:80
admin.acme.com/*       → admin-service:80 (protegido com basic auth)
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: acme-ingress
  namespace: production
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts: [api.acme.com, app.acme.com, admin.acme.com]
    secretName: acme-tls
  rules:
  - host: api.acme.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service: { name: api-v1, port: { number: 80 } }
      - path: /v2
        pathType: Prefix
        backend:
          service: { name: api-v2, port: { number: 80 } }
  - host: app.acme.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: { name: frontend, port: { number: 80 } }
```

### Cenário 2: Troca Blue-Green via Selector de Service

```yaml
# Deployment blue (atual)
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-blue
spec:
  template:
    metadata:
      labels:
        app: api
        slot: blue
        version: "2.4.1"

# Deployment green (nova versão)
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-green
spec:
  template:
    metadata:
      labels:
        app: api
        slot: green
        version: "2.5.0"

# Service — troque entre blue e green atualizando o label slot
---
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
    slot: blue    # mude para 'green' para redirecionar todo o tráfego instantaneamente
```

```bash
# Trocar para green (instantâneo, sem rolling update)
kubectl patch service api \
  -p '{"spec":{"selector":{"slot":"green"}}}'

# Fazer rollback (instantâneo)
kubectl patch service api \
  -p '{"spec":{"selector":{"slot":"blue"}}}'
```

## Leitura Adicional

- [Kubernetes Services — Official Docs](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Ingress — Official Docs](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [nginx-ingress Documentation](https://kubernetes.github.io/ingress-nginx/)
- [cert-manager Documentation](https://cert-manager.io/docs/)
- [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)

## Resumo

Services fornecem aos Pods uma identidade estável (IP virtual + nome DNS) independente do ciclo de vida dos Pods. O tráfego é roteado para Pods saudáveis e prontos via Endpoints.

Tipos de Service:
- **ClusterIP** — apenas interno; padrão e mais comum
- **NodePort** — expõe em cada node; evite em produção
- **LoadBalancer** — um cloud LB por service; use com parcimônia
- **ExternalName** — CNAME para DNS externo; para migração ou abstração

Ingress roteia HTTP/HTTPS externo para Services internos via um cloud LoadBalancer:
- Instale o nginx-ingress via Helm
- Adicione `cert-manager` para TLS automático com Let's Encrypt
- Configure roteamento por host e caminho
- Use annotations para rate limiting, autenticação, timeouts

Lista de verificação para depuração:
1. `kubectl get endpoints <service>` — vazio significa discrepância no selector
2. `kubectl get pods --show-labels` — verifique se os labels correspondem ao selector do service
3. `kubectl describe ingress` — verifique regras de roteamento e configuração de TLS
4. `kubectl describe certificate` — verifique o status de provisionamento do cert-manager
5. `kubectl logs -n ingress-nginx` — logs do controller nginx para erros de roteamento
