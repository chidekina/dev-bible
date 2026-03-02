# Kubernetes Challenges

> Hands-on Kubernetes exercises covering workload manifests, networking, scaling, security, and debugging. Each exercise reflects a real production operations task. Tools assumed: kubectl, Helm 3, k9s (optional), a local cluster (minikube, kind, or k3d). Target skill level: beginner to intermediate.

---

## Exercise 1 — Deployment + Service Manifest for a Node.js App (Easy)

**Scenario:** Write Kubernetes manifests to deploy a Node.js API and expose it inside the cluster.

**Requirements:**
- `Deployment` with 3 replicas of `your-registry/node-api:1.0.0`.
- Container port: `3000`. Resource requests: `cpu: 100m, memory: 128Mi`. No limits set yet (will be added in Exercise 3).
- `Service` of type `ClusterIP` exposing port `80` targeting the container port `3000`.
- App label: `app: node-api`. Use `matchLabels` in the Deployment selector.
- Health check: a simple `exec` probe that runs `wget -qO- http://localhost:3000/health` (will be replaced in Exercise 2).

**Acceptance Criteria:**
- [ ] `kubectl apply -f deployment.yaml` creates 3 running pods.
- [ ] `kubectl get pods -l app=node-api` shows 3 pods with `Running` status.
- [ ] `kubectl exec -it <pod> -- wget -qO- http://node-api/health` returns `{"status":"ok"}` (via the ClusterIP Service).
- [ ] Pod spec includes `imagePullPolicy: Always` (prevents stale cached images).
- [ ] Manifests use `apiVersion: apps/v1` and `apiVersion: v1` correctly.

**Hints:**
1. Deployment selector must match pod template labels:
   ```yaml
   selector:
     matchLabels:
       app: node-api
   template:
     metadata:
       labels:
         app: node-api
   ```
2. Service `selector` must also match `app: node-api`.
3. ClusterIP Service: `type: ClusterIP` is the default — you can omit the `type` field.
4. Verify connectivity: `kubectl run tmp --image=busybox --restart=Never -it --rm -- wget -qO- http://node-api`.

---

## Exercise 2 — Liveness and Readiness Probes (Easy)

**Scenario:** Add HTTP health check probes to the Deployment from Exercise 1. The app exposes `GET /health` (liveness) and `GET /ready` (readiness).

**Requirements:**
- `livenessProbe`: HTTP GET `/health` on port `3000`. Initial delay: 10s, period: 15s, failure threshold: 3.
- `readinessProbe`: HTTP GET `/ready` on port `3000`. Initial delay: 5s, period: 10s, failure threshold: 2.
- `/ready` endpoint returns `503` when the app is still loading data from the DB.
- The liveness probe must not restart the pod during a slow startup — use `startupProbe` to handle that.
- `startupProbe`: HTTP GET `/health`, failure threshold: 30, period: 5s (allows up to 150s for startup).

**Acceptance Criteria:**
- [ ] A pod with a failing liveness probe is restarted (simulate by killing the health endpoint temporarily).
- [ ] Traffic is not routed to a pod failing the readiness probe (`kubectl get endpoints node-api` shows the pod removed).
- [ ] The startup probe gives the container 150 seconds to start before liveness kicks in.
- [ ] `kubectl describe pod <name>` shows all three probes configured correctly.
- [ ] Probes do not fire before the container has had time to start (initial delays are set appropriately).

**Hints:**
1. Startup probe disables liveness until it succeeds. Once it succeeds (first success), liveness takes over.
2. Readiness vs. liveness: readiness controls traffic routing; liveness controls pod restart. A pod can be alive but not ready.
3. `failureThreshold * periodSeconds` = maximum startup time before a restart. `30 * 5 = 150s`.
4. Test readiness: `kubectl exec <pod> -- kill -STOP 1` pauses the process → readiness fails → pod removed from endpoints.

---

## Exercise 3 — Resource Requests and Limits (Easy)

**Scenario:** Set resource requests and limits on the Node.js Deployment to ensure fair scheduling and prevent a noisy neighbor from consuming all node resources.

**Requirements:**
- `requests`: `cpu: 100m, memory: 128Mi`.
- `limits`: `cpu: 500m, memory: 256Mi`.
- Add a `LimitRange` to the `default` namespace that enforces a maximum of `cpu: 1, memory: 512Mi` per container.
- Add a `ResourceQuota` to the namespace: max 10 pods, max total `cpu: 4, memory: 2Gi`.
- Verify that trying to create a pod exceeding the `LimitRange` is rejected.

