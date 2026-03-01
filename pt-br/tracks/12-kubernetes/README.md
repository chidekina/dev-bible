# Track 12 — Kubernetes

Orquestre aplicações containerizadas em escala. Kubernetes é a plataforma padrão da indústria para fazer deploy, escalar e gerenciar cargas de trabalho em containers na produção.

**Tempo estimado:** 3–4 semanas

---

## Tópicos

1. [Conceitos de Kubernetes](kubernetes-concepts.md) — control plane, nodes, etcd, scheduler, o modelo declarativo
2. [kubectl Básico](kubectl-basics.md) — aplicando manifests, inspecionando recursos, exec, logs, port-forward
3. [Pods & Deployments](pods-deployments.md) — ciclo de vida do pod, ReplicaSets, rolling updates, limites de recursos
4. [Services & Ingress](services-ingress.md) — ClusterIP, NodePort, LoadBalancer, Ingress controllers, TLS termination
5. [ConfigMaps & Secrets](configmaps-secrets.md) — externalizar configuração, montar como env ou volume, sealed secrets
6. [Helm Básico](helm-basics.md) — charts, values, templating, releases, upgrade e rollback
7. [K8s em Produção](k8s-production.md) — HPA, PodDisruptionBudgets, readiness/liveness probes, resource quotas

---

## Pré-requisitos

- Track 05 — DevOps (Docker, CI/CD, redes)
- Track 08 — System Design (load balancing, conceitos de sistemas distribuídos)

---

## O que você vai construir

- Uma aplicação Node.js multi-container com deploy em um cluster k3s local usando Deployment, Service e Ingress
- Um Helm chart para a aplicação com réplicas configuráveis, image tag e variáveis de ambiente
- Um Horizontal Pod Autoscaler que escala o deployment com base na utilização de CPU
- Um manifest de deploy pronto para produção com readiness probes, limites de recursos e PodDisruptionBudget
