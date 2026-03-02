# DevOps Labs

> Hands-on labs to build real DevOps skills. Each lab has a clear goal, prerequisites, step-by-step tasks, and verification criteria. You will work with Docker, GitHub Actions, Nginx, Linux system administration, and monitoring tools. All labs assume a Linux/WSL2 environment. Complete each lab end-to-end — do not skip verification steps.

---

## Lab 1 — Dockerize a Node.js Application (Beginner)

**Goal:** Package a Node.js/TypeScript application into a production-ready Docker image using a multi-stage build. The final image must be minimal and secure.

**Prerequisites:**
- Docker Desktop installed and running.
- A Node.js/TypeScript project (use the sample app in the lab's `samples/node-app/` directory or your own).
- Basic understanding of what a container is.

**Tasks:**

1. Write a `Dockerfile` in the project root:
   - Stage 1 (`builder`): use `node:20-alpine`, copy `package.json` and `package-lock.json`, run `npm ci`, copy source, run `npm run build`.
   - Stage 2 (`runtime`): use `node:20-alpine`, copy only the built output (`dist/`) and production `node_modules` from the builder stage.
   - Set a non-root user (`USER node`) in the runtime stage.
   - `EXPOSE 3000` and set `CMD ["node", "dist/index.js"]`.

2. Build the image: `docker build -t myapp:local .`

3. Run the container: `docker run -p 3000:3000 --env-file .env myapp:local`

4. Write a `.dockerignore` file that excludes: `node_modules/`, `dist/`, `.git/`, `.env`, `*.log`.

5. Analyze the image size: `docker image inspect myapp:local | grep Size`. Compare to a single-stage build.

6. Add a Docker health check instruction: `HEALTHCHECK --interval=30s --timeout=5s CMD wget -qO- http://localhost:3000/health || exit 1`

**Verification:**
- [ ] `docker ps` shows the container running.
- [ ] `curl http://localhost:3000/health` returns HTTP 200.
- [ ] `docker image ls myapp:local` shows size < 150MB (for a typical TypeScript app).
- [ ] Running `docker exec <container_id> whoami` returns `node` (non-root).
- [ ] The `.env` file is not baked into the image — verify with `docker run myapp:local printenv` (secrets should not appear).
- [ ] `docker inspect myapp:local` shows the health check configured.

**Stretch goal:** Push the image to GitHub Container Registry (`ghcr.io`) using a Personal Access Token.

---

## Lab 2 — Docker Compose for a Full-Stack Application (Beginner)

**Goal:** Use Docker Compose to orchestrate a multi-service application: Node.js API, PostgreSQL database, and Redis cache — all running locally with a single command.

**Prerequisites:**
- Docker Desktop with Compose v2 (`docker compose version`).
- Lab 1 completed (you have a working Dockerfile).

**Tasks:**

1. Create `docker-compose.yml` with three services:
   - `api`: builds from `./Dockerfile`, depends on `db` and `redis`, maps port `3000:3000`, passes environment variables from a `.env` file.
   - `db`: uses `postgres:16-alpine`, sets `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` from env, mounts a named volume `pgdata:/var/lib/postgresql/data`.
   - `redis`: uses `redis:7-alpine`, mounts a named volume `redisdata:/data`, uses `redis-server --appendonly yes` for persistence.

2. Add health checks to `db` and `redis` services. Make the `api` service use `depends_on` with `condition: service_healthy`.

3. Start the stack: `docker compose up -d`

4. Run database migrations inside the running `api` container: `docker compose exec api npm run db:migrate`

5. Add a `docker-compose.override.yml` for local development:
   - Mount the source code as a volume for hot reload: `./src:/app/src`.
   - Override the command to use `npm run dev`.

6. Tear down: `docker compose down -v` (removes volumes too — note the difference from `down` without `-v`).

**Verification:**
- [ ] `docker compose ps` shows all three services as `running` (not `starting` or `exiting`).
- [ ] `docker compose logs api` shows successful DB connection.
- [ ] `curl http://localhost:3000/health` returns `{ "status": "ok", "db": "connected", "redis": "connected" }`.
- [ ] After `docker compose down` and `docker compose up`, database data persists (volumes were not removed).
- [ ] The override file hot-reloads on source file changes when using `docker compose -f docker-compose.yml -f docker-compose.override.yml up`.

---

## Lab 3 — CI/CD Pipeline with GitHub Actions (Intermediate)

**Goal:** Build a complete CI/CD pipeline that runs tests, builds a Docker image, and deploys to a remote server on every push to `main`.

**Prerequisites:**
- A GitHub repository with the Node.js application.
- A remote Linux server (a free-tier VPS or a local VM reachable via SSH).
- Docker installed on the remote server.
- GitHub repository secrets configured.

**Tasks:**

1. Create `.github/workflows/ci.yml` — the CI job:
   - Triggers on: `push` to `main` and `pull_request` to `main`.
   - Steps: checkout, setup Node.js 20, `npm ci`, `npm run lint`, `npm run test`, `npm run build`.
   - Cache: use `actions/cache` to cache `node_modules` based on `package-lock.json` hash.

2. Create `.github/workflows/deploy.yml` — the deploy job (runs only on push to `main`, after CI passes):
   - Build and push the Docker image to `ghcr.io/<your-username>/<app>:latest`.
   - SSH into the remote server using `appleboy/ssh-action`.
   - On the server: pull the new image, stop the old container, start the new one.

3. Add required GitHub Secrets: `SSH_HOST`, `SSH_USER`, `SSH_PRIVATE_KEY`, `GHCR_TOKEN`.

4. Add a status badge to `README.md` showing the CI status.

5. Introduce a deliberate test failure to verify the pipeline blocks the deploy.

6. (Stretch) Add a smoke test step after deploy: `curl https://<your-domain>/health` and fail the job if it returns non-200.

**Verification:**
- [ ] Opening a PR triggers the CI job. The PR cannot be merged if CI fails (branch protection rule).
- [ ] Merging to `main` triggers the deploy job.
- [ ] The deploy job completes and the new version is live on the remote server within 3 minutes.
- [ ] A failing test (`expect(1).toBe(2)`) in a PR blocks the pipeline — no deploy occurs.
- [ ] `docker ps` on the remote server shows the new container image tag.
- [ ] GitHub Actions logs show the `node_modules` cache being restored (not re-downloaded) on the second run.

---

## Lab 4 — Nginx as a Reverse Proxy (Intermediate)

**Goal:** Configure Nginx to reverse proxy multiple backend services, handle SSL termination with Let's Encrypt, and serve static assets efficiently.

**Prerequisites:**
- A Linux server with a public IP address.
- A domain name pointing to that IP.
- Nginx installed (`sudo apt install nginx`).
- `certbot` installed (`sudo snap install certbot --classic`).

**Tasks:**

1. Create an Nginx server block at `/etc/nginx/sites-available/myapp`:
   - Listen on port 80. Redirect all HTTP traffic to HTTPS.
   - Listen on port 443 with SSL (certificates will be added by Certbot).
   - Proxy `/api/` to `http://localhost:3000/` with headers: `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`.
   - Serve static files from `/var/www/myapp/` directly (for a frontend build).
   - Set `gzip on` with `gzip_types text/plain text/css application/json application/javascript`.

2. Enable the site: `sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/`

3. Test the config: `sudo nginx -t`

4. Obtain an SSL certificate: `sudo certbot --nginx -d yourdomain.com`

5. Configure Nginx for security headers:
   - `add_header X-Frame-Options "SAMEORIGIN";`
   - `add_header X-Content-Type-Options "nosniff";`
   - `add_header Referrer-Policy "strict-origin-when-cross-origin";`
   - `add_header Content-Security-Policy "default-src 'self';";`

6. Configure rate limiting in Nginx:
   - `limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;`
   - Apply `limit_req zone=api burst=20 nodelay;` to the `/api/` location block.

7. Set up automatic certificate renewal: `sudo certbot renew --dry-run`

**Verification:**
- [ ] `curl -I http://yourdomain.com` returns `301 Moved Permanently` to HTTPS.
- [ ] `curl https://yourdomain.com/api/health` returns the API response.
- [ ] `curl https://yourdomain.com/index.html` serves the static file.
- [ ] SSL Labs test (ssllabs.com/ssltest) gives an A or A+ rating.
- [ ] Security headers are present in the response (`curl -I https://yourdomain.com`).
- [ ] Sending 30 rapid requests to `/api/` results in `429 Too Many Requests` responses for the excess.
- [ ] `sudo crontab -l` or `systemctl list-timers` shows Certbot auto-renewal scheduled.

---

## Lab 5 — Harden a Linux VPS (Intermediate)

**Goal:** Apply security hardening to a freshly provisioned Ubuntu VPS to reduce the attack surface before deploying any application.

**Prerequisites:**
- A fresh Ubuntu 22.04 LTS VPS with root access.
- A local SSH key pair.

**Tasks:**

1. **Create a non-root user:**
   ```bash
   adduser deploy
   usermod -aG sudo deploy
   ```

2. **SSH hardening** — edit `/etc/ssh/sshd_config`:
   - `PermitRootLogin no`
   - `PasswordAuthentication no`
   - `PubkeyAuthentication yes`
   - `Port 2222` (change from default 22)
   - Copy your public key: `ssh-copy-id -i ~/.ssh/id_ed25519.pub -p 22 deploy@<ip>`

3. **Configure UFW (Uncomplicated Firewall):**
   ```bash
   ufw default deny incoming
   ufw default allow outgoing
   ufw allow 2222/tcp   # new SSH port
   ufw allow 80/tcp
   ufw allow 443/tcp
   ufw enable
   ```

4. **Install and configure Fail2Ban:**
   - Install: `sudo apt install fail2ban`
   - Create `/etc/fail2ban/jail.local` with SSH jail: maxretry 3, bantime 1h, findtime 10m.
   - Start and enable: `sudo systemctl enable --now fail2ban`

5. **Automatic security updates:**
   ```bash
   sudo apt install unattended-upgrades
   sudo dpkg-reconfigure --priority=low unattended-upgrades
   ```

6. **System hardening — `/etc/sysctl.conf` additions:**
   ```
   net.ipv4.conf.all.rp_filter = 1
   net.ipv4.conf.all.accept_source_route = 0
   net.ipv4.tcp_syncookies = 1
   kernel.dmesg_restrict = 1
   ```
   Apply: `sudo sysctl -p`

7. **Audit installed packages:** `sudo apt list --installed | wc -l`. Remove obviously unneeded packages.

**Verification:**
- [ ] `ssh root@<ip> -p 2222` is rejected: "Permission denied".
- [ ] `ssh deploy@<ip> -p 2222` with the private key works.
- [ ] `ssh deploy@<ip> -p 22` times out (port 22 is closed by UFW).
- [ ] `sudo ufw status verbose` shows only ports 2222, 80, 443 open.
- [ ] `sudo fail2ban-client status sshd` shows the jail is active.
- [ ] `sudo apt list --upgradable` returns empty or only non-security updates.
- [ ] Running a quick port scan with `nmap -sV <ip>` from outside shows minimal open ports.

---

## Lab 6 — Set Up Monitoring with Prometheus and Grafana (Intermediate)

**Goal:** Deploy a monitoring stack using Docker Compose. Instrument a Node.js application, collect metrics with Prometheus, and visualize them in Grafana.

**Prerequisites:**
- Docker Compose installed.
- The Node.js application from Lab 1 running locally.
- Basic understanding of time-series metrics concepts.

**Tasks:**

1. Add Prometheus client to the Node.js app:
   ```bash
   npm install prom-client
   ```
   Expose a `/metrics` endpoint that returns Prometheus-format metrics. Instrument:
   - HTTP request rate (counter by method, route, status code).
   - HTTP request duration histogram (P50, P95, P99).
   - Active connections gauge.
   - Node.js default metrics (CPU, memory, event loop lag) via `collectDefaultMetrics()`.

2. Create `monitoring/prometheus.yml`:
   ```yaml
   scrape_configs:
     - job_name: 'nodejs-app'
       static_configs:
         - targets: ['app:3000']
       metrics_path: '/metrics'
       scrape_interval: 15s
   ```

3. Add `prometheus` and `grafana` services to `docker-compose.yml`:
   - Prometheus: `prom/prometheus:latest`, mount the config, port `9090:9090`.
   - Grafana: `grafana/grafana:latest`, port `3001:3000`, volume for persistence, env vars for default admin credentials.

4. Import the **Node.js Application Dashboard** (Grafana dashboard ID `11159`) via the Grafana UI.

5. Create a custom Grafana dashboard with panels:
   - Request rate (req/sec) as a graph.
   - Error rate (5xx/min) as a stat panel with a red threshold.
   - P99 latency as a gauge.
   - Memory usage (RSS) as a time-series graph.

6. Set up a Grafana alert: if error rate exceeds 5 errors/min for more than 2 minutes, trigger an alert (use the built-in email or webhook alerting — no external service required for the lab).

**Verification:**
- [ ] `curl http://localhost:3000/metrics` returns Prometheus-formatted text.
- [ ] `http://localhost:9090/targets` shows the Node.js app as `UP`.
- [ ] The imported Grafana dashboard shows data for all panels.
- [ ] Generating load (`ab -n 1000 -c 10 http://localhost:3000/`) shows a spike in the request rate panel.
- [ ] Introducing a deliberate 500 error triggers the Grafana alert within the configured window.
- [ ] Grafana persists dashboards after `docker compose restart grafana`.

---

## Lab 7 — Zero-Downtime Deployment Strategy (Intermediate)

**Goal:** Implement a blue-green deployment for the Node.js application using Nginx and Docker Compose, achieving zero-downtime updates.

**Prerequisites:**
- Labs 1, 2, and 4 completed.
- Nginx configured as a reverse proxy.

**Tasks:**

1. Run two versions of the app simultaneously:
   - `blue` container: current production version on port `3001`.
   - `green` container: new version on port `3002`.

2. Nginx upstream config:
   ```nginx
   upstream app {
     server localhost:3001;  # active: blue
   }
   ```

3. Write a deploy script `scripts/deploy.sh`:
   - Pull the new Docker image.
   - Start the `green` container on port 3002.
   - Wait for the health check to pass (`curl http://localhost:3002/health`).
   - Update the Nginx upstream to point to port 3002.
   - Reload Nginx (`nginx -s reload` — zero-downtime reload, no restart).
   - Stop the old `blue` container.
   - Rename `green` to `blue` for the next deploy.

4. Test with a continuous load: `while true; do curl -s http://localhost/health; sleep 0.1; done`
   Run the deploy script while this loop is active. Zero requests should fail.

5. Implement a rollback mechanism in the script: if the health check fails, remove the `green` container and keep `blue` running.

**Verification:**
- [ ] During deployment, zero HTTP 502 or 500 errors are observed in the load-test output.
- [ ] Nginx error log shows no errors during the config reload.
- [ ] After deployment, `docker ps` shows only one app container running.
- [ ] Deliberately breaking the new version (returning 500 on `/health`) causes the rollback to trigger automatically.
- [ ] The total deployment time (start script → traffic switched) is under 60 seconds.

---

## Lab 8 — Structured Logging Pipeline with Loki and Grafana (Intermediate)

**Goal:** Set up a centralized log aggregation system using Grafana Loki, Promtail, and Grafana. Ship application and system logs and query them in the Grafana UI.

**Prerequisites:**
- Docker Compose and the monitoring stack from Lab 6.
- The Node.js application logging in JSON format (Pino).

**Tasks:**

1. Add `loki` and `promtail` services to `docker-compose.yml`:
   - Loki: `grafana/loki:latest`, port `3100:3100`, persist data with a volume.
   - Promtail: `grafana/promtail:latest`, mount `/var/log` and the Docker socket.

2. Configure `monitoring/promtail.yml`:
   - Scrape Docker container logs from `/var/log/docker/containers/**/*.log`.
   - Add labels: `job`, `container_name`, `stream` (stdout/stderr).
   - Pipeline stage: parse JSON logs, extract `level`, `requestId`, `statusCode` as labels.

3. Add Loki as a data source in Grafana (`http://loki:3100`).

4. Write LogQL queries in the Grafana Explore view:
   - All error logs in the last hour: `{job="app"} |= "error"`
   - Requests with status 500: `{job="app"} | json | statusCode = "500"`
   - Request rate by route: `sum by (url) (rate({job="app"}[5m]))`

5. Create a Logs panel in your Grafana dashboard showing real-time application logs.

6. Set up a log-based alert: if the string `"CRITICAL"` appears in any log within 1 minute, fire an alert.

**Verification:**
- [ ] `curl http://localhost:3100/ready` returns `ready`.
- [ ] Grafana Explore with Loki data source shows logs from the Node.js container.
- [ ] Querying `{job="app"} |= "error"` returns error-level log entries.
- [ ] Generating a 500 error on the app produces a log entry visible in Grafana within 15 seconds.
- [ ] The JSON fields (`level`, `requestId`) are properly parsed and available as labels for filtering.

---

## Lab 9 — Infrastructure as Code with Terraform (Advanced)

**Goal:** Provision a complete application infrastructure on a cloud provider using Terraform. The infrastructure includes a VPS/VM, managed database, and DNS record — all defined as code.

**Prerequisites:**
- Terraform CLI installed (`terraform version`).
- A cloud provider account (DigitalOcean or AWS free tier recommended).
- API token/credentials configured.

**Tasks:**

1. Create `infra/main.tf`:
   - Provider: `digitalocean` (or `aws`).
   - Resources: `digitalocean_droplet` (2GB RAM, `ubuntu-22-04`), `digitalocean_database_cluster` (PostgreSQL 15, 1 node, smallest plan), `digitalocean_domain` and `digitalocean_record` for DNS.

2. Create `infra/variables.tf` for: `do_token`, `ssh_key_fingerprint`, `domain_name`, `region`.

3. Create `infra/outputs.tf` to output: `droplet_ip`, `db_connection_string` (sensitive = true), `database_host`.

4. Initialize and plan: `terraform init && terraform plan`

5. Apply (this creates real resources — they will cost money, stay within free tier):
   `terraform apply -var="do_token=$DO_TOKEN"`

6. Store state remotely: configure a `terraform { backend "s3" {} }` block or use Terraform Cloud for state locking and remote storage.

7. Destroy all resources when done: `terraform destroy`

**Verification:**
- [ ] `terraform plan` shows exactly the expected resources with no errors.
- [ ] After `apply`, the droplet IP is SSH-accessible.
- [ ] The DNS record resolves to the droplet IP within 5 minutes.
- [ ] `terraform state list` shows all expected resources.
- [ ] `terraform plan` after a successful apply shows "No changes" (idempotency).
- [ ] Changing the droplet size in `variables.tf` and re-running `plan` shows the expected change (resize or replace).
- [ ] `terraform destroy` removes all created resources (verify in the cloud console).

---

## Lab 10 — Kubernetes Deployment and Service (Advanced)

**Goal:** Deploy the Node.js application to a local Kubernetes cluster (k3s or minikube), expose it via a Service and Ingress, and perform a rolling update.

**Prerequisites:**
- `minikube` or `k3s` installed locally.
- `kubectl` configured (`kubectl cluster-info` works).
- Docker image from Lab 1 pushed to a registry accessible to the cluster.

**Tasks:**

1. Write `k8s/deployment.yml`:
   - `replicas: 3`
   - Resource requests and limits: `cpu: 100m/250m`, `memory: 128Mi/256Mi`.
   - Liveness probe: `GET /health` every 10s, failure threshold 3.
   - Readiness probe: `GET /health` every 5s, success threshold 1.
   - `strategy: RollingUpdate` with `maxSurge: 1`, `maxUnavailable: 0` (zero-downtime update).
   - ConfigMap for non-secret env vars; Secret for `DATABASE_URL`.

2. Write `k8s/service.yml`:
   - `type: ClusterIP`
   - Port: `80` → container `3000`.

3. Write `k8s/ingress.yml` (enable the Nginx ingress addon for minikube):
   - Host: `myapp.local`
   - Path: `/` → service.

4. Apply: `kubectl apply -f k8s/`

5. Verify rollout: `kubectl rollout status deployment/myapp`

6. Trigger a rolling update by changing the image tag in the deployment and re-applying. Watch pods be replaced without downtime:
   ```bash
   watch kubectl get pods
   ```

7. Simulate a bad deploy: update to a broken image tag. Observe the rollout pause. Roll back:
   `kubectl rollout undo deployment/myapp`

**Verification:**
- [ ] `kubectl get pods` shows 3 pods in `Running` state.
- [ ] `curl http://myapp.local/health` (via the Ingress) returns 200.
- [ ] During a rolling update, `kubectl get pods -w` shows old pods terminating only after new pods are ready.
- [ ] A broken image tag causes the rollout to halt — old pods remain running.
- [ ] `kubectl rollout undo` restores the previous working version within 60 seconds.
- [ ] `kubectl describe pod <name>` shows the resource limits and probes configured correctly.
- [ ] `kubectl logs -l app=myapp --tail=50` streams logs from all three pods.