**Acceptance Criteria:**
- [ ] `kubectl describe pod <name>` shows the correct requests and limits.
- [ ] Attempting to create a pod with `limits.memory: 1Gi` is rejected with a `LimitRange` error.
- [ ] `kubectl describe resourcequota` shows current usage vs. hard limits.
- [ ] The QoS class of the pods is `Burstable` (requests < limits).
- [ ] A pod with equal requests and limits would have `Guaranteed` QoS — explain the difference in a comment.

**Hints:**
1. QoS classes: `Guaranteed` (requests == limits), `Burstable` (requests < limits or only limits set), `BestEffort` (no requests or limits). `Guaranteed` pods are the last to be evicted under memory pressure.
2. `LimitRange` applies per container. `ResourceQuota` applies per namespace.
3. `cpu: 100m` = 0.1 CPU cores. `cpu: 500m` = 0.5 CPU cores.
4. Test quota: create pods until the quota is exhausted — the 11th pod should be rejected.

---

## Exercise 4 — Deploy with Helm: Create a Chart from Scratch (Medium)

**Scenario:** Package the Node.js API as a Helm chart so it can be deployed to multiple environments (dev, staging, prod) with different configurations.

**Requirements:**
- Chart structure: `Chart.yaml`, `values.yaml`, `templates/deployment.yaml`, `templates/service.yaml`, `templates/ingress.yaml`.
- `values.yaml` must include: `replicaCount`, `image.repository`, `image.tag`, `service.port`, `ingress.enabled`, `ingress.host`, `resources`.
- The Ingress is only created if `ingress.enabled: true`.
- Prod overrides: `values-prod.yaml` with `replicaCount: 5` and `resources.limits.memory: 512Mi`.
- The chart must pass `helm lint` with no warnings.

**Acceptance Criteria:**
- [ ] `helm install node-api ./chart` deploys the app with default values.
- [ ] `helm install node-api ./chart -f values-prod.yaml` deploys with 5 replicas.
- [ ] `helm template ./chart | kubectl apply --dry-run=client -f -` succeeds without errors.
- [ ] `helm lint ./chart` returns "1 chart(s) linted, 0 chart(s) failed".
- [ ] Changing `image.tag` in `values.yaml` is the only change needed to deploy a new image version.

**Hints:**
1. `Chart.yaml` minimum: `apiVersion: v2`, `name: node-api`, `version: 0.1.0`, `appVersion: "1.0.0"`.
2. Conditional Ingress: `{{- if .Values.ingress.enabled }}` ... `{{- end }}`.
3. Image: `image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"`.
4. `helm upgrade --install` is idempotent — use it in CI instead of separate install/upgrade commands.

---

## Exercise 5 — Set Up Horizontal Pod Autoscaling (Medium)

**Scenario:** Configure the Node.js Deployment to automatically scale between 2 and 10 replicas based on CPU utilization.

**Requirements:**
- `HorizontalPodAutoscaler` targeting the `node-api` Deployment.
- Scale up when average CPU utilization exceeds 70% of requests.
- Scale down when CPU drops below 50% for more than 5 minutes (use `stabilizationWindowSeconds`).
- Minimum replicas: 2 (availability). Maximum: 10.
- Metrics server must be running in the cluster — install it with `kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml`.

**Acceptance Criteria:**
- [ ] `kubectl get hpa` shows the HPA with correct min/max replicas and current metrics.
- [ ] Simulating CPU load (`kubectl run load --image=busybox --command -- sh -c "while true; do wget -q -O- http://node-api/cpu-burn; done"`) triggers scale-up.
- [ ] After removing the load, the replica count decreases back to 2 within 10 minutes.
- [ ] `kubectl describe hpa node-api` shows the stabilization window configuration.
- [ ] Resource `requests.cpu` is set on the container (HPA requires requests to calculate utilization %).

**Hints:**
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
2. Scale-down stabilization: `behavior.scaleDown.stabilizationWindowSeconds: 300` (5 minutes).
3. Metrics server in minikube: `minikube addons enable metrics-server`.
4. Check current metrics: `kubectl top pods -l app=node-api`.

---

## Exercise 6 — Configure a NetworkPolicy (Medium)

**Scenario:** By default, all pods in a Kubernetes cluster can communicate with each other. Implement a default-deny policy and then selectively allow traffic.

