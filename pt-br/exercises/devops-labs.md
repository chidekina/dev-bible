# Laboratórios de DevOps

> Labs práticos para construir habilidades reais de DevOps. Cada lab tem um objetivo claro, pré-requisitos, tarefas passo a passo e critérios de verificação. Você trabalhará com Docker, GitHub Actions, Nginx, administração de sistemas Linux e ferramentas de monitoramento. Todos os labs assumem um ambiente Linux/WSL2. Conclua cada lab do início ao fim — não pule as etapas de verificação.

---

## Lab 1 — Dockerize uma Aplicação Node.js (Iniciante)

**Objetivo:** Empacotar uma aplicação Node.js/TypeScript em uma imagem Docker pronta para produção usando um build multi-stage. A imagem final deve ser mínima e segura.

**Pré-requisitos:**
- Docker Desktop instalado e rodando.
- Um projeto Node.js/TypeScript (use o app de exemplo no diretório `samples/node-app/` do lab ou o seu próprio).
- Entendimento básico do que é um container.

**Tarefas:**

1. Escreva um `Dockerfile` na raiz do projeto:
   - Estágio 1 (`builder`): use `node:20-alpine`, copie `package.json` e `package-lock.json`, execute `npm ci`, copie o código-fonte, execute `npm run build`.
   - Estágio 2 (`runtime`): use `node:20-alpine`, copie apenas a saída do build (`dist/`) e os `node_modules` de produção do estágio builder.
   - Defina um usuário não-root (`USER node`) no estágio de runtime.
   - `EXPOSE 3000` e defina `CMD ["node", "dist/index.js"]`.

2. Construa a imagem: `docker build -t myapp:local .`

3. Execute o container: `docker run -p 3000:3000 --env-file .env myapp:local`

4. Escreva um arquivo `.dockerignore` que exclua: `node_modules/`, `dist/`, `.git/`, `.env`, `*.log`.

5. Analise o tamanho da imagem: `docker image inspect myapp:local | grep Size`. Compare com um build de estágio único.

6. Adicione uma instrução de health check Docker: `HEALTHCHECK --interval=30s --timeout=5s CMD wget -qO- http://localhost:3000/health || exit 1`

**Verificação:**
- [ ] `docker ps` mostra o container rodando.
- [ ] `curl http://localhost:3000/health` retorna HTTP 200.
- [ ] `docker image ls myapp:local` mostra tamanho < 150MB (para um app TypeScript típico).
- [ ] Executar `docker exec <container_id> whoami` retorna `node` (não-root).
- [ ] O arquivo `.env` não está incorporado na imagem — verifique com `docker run myapp:local printenv` (segredos não devem aparecer).
- [ ] `docker inspect myapp:local` mostra o health check configurado.

**Desafio extra:** Faça o push da imagem para o GitHub Container Registry (`ghcr.io`) usando um Personal Access Token.

---

## Lab 2 — Docker Compose para uma Aplicação Full-Stack (Iniciante)

**Objetivo:** Usar o Docker Compose para orquestrar uma aplicação multi-serviço: API Node.js, banco PostgreSQL e cache Redis — tudo rodando localmente com um único comando.

**Pré-requisitos:**
- Docker Desktop com Compose v2 (`docker compose version`).
- Lab 1 concluído (você tem um Dockerfile funcionando).

**Tarefas:**

1. Crie `docker-compose.yml` com três serviços:
   - `api`: constrói a partir de `./Dockerfile`, depende de `db` e `redis`, mapeia a porta `3000:3000`, passa variáveis de ambiente de um arquivo `.env`.
   - `db`: usa `postgres:16-alpine`, define `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` do env, monta um volume nomeado `pgdata:/var/lib/postgresql/data`.
   - `redis`: usa `redis:7-alpine`, monta um volume nomeado `redisdata:/data`, usa `redis-server --appendonly yes` para persistência.

2. Adicione health checks aos serviços `db` e `redis`. Faça o serviço `api` usar `depends_on` com `condition: service_healthy`.

3. Inicie a stack: `docker compose up -d`

4. Execute as migrations do banco dentro do container `api` em execução: `docker compose exec api npm run db:migrate`

