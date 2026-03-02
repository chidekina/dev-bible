# Kubernetes Cheatsheet

> Referência rápida — use Ctrl+F para encontrar o que precisa.

---

## Contexto e Cluster

```bash
# kubectl config
kubectl config get-contexts                  # listar todos os contextos
kubectl config current-context               # mostrar contexto ativo
kubectl config use-context <nome>            # trocar contexto
kubectl config set-context --current --namespace=meuns  # definir namespace padrão
kubectl config view                          # exibir kubeconfig completo
kubectl config delete-context <nome>         # remover contexto

# kubectx / kubens (instalar: brew install kubectx)
kubectx                    # listar contextos
kubectx <nome>             # trocar contexto
kubectx -                  # voltar ao contexto anterior
kubens                     # listar namespaces
kubens <nome>              # trocar namespace
kubens -                   # voltar ao namespace anterior

# Informações do cluster
kubectl cluster-info
kubectl get nodes
kubectl get nodes -o wide
kubectl describe node <nome>
kubectl top node                             # uso de CPU/memória (requer metrics-server)
```

---

## Comandos de Pod

```bash
# Listar e inspecionar
kubectl get pods                             # namespace atual
kubectl get pods -A                          # todos os namespaces
kubectl get pods -n <namespace>
kubectl get pods -o wide                     # mostrar nó e IP
kubectl get pod <nome> -o yaml               # spec completo
kubectl describe pod <nome>                  # eventos + status

# Logs
kubectl logs <pod>                           # stdout
kubectl logs <pod> -f                        # acompanhar em tempo real
kubectl logs <pod> --tail=100
kubectl logs <pod> -c <container>            # pod com múltiplos containers
kubectl logs <pod> --previous               # instância anterior do container

# Exec
kubectl exec -it <pod> -- bash
kubectl exec -it <pod> -c <container> -- sh
kubectl exec <pod> -- env                    # não-interativo

# Deletar
kubectl delete pod <nome>
kubectl delete pod <nome> --grace-period=0   # imediato
kubectl delete pods --field-selector=status.phase=Failed
```

---

## Comandos de Deployment

```bash
# Aplicar e criar
kubectl apply -f deployment.yaml
kubectl apply -f ./manifests/               # aplicar diretório
kubectl apply -k ./overlays/prod/           # kustomize
kubectl create deployment nginx --image=nginx:alpine

# Inspecionar
kubectl get deployments
kubectl describe deployment <nome>
kubectl get deployment <nome> -o yaml

# Escalar
kubectl scale deployment <nome> --replicas=5
kubectl autoscale deployment <nome> --min=2 --max=10 --cpu-percent=80

# Rollout
kubectl rollout status deployment/<nome>
kubectl rollout history deployment/<nome>
kubectl rollout history deployment/<nome> --revision=3
kubectl rollout undo deployment/<nome>               # rollback para versão anterior
kubectl rollout undo deployment/<nome> --to-revision=2
kubectl rollout restart deployment/<nome>            # reinício gradual

# Editar e aplicar patch
kubectl edit deployment <nome>
kubectl set image deployment/<nome> app=nginx:1.25
kubectl patch deployment <nome> -p '{"spec":{"replicas":3}}'
```

---

## Service e Ingress

```bash
# Services
kubectl get svc
kubectl get svc -A
kubectl describe svc <nome>
kubectl expose deployment <nome> --port=80 --target-port=3000 --type=ClusterIP
kubectl expose deployment <nome> --type=LoadBalancer --port=80

# Tipos de service
# ClusterIP    — apenas interno (padrão)
# NodePort     — expõe no IP do nó + porta aleatória (30000-32767)
# LoadBalancer — load balancer do provedor cloud
# ExternalName — alias DNS

# Ingress
kubectl get ingress
kubectl describe ingress <nome>
kubectl apply -f ingress.yaml
```

```yaml
# Exemplo de ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: meu-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: app.exemplo.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: meu-service
                port:
                  number: 80
```

---

## ConfigMap e Secret

```bash
# ConfigMap
kubectl create configmap app-config --from-literal=CHAVE=valor
kubectl create configmap app-config --from-file=config.json
kubectl create configmap app-config --from-env-file=.env
kubectl get configmap app-config -o yaml
kubectl edit configmap app-config
kubectl delete configmap app-config

# Secret
kubectl create secret generic db-secret --from-literal=password=s3cr3t
kubectl create secret generic tls-secret --from-file=tls.crt --from-file=tls.key
kubectl create secret docker-registry regcred \
  --docker-server=registry.exemplo.com \
  --docker-username=usuario \
  --docker-password=senha
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d
```

```yaml
# Usando configmap no pod
envFrom:
  - configMapRef:
      name: app-config

# Usando secret como variável de ambiente
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password

# Montando secret como volume
volumes:
  - name: certs
    secret:
      secretName: tls-secret
volumeMounts:
  - name: certs
    mountPath: /etc/ssl
    readOnly: true
```