**Requirements:**
- Create a `NetworkPolicy` that denies all ingress and egress traffic to all pods in the `default` namespace by default.
- Allow: the `node-api` pods to receive traffic from the `ingress-nginx` namespace on port 3000.
- Allow: the `node-api` pods to connect to the `postgres` pods on port 5432.
- Allow: all pods to reach the Kubernetes DNS service (UDP/TCP port 53).
- Deny: `node-api` connecting to any external IP (egress to the internet blocked).

**Acceptance Criteria:**
- [ ] After applying the default-deny policy, `kubectl exec` from one pod to another fails (connection refused).
- [ ] After the allow policy, `kubectl exec <node-api-pod> -- wget http://postgres:5432` connects (or is refused at the PostgreSQL level, not by NetworkPolicy).
- [ ] `kubectl exec <node-api-pod> -- wget http://example.com` times out (external egress blocked).
- [ ] DNS resolution still works inside pods: `nslookup kubernetes.default` succeeds.
- [ ] `kubectl describe networkpolicy` shows the correct pod and namespace selectors.

**Hints:**
1. Default-deny ingress: `spec.podSelector: {}` (matches all pods) + `spec.ingress: []` (no allowed ingress).
2. Default-deny egress: same structure with `spec.egress: []`.
3. Allow DNS: add an egress rule to allow UDP/TCP port 53 to the `kube-system` namespace or to `kube-dns` service CIDR.
4. NetworkPolicy requires a CNI plugin that supports it (Calico, Cilium, WeaveNet). Flannel does not support it.

---

## Exercise 7 — ConfigMap and Secret Injection (Easy)

**Scenario:** Externalize the Node.js app's configuration using a `ConfigMap` for non-sensitive settings and a `Secret` for database credentials.

**Requirements:**
- `ConfigMap`: `LOG_LEVEL=info`, `PORT=3000`, `NODE_ENV=production`.
- `Secret`: `DATABASE_URL=postgres://user:pass@db:5432/myapp`, `JWT_SECRET=supersecret`.
- Inject the `ConfigMap` as environment variables using `envFrom`.
- Inject `Secret` values as individual env vars using `valueFrom.secretKeyRef`.
- Mount the `ConfigMap` as a volume at `/app/config/app.conf` for a secondary config file format.

**Acceptance Criteria:**
- [ ] `kubectl exec <pod> -- env | grep LOG_LEVEL` returns `LOG_LEVEL=info`.
- [ ] `kubectl exec <pod> -- env | grep DATABASE_URL` returns the database URL.
- [ ] `kubectl get secret node-api-secrets -o jsonpath='{.data.JWT_SECRET}' | base64 -d` returns the secret value.
- [ ] The `ConfigMap` volume is mounted at `/app/config/app.conf` and readable inside the pod.
- [ ] Secrets are base64-encoded in the manifest — not stored as plain text in the YAML committed to git.

**Hints:**
1. Create secret from literal: `kubectl create secret generic node-api-secrets --from-literal=DATABASE_URL=... --from-literal=JWT_SECRET=...`. Then export to YAML with `kubectl get secret node-api-secrets -o yaml > secret.yaml` for the manifest.
2. Never commit plain-text secrets. Use `kubectl create secret` from a CI pipeline with values from a vault.
3. `envFrom: [configMapRef: {name: node-api-config}]` injects all ConfigMap keys as env vars.
4. Volume mount: `volumes: [{name: config, configMap: {name: node-api-config}}]` + `volumeMounts: [{name: config, mountPath: /app/config}]`.

---

## Exercise 8 — Roll Out a Zero-Downtime Update (Medium)

**Scenario:** Deploy a new version of the Node.js app (`image.tag: 1.1.0`) with zero downtime. Ensure rollback is fast if the new version is broken.

**Requirements:**
- Configure `RollingUpdate` strategy with `maxUnavailable: 0` and `maxSurge: 1`.
- The readiness probe must be passing before the rollout considers a pod ready.
- Set `minReadySeconds: 30` — new pods must be stable for 30 seconds before the next pod is replaced.
- Verify the rollout with `kubectl rollout status`.
- Simulate a bad deploy (image that fails readiness) and roll back with `kubectl rollout undo`.

**Acceptance Criteria:**
- [ ] `kubectl rollout status deployment/node-api` shows "successfully rolled out" after the update.
- [ ] During the rollout, at least 3 pods are always `Running` and serving traffic (verified with a load test running concurrently).
- [ ] A broken image (`image.tag: broken`) causes the rollout to stall — `kubectl rollout status` shows pending.
- [ ] `kubectl rollout undo deployment/node-api` restores the previous version within 60 seconds.
- [ ] `kubectl rollout history deployment/node-api` shows at least 2 revisions.