5. Adicione um `docker-compose.override.yml` para desenvolvimento local:
   - Monte o código-fonte como volume para hot reload: `./src:/app/src`.
   - Sobrescreva o comando para usar `npm run dev`.

6. Derrube: `docker compose down -v` (remove os volumes também — note a diferença de `down` sem `-v`).

**Verificação:**
- [ ] `docker compose ps` mostra os três serviços como `running` (não `starting` ou `exiting`).
- [ ] `docker compose logs api` mostra conexão bem-sucedida ao banco.
- [ ] `curl http://localhost:3000/health` retorna `{ "status": "ok", "db": "connected", "redis": "connected" }`.
- [ ] Após `docker compose down` e `docker compose up`, os dados do banco persistem (volumes não foram removidos).
- [ ] O arquivo override faz hot-reload em mudanças de arquivo ao usar `docker compose -f docker-compose.yml -f docker-compose.override.yml up`.

---

## Lab 3 — Pipeline CI/CD com GitHub Actions (Intermediário)

**Objetivo:** Construir um pipeline CI/CD completo que roda testes, constrói uma imagem Docker e faz deploy em um servidor remoto a cada push para `main`.

**Pré-requisitos:**
- Um repositório GitHub com a aplicação Node.js.
- Um servidor Linux remoto (VPS de tier gratuito ou VM local acessível via SSH).
- Docker instalado no servidor remoto.
- Secrets do repositório GitHub configurados.

**Tarefas:**

1. Crie `.github/workflows/ci.yml` — o job de CI:
   - Aciona em: `push` para `main` e `pull_request` para `main`.
   - Passos: checkout, setup Node.js 20, `npm ci`, `npm run lint`, `npm run test`, `npm run build`.
   - Cache: use `actions/cache` para fazer cache de `node_modules` baseado no hash de `package-lock.json`.

2. Crie `.github/workflows/deploy.yml` — o job de deploy (roda apenas em push para `main`, depois do CI passar):
   - Construa e faça push da imagem Docker para `ghcr.io/<seu-usuario>/<app>:latest`.
   - Conecte via SSH ao servidor remoto usando `appleboy/ssh-action`.
   - No servidor: faça pull da nova imagem, pare o container antigo, inicie o novo.

3. Adicione os Secrets necessários no GitHub: `SSH_HOST`, `SSH_USER`, `SSH_PRIVATE_KEY`, `GHCR_TOKEN`.

4. Adicione um badge de status no `README.md` mostrando o status do CI.

5. Introduza uma falha de teste deliberada para verificar que o pipeline bloqueia o deploy.

6. (Desafio) Adicione um passo de smoke test após o deploy: `curl https://<seu-dominio>/health` e falhe o job se retornar não-200.

**Verificação:**
- [ ] Abrir um PR aciona o job de CI. O PR não pode ser mergeado se o CI falhar (regra de proteção de branch).
- [ ] Fazer merge para `main` aciona o job de deploy.
- [ ] O job de deploy é concluído e a nova versão está no ar no servidor remoto em 3 minutos.
- [ ] Um teste falhando (`expect(1).toBe(2)`) em um PR bloqueia o pipeline — nenhum deploy ocorre.
- [ ] `docker ps` no servidor remoto mostra a nova tag de imagem do container.
- [ ] Os logs do GitHub Actions mostram o cache de `node_modules` sendo restaurado (não re-baixado) na segunda execução.

---

## Lab 4 — Nginx como Reverse Proxy (Intermediário)

**Objetivo:** Configurar o Nginx para fazer reverse proxy de múltiplos serviços backend, gerenciar terminação SSL com Let's Encrypt e servir assets estáticos de forma eficiente.

**Pré-requisitos:**
- Um servidor Linux com endereço IP público.
- Um nome de domínio apontando para esse IP.
- Nginx instalado (`sudo apt install nginx`).
- `certbot` instalado (`sudo snap install certbot --classic`).

**Tarefas:**

