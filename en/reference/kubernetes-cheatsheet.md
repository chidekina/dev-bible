# Kubernetes Cheatsheet

> Quick reference — use Ctrl+F to find what you need.

---

## Context & Cluster

```bash
# kubectl config
kubectl config get-contexts                  # list all contexts
kubectl config current-context               # show active context
kubectl config use-context <name>            # switch context
kubectl config set-context --current --namespace=myns  # set default namespace
kubectl config view                          # show full kubeconfig
kubectl config delete-context <name>         # remove context

# kubectx / kubens (install: brew install kubectx)
kubectx                    # list contexts
kubectx <name>             # switch context
kubectx -                  # switch to previous context
kubens                     # list namespaces
kubens <name>              # switch namespace
kubens -                   # switch to previous namespace

# Cluster info
kubectl cluster-info
kubectl get nodes
kubectl get nodes -o wide
kubectl describe node <name>
kubectl top node                             # CPU/memory usage (metrics-server required)
```

---

## Pod Commands

```bash
# List & inspect
kubectl get pods                             # current namespace
kubectl get pods -A                          # all namespaces
kubectl get pods -n <namespace>
kubectl get pods -o wide                     # show node & IP
kubectl get pod <name> -o yaml               # full spec
kubectl describe pod <name>                  # events + status

# Logs
kubectl logs <pod>                           # stdout
kubectl logs <pod> -f                        # follow
kubectl logs <pod> --tail=100
kubectl logs <pod> -c <container>            # multi-container pod
kubectl logs <pod> --previous               # previous container instance

# Exec
kubectl exec -it <pod> -- bash
kubectl exec -it <pod> -c <container> -- sh
kubectl exec <pod> -- env                    # non-interactive

# Delete
kubectl delete pod <name>
kubectl delete pod <name> --grace-period=0   # immediate
kubectl delete pods --field-selector=status.phase=Failed
```

---

## Deployment Commands

```bash
# Apply & create
kubectl apply -f deployment.yaml
kubectl apply -f ./manifests/               # apply directory
kubectl apply -k ./overlays/prod/           # kustomize
kubectl create deployment nginx --image=nginx:alpine

# Inspect
kubectl get deployments
kubectl describe deployment <name>
kubectl get deployment <name> -o yaml

# Scale
kubectl scale deployment <name> --replicas=5
kubectl autoscale deployment <name> --min=2 --max=10 --cpu-percent=80

# Rollout
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout history deployment/<name> --revision=3
kubectl rollout undo deployment/<name>               # rollback to previous
kubectl rollout undo deployment/<name> --to-revision=2
kubectl rollout restart deployment/<name>            # rolling restart

# Edit & patch
kubectl edit deployment <name>
kubectl set image deployment/<name> app=nginx:1.25
kubectl patch deployment <name> -p '{"spec":{"replicas":3}}'
```

---

## Service & Ingress

```bash
# Services
kubectl get svc
kubectl get svc -A
kubectl describe svc <name>
kubectl expose deployment <name> --port=80 --target-port=3000 --type=ClusterIP
kubectl expose deployment <name> --type=LoadBalancer --port=80

# Service types
# ClusterIP  — internal only (default)
# NodePort   — expose on node IP + random port (30000-32767)
# LoadBalancer — cloud provider LB
# ExternalName — DNS alias

# Ingress
kubectl get ingress
kubectl describe ingress <name>
kubectl apply -f ingress.yaml
```

```yaml
# ingress.yaml example
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  number: 80
```

---

## ConfigMap & Secret

```bash
# ConfigMap
kubectl create configmap app-config --from-literal=KEY=value
kubectl create configmap app-config --from-file=config.json
kubectl create configmap app-config --from-env-file=.env
kubectl get configmap app-config -o yaml
kubectl edit configmap app-config
kubectl delete configmap app-config

# Secret
kubectl create secret generic db-secret --from-literal=password=s3cr3t
kubectl create secret generic tls-secret --from-file=tls.crt --from-file=tls.key
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d
```