**Hints:**
1. `maxUnavailable: 0` means no pod is taken down until a new one is ready. `maxSurge: 1` allows one extra pod during the rollout.
2. `minReadySeconds: 30` prevents a flapping pod (passes readiness briefly then fails) from advancing the rollout.
3. To simulate a bad deploy: set the image to a non-existent tag. The pod stays in `ImagePullBackOff` and the rollout stalls.
4. Annotate the rollout: `kubectl annotate deployment node-api kubernetes.io/change-cause="Update to v1.1.0"` — this appears in `rollout history`.

---

## Exercise 9 — Set Up RBAC for a CI/CD Service Account (Medium)

**Scenario:** Your CI/CD pipeline needs to deploy updates to the `staging` namespace. Create a service account with the minimal permissions required — nothing more.

**Requirements:**
- Create a `ServiceAccount` named `cicd-deployer` in the `staging` namespace.
- Create a `Role` that allows: `get`, `list`, `create`, `update`, `patch` on `Deployments`, `Services`, and `ConfigMaps`.
- Create a `RoleBinding` that binds the role to the service account.
- The service account must NOT be able to delete resources or access other namespaces.
- Generate a kubeconfig for the service account to use in CI.

**Acceptance Criteria:**
- [ ] `kubectl auth can-i update deployments --as=system:serviceaccount:staging:cicd-deployer -n staging` returns `yes`.
- [ ] `kubectl auth can-i delete deployments --as=system:serviceaccount:staging:cicd-deployer -n staging` returns `no`.
- [ ] `kubectl auth can-i get pods --as=system:serviceaccount:staging:cicd-deployer -n production` returns `no` (no cross-namespace access).
- [ ] The kubeconfig uses a token from a `Secret` of type `kubernetes.io/service-account-token`.
- [ ] The `Role` uses the principle of least privilege — no wildcard (`*`) resource or verb.

**Hints:**
1. Service account token (Kubernetes 1.24+): manually create a `Secret` of type `kubernetes.io/service-account-token` referencing the service account — automatic mounting was deprecated.
2. `kubectl auth can-i` is your best friend for testing RBAC rules before applying them in production.
3. `Role` vs. `ClusterRole`: use `Role` for namespace-scoped permissions. `ClusterRole` is for cluster-wide or multi-namespace permissions.
4. Kubeconfig generation: extract the token from the secret, the CA cert from the cluster, and construct the kubeconfig YAML with `kubectl config set-*` commands.

---

## Exercise 10 — Debug a CrashLoopBackOff Pod (Medium)

**Scenario:** A pod is stuck in `CrashLoopBackOff`. Walk through the systematic debugging steps to identify and fix the root cause.

**Requirements:**
- Deploy the following intentionally broken manifest and debug it:
  ```yaml
  containers:
    - name: api
      image: node:20-alpine
      command: ["node", "dist/missing-file.js"]  # file does not exist
      env:
        - name: DATABASE_URL
          value: ""  # empty — app crashes on startup
  ```
- Document each debugging step you take.
- Fix the root cause — do not just mask it by changing the restart policy.
- After fixing, the pod should reach `Running` state and stay there for 5+ minutes.

**Acceptance Criteria:**
- [ ] Step 1: `kubectl get pods` identifies the `CrashLoopBackOff` status.
- [ ] Step 2: `kubectl describe pod <name>` reveals the exit code and last state.
- [ ] Step 3: `kubectl logs <name> --previous` shows the error from the previous (crashed) container.
- [ ] Step 4: root cause is identified from the logs (missing file + empty DATABASE_URL).
- [ ] Step 5: fixed manifest is applied and pod reaches `Running` with `0` restarts.

**Hints:**
1. Exit code 1 = application error (check logs). Exit code 137 = OOMKilled (increase memory limit). Exit code 139 = segfault (usually a native module issue).
2. `kubectl logs <pod> --previous` shows logs from the container's last run before it crashed.
3. `kubectl describe pod <name>` → `Last State` section shows the exit code and reason.
4. For a pod that crashes before your code even runs (e.g., wrong entry point): use `command: ["sleep", "infinity"]` temporarily to get a shell inside the container and debug: `kubectl exec -it <pod> -- sh`.