1. Crie um server block Nginx em `/etc/nginx/sites-available/myapp`:
   - Escute na porta 80. Redirecione todo o tráfego HTTP para HTTPS.
   - Escute na porta 443 com SSL (certificados serão adicionados pelo Certbot).
   - Faça proxy de `/api/` para `http://localhost:3000/` com headers: `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`.
   - Sirva arquivos estáticos de `/var/www/myapp/` diretamente (para um build de frontend).
   - Defina `gzip on` com `gzip_types text/plain text/css application/json application/javascript`.

2. Habilite o site: `sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/`

3. Teste a configuração: `sudo nginx -t`

4. Obtenha um certificado SSL: `sudo certbot --nginx -d seudominio.com`

5. Configure o Nginx com headers de segurança:
   - `add_header X-Frame-Options "SAMEORIGIN";`
   - `add_header X-Content-Type-Options "nosniff";`
   - `add_header Referrer-Policy "strict-origin-when-cross-origin";`
   - `add_header Content-Security-Policy "default-src 'self';";`

6. Configure rate limiting no Nginx:
   - `limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;`
   - Aplique `limit_req zone=api burst=20 nodelay;` ao bloco de location `/api/`.

7. Configure a renovação automática do certificado: `sudo certbot renew --dry-run`

**Verificação:**
- [ ] `curl -I http://seudominio.com` retorna `301 Moved Permanently` para HTTPS.
- [ ] `curl https://seudominio.com/api/health` retorna a resposta da API.
- [ ] `curl https://seudominio.com/index.html` serve o arquivo estático.
- [ ] Teste do SSL Labs (ssllabs.com/ssltest) dá nota A ou A+.
- [ ] Os headers de segurança estão presentes na resposta (`curl -I https://seudominio.com`).
- [ ] Enviar 30 requisições rápidas para `/api/` resulta em respostas `429 Too Many Requests` para o excesso.
- [ ] `sudo crontab -l` ou `systemctl list-timers` mostra a renovação automática do Certbot agendada.

---

## Lab 5 — Hardening de um VPS Linux (Intermediário)

**Objetivo:** Aplicar hardening de segurança em um VPS Ubuntu recém-provisionado para reduzir a superfície de ataque antes de fazer deploy de qualquer aplicação.

**Pré-requisitos:**
- Um VPS Ubuntu 22.04 LTS fresco com acesso root.
- Um par de chaves SSH local.

**Tarefas:**

1. **Crie um usuário não-root:**
   ```bash
   adduser deploy
   usermod -aG sudo deploy
   ```

2. **Hardening de SSH** — edite `/etc/ssh/sshd_config`:
   - `PermitRootLogin no`
   - `PasswordAuthentication no`
   - `PubkeyAuthentication yes`
   - `Port 2222` (mude da porta padrão 22)
   - Copie sua chave pública: `ssh-copy-id -i ~/.ssh/id_ed25519.pub -p 22 deploy@<ip>`

3. **Configure o UFW (Uncomplicated Firewall):**
   ```bash
   ufw default deny incoming
   ufw default allow outgoing
   ufw allow 2222/tcp   # nova porta SSH
   ufw allow 80/tcp
   ufw allow 443/tcp
   ufw enable
   ```

4. **Instale e configure o Fail2Ban:**
   - Instale: `sudo apt install fail2ban`
   - Crie `/etc/fail2ban/jail.local` com jail SSH: maxretry 3, bantime 1h, findtime 10m.
   - Inicie e habilite: `sudo systemctl enable --now fail2ban`

5. **Atualizações de segurança automáticas:**
   ```bash
   sudo apt install unattended-upgrades
   sudo dpkg-reconfigure --priority=low unattended-upgrades
   ```

6. **Hardening do sistema — adições em `/etc/sysctl.conf`:**
   ```
   net.ipv4.conf.all.rp_filter = 1
   net.ipv4.conf.all.accept_source_route = 0
   net.ipv4.tcp_syncookies = 1
   kernel.dmesg_restrict = 1
   ```
   Aplique: `sudo sysctl -p`

7. **Audite os pacotes instalados:** `sudo apt list --installed | wc -l`. Remova pacotes obviamente desnecessários.