---

## Operações com Namespace

```bash
kubectl get namespaces
kubectl create namespace staging
kubectl delete namespace staging
kubectl apply -f manifest.yaml -n staging

# Cota de recursos para namespace
kubectl create quota minha-cota \
  --hard=cpu=4,memory=8Gi,pods=20 \
  -n staging
kubectl describe quota -n staging
```

---

## Resolução de Problemas

```bash
# Eventos (mais útil para debug)
kubectl get events --sort-by='.lastTimestamp'
kubectl get events -n <namespace> --field-selector=type=Warning

# Uso de recursos (requer metrics-server)
kubectl top pods
kubectl top pods -A --sort-by=memory
kubectl top node

# Port forward
kubectl port-forward pod/<nome> 8080:3000
kubectl port-forward svc/<nome> 8080:80
kubectl port-forward deployment/<nome> 8080:3000

# Copiar arquivos
kubectl cp <pod>:/app/logs/app.log ./app.log
kubectl cp ./config.json <pod>:/app/config.json

# Debug com container efêmero (k8s 1.23+)
kubectl debug -it <pod> --image=busybox --target=app

# Causas comuns de crash
kubectl describe pod <nome>   # ver: Events, State, Last State, Exit Code
# Exit 0   = saída limpa       Exit 1   = erro na aplicação
# Exit 137 = OOMKilled          Exit 143 = SIGTERM (encerramento gracioso)
```

---

## RBAC

```bash
# ServiceAccount
kubectl create serviceaccount minha-sa -n meuns
kubectl get serviceaccounts

# Role (escopo de namespace)
kubectl create role leitor-pods \
  --verb=get,list,watch \
  --resource=pods \
  -n meuns

# ClusterRole (escopo de cluster)
kubectl create clusterrole leitor-pods \
  --verb=get,list,watch \
  --resource=pods

# RoleBinding
kubectl create rolebinding sa-leitor-pods \
  --role=leitor-pods \
  --serviceaccount=meuns:minha-sa \
  -n meuns

# ClusterRoleBinding
kubectl create clusterrolebinding sa-leitor-pods \
  --clusterrole=leitor-pods \
  --serviceaccount=meuns:minha-sa

# Verificar permissões
kubectl auth can-i list pods --as=system:serviceaccount:meuns:minha-sa
kubectl auth can-i --list -n meuns
```

---

## Comandos do Helm

```bash
# Repositórios
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm repo list
helm repo remove bitnami
helm search repo nginx
helm search hub nginx                        # buscar no Artifact Hub

# Instalar
helm install meu-release bitnami/nginx
helm install meu-release bitnami/nginx -n meuns --create-namespace
helm install meu-release ./meuchart
helm install meu-release bitnami/nginx \
  --set service.type=LoadBalancer \
  --values custom-values.yaml
helm install meu-release bitnami/nginx --dry-run --debug

# Atualizar
helm upgrade meu-release bitnami/nginx
helm upgrade --install meu-release bitnami/nginx  # instala se não existir
helm upgrade meu-release bitnami/nginx --set image.tag=1.25

# Rollback
helm rollback meu-release 1                   # voltar para revisão 1
helm history meu-release                      # listar revisões

# Gerenciar
helm list
helm list -A                                 # todos os namespaces
helm status meu-release
helm uninstall meu-release
helm get values meu-release                   # valores aplicados
helm get manifest meu-release                 # manifests renderizados
```

---

## One-Liners Úteis e Aliases

```bash
# Deletar todos os pods em CrashLoopBackOff
kubectl get pods -A | grep CrashLoopBackOff | awk '{print $1, $2}' | \
  xargs -n2 kubectl delete pod -n

# Monitorar pods em tempo real
watch -n 2 kubectl get pods -A

# Listar todas as imagens rodando no cluster
kubectl get pods -A -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}' | sort -u

# Forçar deleção de pod preso em Terminating
kubectl delete pod <nome> --grace-period=0 --force

# Decodificar todos os valores de um secret
kubectl get secret <nome> -o json | jq '.data | map_values(@base64d)'

# Listar todos os recursos em um namespace
kubectl api-resources --verbs=list --namespaced -o name | \
  xargs -I {} kubectl get {} -n meuns --ignore-not-found

# Aliases (adicionar em ~/.aliases ou ~/.zshrc)
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgpa='kubectl get pods -A'
alias kgs='kubectl get svc'
alias kgd='kubectl get deployments'
alias kdp='kubectl describe pod'
alias kl='kubectl logs -f'
alias kx='kubectl exec -it'
```

```yaml
# Template mínimo de Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minha-app
  labels:
    app: minha-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: minha-app
  template:
    metadata:
      labels:
        app: minha-app
    spec:
      containers:
        - name: app
          image: minhaapp:1.0
          ports:
            - containerPort: 3000
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
```