```yaml
# Using configmap in pod
envFrom:
  - configMapRef:
      name: app-config

# Using secret as env
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password

# Mounting secret as volume
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

## Namespace Operations

```bash
kubectl get namespaces
kubectl create namespace staging
kubectl delete namespace staging
kubectl apply -f manifest.yaml -n staging

# Resource quota for namespace
kubectl create quota my-quota \
  --hard=cpu=4,memory=8Gi,pods=20 \
  -n staging
kubectl describe quota -n staging
```

---

## Troubleshooting

```bash
# Events (most useful for debugging)
kubectl get events --sort-by='.lastTimestamp'
kubectl get events -n <namespace> --field-selector=type=Warning

# Resource usage (requires metrics-server)
kubectl top pods
kubectl top pods -A --sort-by=memory
kubectl top node

# Port forward
kubectl port-forward pod/<name> 8080:3000
kubectl port-forward svc/<name> 8080:80
kubectl port-forward deployment/<name> 8080:3000

# Copy files
kubectl cp <pod>:/app/logs/app.log ./app.log
kubectl cp ./config.json <pod>:/app/config.json

# Debug with ephemeral container (k8s 1.23+)
kubectl debug -it <pod> --image=busybox --target=app

# Common crash reasons
kubectl describe pod <name>   # look at: Events, State, Last State, Exit Code
# Exit 0  = clean exit        Exit 1  = app error
# Exit 137 = OOMKilled         Exit 143 = SIGTERM (graceful termination)
```

---

## RBAC

```bash
# ServiceAccount
kubectl create serviceaccount my-sa -n myns
kubectl get serviceaccounts

# Role (namespace-scoped)
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  -n myns

# ClusterRole (cluster-scoped)
kubectl create clusterrole pod-reader \
  --verb=get,list,watch \
  --resource=pods

# RoleBinding
kubectl create rolebinding sa-pod-reader \
  --role=pod-reader \
  --serviceaccount=myns:my-sa \
  -n myns

# ClusterRoleBinding
kubectl create clusterrolebinding sa-pod-reader \
  --clusterrole=pod-reader \
  --serviceaccount=myns:my-sa

# Check permissions
kubectl auth can-i list pods --as=system:serviceaccount:myns:my-sa
kubectl auth can-i --list -n myns
```

---

## Helm Commands

```bash
# Repos
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm repo list
helm repo remove bitnami
helm search repo nginx
helm search hub nginx                        # search Artifact Hub

# Install
helm install my-release bitnami/nginx
helm install my-release bitnami/nginx -n myns --create-namespace
helm install my-release ./mychart
helm install my-release bitnami/nginx \
  --set service.type=LoadBalancer \
  --values custom-values.yaml
helm install my-release bitnami/nginx --dry-run --debug

# Upgrade
helm upgrade my-release bitnami/nginx
helm upgrade --install my-release bitnami/nginx  # install if not exists
helm upgrade my-release bitnami/nginx --set image.tag=1.25

# Rollback
helm rollback my-release 1                   # rollback to revision 1
helm history my-release                      # list revisions

# Manage
helm list
helm list -A                                 # all namespaces
helm status my-release
helm uninstall my-release
helm get values my-release                   # show applied values
helm get manifest my-release                 # show rendered manifests
```

---

## Useful One-Liners & Aliases

```bash
# Delete all pods in CrashLoopBackOff
kubectl get pods -A | grep CrashLoopBackOff | awk '{print $1, $2}' | \
  xargs -n2 kubectl delete pod -n

# Watch pods in real time
watch -n 2 kubectl get pods -A

# Get all images running in cluster
kubectl get pods -A -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}' | sort -u

# Force delete stuck terminating pod
kubectl delete pod <name> --grace-period=0 --force

# Decode all secret values
kubectl get secret <name> -o json | jq '.data | map_values(@base64d)'

# List all resources in a namespace
kubectl api-resources --verbs=list --namespaced -o name | \
  xargs -I {} kubectl get {} -n myns --ignore-not-found

# Aliases (add to ~/.aliases or ~/.zshrc)
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
# Minimal Deployment template
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: myapp:1.0
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