**Verificação:**
- [ ] `ssh root@<ip> -p 2222` é rejeitado: "Permission denied".
- [ ] `ssh deploy@<ip> -p 2222` com a chave privada funciona.
- [ ] `ssh deploy@<ip> -p 22` timed out (porta 22 está fechada pelo UFW).
- [ ] `sudo ufw status verbose` mostra apenas as portas 2222, 80, 443 abertas.
- [ ] `sudo fail2ban-client status sshd` mostra o jail ativo.
- [ ] `sudo apt list --upgradable` retorna vazio ou apenas atualizações não-segurança.
- [ ] Executar um port scan rápido com `nmap -sV <ip>` de fora mostra portas abertas mínimas.

---

## Lab 6 — Configurar Monitoramento com Prometheus e Grafana (Intermediário)

**Objetivo:** Fazer deploy de uma stack de monitoramento usando Docker Compose. Instrumentar uma aplicação Node.js, coletar métricas com Prometheus e visualizá-las no Grafana.

**Pré-requisitos:**
- Docker Compose instalado.
- A aplicação Node.js do Lab 1 rodando localmente.
- Entendimento básico de conceitos de métricas de série temporal.

**Tarefas:**

1. Adicione o client Prometheus à aplicação Node.js:
   ```bash
   npm install prom-client
   ```
   Exponha um endpoint `/metrics` que retorna métricas no formato Prometheus. Instrumente:
   - Taxa de requisições HTTP (counter por método, rota, status code).
   - Histograma de duração de requisições HTTP (P50, P95, P99).
   - Gauge de conexões ativas.
   - Métricas padrão do Node.js (CPU, memória, event loop lag) via `collectDefaultMetrics()`.

2. Crie `monitoring/prometheus.yml`:
   ```yaml
   scrape_configs:
     - job_name: 'nodejs-app'
       static_configs:
         - targets: ['app:3000']
       metrics_path: '/metrics'
       scrape_interval: 15s
   ```

3. Adicione os serviços `prometheus` e `grafana` ao `docker-compose.yml`:
   - Prometheus: `prom/prometheus:latest`, monte a configuração, porta `9090:9090`.
   - Grafana: `grafana/grafana:latest`, porta `3001:3000`, volume para persistência, variáveis de env para credenciais admin padrão.

4. Importe o **Node.js Application Dashboard** (ID do dashboard Grafana `11159`) pela UI do Grafana.

5. Crie um dashboard customizado no Grafana com painéis:
   - Taxa de requisições (req/seg) como gráfico.
   - Taxa de erros (5xx/min) como painel stat com threshold vermelho.
   - Latência P99 como gauge.
   - Uso de memória (RSS) como gráfico de série temporal.

6. Configure um alerta no Grafana: se a taxa de erros ultrapassar 5 erros/min por mais de 2 minutos, dispare um alerta (use o alerting de email ou webhook embutido — nenhum serviço externo necessário para o lab).

**Verificação:**
- [ ] `curl http://localhost:3000/metrics` retorna texto no formato Prometheus.
- [ ] `http://localhost:9090/targets` mostra o app Node.js como `UP`.
- [ ] O dashboard importado do Grafana mostra dados em todos os painéis.
- [ ] Gerar carga (`ab -n 1000 -c 10 http://localhost:3000/`) mostra um pico no painel de taxa de requisições.
- [ ] Introduzir um erro 500 deliberado aciona o alerta do Grafana dentro da janela configurada.
- [ ] O Grafana persiste dashboards após `docker compose restart grafana`.

---

## Lab 7 — Estratégia de Deploy Zero-Downtime (Intermediário)

**Objetivo:** Implementar um deploy blue-green para a aplicação Node.js usando Nginx e Docker Compose, obtendo atualizações sem downtime.

**Pré-requisitos:**
- Labs 1, 2 e 4 concluídos.
- Nginx configurado como reverse proxy.

**Tarefas:**

1. Execute duas versões do app simultaneamente:
   - Container `blue`: versão atual de produção na porta `3001`.
   - Container `green`: nova versão na porta `3002`.

2. Configuração de upstream do Nginx:
   ```nginx
   upstream app {
     server localhost:3001;  # ativo: blue
   }
   ```

