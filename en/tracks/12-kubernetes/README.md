# Track 12 — Kubernetes

Orchestrate containerized applications at scale. Kubernetes is the industry-standard platform for deploying, scaling, and managing container workloads in production.

**Estimated time:** 3–4 weeks

---

## Topics

1. [Kubernetes Concepts](kubernetes-concepts.md) — control plane, nodes, etcd, scheduler, the declarative model
2. [kubectl Basics](kubectl-basics.md) — applying manifests, inspecting resources, exec, logs, port-forward
3. [Pods & Deployments](pods-deployments.md) — pod lifecycle, ReplicaSets, rolling updates, resource limits
4. [Services & Ingress](services-ingress.md) — ClusterIP, NodePort, LoadBalancer, Ingress controllers, TLS termination
5. [ConfigMaps & Secrets](configmaps-secrets.md) — externalizing config, mounting as env or volume, sealed secrets
6. [Helm Basics](helm-basics.md) — charts, values, templating, releases, upgrading and rolling back
7. [K8s in Production](k8s-production.md) — HPA, PodDisruptionBudgets, readiness/liveness probes, resource quotas

---

## Prerequisites

- Track 05 — DevOps (Docker, CI/CD, networking)
- Track 08 — System Design (load balancing, distributed systems concepts)

---

## What You'll Build

- A multi-container Node.js application deployed to a local k3s cluster with a Deployment, Service, and Ingress
- A Helm chart for the application with configurable replicas, image tag, and environment variables
- A Horizontal Pod Autoscaler that scales the deployment based on CPU utilization
- A production-ready deployment manifest with readiness probes, resource limits, and a PodDisruptionBudget