3. Escreva um script de deploy `scripts/deploy.sh`:
   - Faça pull da nova imagem Docker.
   - Inicie o container `green` na porta 3002.
   - Aguarde o health check passar (`curl http://localhost:3002/health`).
   - Atualize o upstream do Nginx para apontar para a porta 3002.
   - Recarregue o Nginx (`nginx -s reload` — reload zero-downtime, sem restart).
   - Pare o container `blue` antigo.
   - Renomeie `green` para `blue` para o próximo deploy.

4. Teste com carga contínua: `while true; do curl -s http://localhost/health; sleep 0.1; done`
   Execute o script de deploy enquanto este loop está ativo. Zero requisições devem falhar.

5. Implemente um mecanismo de rollback no script: se o health check falhar, remova o container `green` e mantenha o `blue` rodando.

**Verificação:**
- [ ] Durante o deploy, zero erros HTTP 502 ou 500 são observados na saída do teste de carga.
- [ ] O log de erros do Nginx não mostra erros durante o reload de configuração.
- [ ] Após o deploy, `docker ps` mostra apenas um container do app rodando.
- [ ] Quebrar deliberadamente a nova versão (retornando 500 em `/health`) faz o rollback disparar automaticamente.
- [ ] O tempo total de deploy (início do script → tráfego redirecionado) é inferior a 60 segundos.

---

## Lab 8 — Pipeline de Logging Estruturado com Loki e Grafana (Intermediário)

**Objetivo:** Configurar um sistema centralizado de agregação de logs usando Grafana Loki, Promtail e Grafana. Enviar logs da aplicação e do sistema e consultá-los na UI do Grafana.

**Pré-requisitos:**
- Docker Compose e a stack de monitoramento do Lab 6.
- A aplicação Node.js registrando logs em formato JSON (Pino).

**Tarefas:**

1. Adicione os serviços `loki` e `promtail` ao `docker-compose.yml`:
   - Loki: `grafana/loki:latest`, porta `3100:3100`, persista dados com volume.
   - Promtail: `grafana/promtail:latest`, monte `/var/log` e o socket Docker.

2. Configure `monitoring/promtail.yml`:
   - Colete logs de containers Docker de `/var/log/docker/containers/**/*.log`.
   - Adicione labels: `job`, `container_name`, `stream` (stdout/stderr).
   - Estágio de pipeline: analise logs JSON, extraia `level`, `requestId`, `statusCode` como labels.

3. Adicione o Loki como fonte de dados no Grafana (`http://loki:3100`).

4. Escreva queries LogQL na view Explore do Grafana:
   - Todos os logs de erro na última hora: `{job="app"} |= "error"`
   - Requisições com status 500: `{job="app"} | json | statusCode = "500"`
   - Taxa de requisições por rota: `sum by (url) (rate({job="app"}[5m]))`

5. Crie um painel de Logs no seu dashboard do Grafana mostrando logs da aplicação em tempo real.

6. Configure um alerta baseado em log: se a string `"CRITICAL"` aparecer em qualquer log em 1 minuto, dispare um alerta.

**Verificação:**
- [ ] `curl http://localhost:3100/ready` retorna `ready`.
- [ ] O Grafana Explore com fonte de dados Loki mostra logs do container Node.js.
- [ ] Consultar `{job="app"} |= "error"` retorna entradas de log de nível error.
- [ ] Gerar um erro 500 no app produz uma entrada de log visível no Grafana em 15 segundos.
- [ ] Os campos JSON (`level`, `requestId`) são corretamente analisados e disponíveis como labels para filtragem.

---

## Lab 9 — Infraestrutura como Código com Terraform (Avançado)

**Objetivo:** Provisionar uma infraestrutura completa de aplicação em um provedor cloud usando Terraform. A infraestrutura inclui um VPS/VM, banco gerenciado e registro DNS — tudo definido como código.

**Pré-requisitos:**
- Terraform CLI instalado (`terraform version`).
- Conta em um provedor cloud (DigitalOcean ou AWS free tier recomendado).
- Token de API/credenciais configurados.

**Tarefas:**

1. Crie `infra/main.tf`:
   - Provedor: `digitalocean` (ou `aws`).
   - Recursos: `digitalocean_droplet` (2GB RAM, `ubuntu-22-04`), `digitalocean_database_cluster` (PostgreSQL 15, 1 nó, plano menor), `digitalocean_domain` e `digitalocean_record` para DNS.

2. Crie `infra/variables.tf` para: `do_token`, `ssh_key_fingerprint`, `domain_name`, `region`.

3. Crie `infra/outputs.tf` para retornar: `droplet_ip`, `db_connection_string` (sensitive = true), `database_host`.

4. Inicialize e planeje: `terraform init && terraform plan`

5. Aplique (isso cria recursos reais — eles terão custo, fique dentro do free tier):
   `terraform apply -var="do_token=$DO_TOKEN"`

6. Armazene o state remotamente: configure um bloco `terraform { backend "s3" {} }` ou use o Terraform Cloud para locking de state e armazenamento remoto.

7. Destrua todos os recursos quando terminar: `terraform destroy`

**Verificação:**
- [ ] `terraform plan` mostra exatamente os recursos esperados sem erros.
- [ ] Após o `apply`, o IP do droplet é acessível via SSH.
- [ ] O registro DNS resolve para o IP do droplet em 5 minutos.
- [ ] `terraform state list` mostra todos os recursos esperados.
- [ ] `terraform plan` após um apply bem-sucedido mostra "No changes" (idempotência).
- [ ] Mudar o tamanho do droplet em `variables.tf` e re-executar `plan` mostra a mudança esperada (resize ou replace).
- [ ] `terraform destroy` remove todos os recursos criados (verifique no console do provedor cloud).

---

## Lab 10 — Deployment e Service no Kubernetes (Avançado)

**Objetivo:** Fazer deploy da aplicação Node.js em um cluster Kubernetes local (k3s ou minikube), expô-la via Service e Ingress, e realizar uma rolling update.

**Pré-requisitos:**
- `minikube` ou `k3s` instalado localmente.
- `kubectl` configurado (`kubectl cluster-info` funciona).
- Imagem Docker do Lab 1 enviada para um registry acessível ao cluster.

**Tarefas:**

1. Escreva `k8s/deployment.yml`:
   - `replicas: 3`
   - Requests e limits de recursos: `cpu: 100m/250m`, `memory: 128Mi/256Mi`.
   - Liveness probe: `GET /health` a cada 10s, failure threshold 3.
   - Readiness probe: `GET /health` a cada 5s, success threshold 1.
   - `strategy: RollingUpdate` com `maxSurge: 1`, `maxUnavailable: 0` (atualização zero-downtime).
   - ConfigMap para variáveis de env não-secretas; Secret para `DATABASE_URL`.

2. Escreva `k8s/service.yml`:
   - `type: ClusterIP`
   - Porta: `80` → container `3000`.

3. Escreva `k8s/ingress.yml` (habilite o addon Nginx ingress para minikube):
   - Host: `myapp.local`
   - Path: `/` → service.

4. Aplique: `kubectl apply -f k8s/`

5. Verifique o rollout: `kubectl rollout status deployment/myapp`

6. Acione uma rolling update mudando a tag de imagem no deployment e reaplicando. Observe os pods sendo substituídos sem downtime:
   ```bash
   watch kubectl get pods
   ```

7. Simule um deploy ruim: atualize para uma tag de imagem quebrada. Observe o rollout pausar. Faça rollback:
   `kubectl rollout undo deployment/myapp`

**Verificação:**
- [ ] `kubectl get pods` mostra 3 pods no estado `Running`.
- [ ] `curl http://myapp.local/health` (via Ingress) retorna 200.
- [ ] Durante uma rolling update, `kubectl get pods -w` mostra pods antigos terminando apenas depois que os novos estão prontos.
- [ ] Uma tag de imagem quebrada faz o rollout pausar — os pods antigos continuam rodando.
- [ ] `kubectl rollout undo` restaura a versão anterior funcionando em 60 segundos.
- [ ] `kubectl describe pod <name>` mostra os limits de recursos e probes configurados corretamente.
- [ ] `kubectl logs -l app=myapp --tail=50` transmite logs dos três pods.
