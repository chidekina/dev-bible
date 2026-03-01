# Deploy em VPS

## Visão Geral

Um VPS (Virtual Private Server) é um servidor Linux virtualizado em um data center que você aluga e controla totalmente. Ao contrário de plataformas gerenciadas (Heroku, Railway, Render), um VPS dá acesso root, controle total sobre o stack de software e custos significativamente menores em escala — ao custo da responsabilidade operacional.

Fazer deploy em um VPS significa que você é dono de toda a stack: SO, servidor web, runtime, banco de dados, backups, monitoramento e segurança. Isso dá mais trabalho do que um PaaS, mas as habilidades se transferem para qualquer lugar, os custos são previsíveis e não há restrições específicas de plataforma.

Este capítulo cobre o deploy de uma aplicação Node.js/TypeScript em produção para um VPS usando Docker, Nginx e um pipeline de CI/CD com GitHub Actions.

---

## Pré-requisitos

- Um VPS com Ubuntu 22.04/24.04 (DigitalOcean, Hetzner, Linode, Vultr, etc.)
- Um domínio com DNS apontando para o IP do seu VPS
- Conhecimento básico de Linux e Docker
- Um repositório GitHub com sua aplicação

Especificações mínimas recomendadas para uma aplicação pequena em produção: 2 vCPU, 2 GB RAM, 40 GB SSD.

---

## Conceitos Fundamentais

### Arquitetura de Deploy

```
GitHub (código-fonte + CI/CD)
    |
    | push para main → GitHub Actions roda
    |
    ↓
Docker Hub / GHCR (registry de containers)
    |
    | docker pull (do VPS)
    |
    ↓
VPS (Ubuntu 22.04)
    ├── Nginx (reverse proxy, terminação SSL)
    ├── Containers Docker:
    │   ├── myapi:latest    (porta 3000, interna)
    │   ├── postgres:16     (porta 5432, interna)
    │   └── redis:7         (porta 6379, interna)
    └── Certbot (Let's Encrypt, renovação automática)
```

### Estrutura de Diretórios no Servidor

```
/opt/myapp/
  compose.yml              ← arquivo docker compose de produção
  .env.production          ← secrets (permissões 600, não no git)
  backups/                 ← backups do banco de dados
  logs/                    ← logs da aplicação (se montados via bind)

/var/log/nginx/            ← logs de acesso e erros do Nginx
/etc/nginx/sites-available/myapp  ← config do Nginx
/etc/letsencrypt/          ← certificados SSL
```

### Estratégias de Deploy

| Estratégia | Descrição | Downtime |
|----------|-----------|----------|
| Stop → Pull → Start | Mais simples. Para o antigo, inicia o novo. | ~5-30 segundos |
| Blue-Green | Rodar dois ambientes, chavear o roteamento | Zero |
| Rolling update | Substituir instâncias uma por uma | Zero (com load balancer) |
| Canary | Rotear pequeno % para nova versão primeiro | Zero |

Para um único VPS, Stop → Pull → Start é geralmente aceitável. Para zero downtime, você precisa de dois containers e um restart que respeite os health checks.

---

## Exemplos Práticos

### Exemplo 1: Configuração Inicial do Servidor

Após obter acesso root ao VPS:

```bash
# 1. Atualizar o sistema
apt update && apt upgrade -y

# 2. Instalar essenciais
apt install -y \
  curl wget git ufw fail2ban \
  htop ncdu tmux jq

# 3. Instalar Docker
curl -fsSL https://get.docker.com | sh

# 4. Criar usuário de deploy (nunca rodar aplicações como root)
useradd -m -s /bin/bash deploy
usermod -aG docker deploy
usermod -aG sudo deploy

# 5. Configurar chave SSH para o usuário deploy
mkdir -p /home/deploy/.ssh
# Adicionar sua chave pública:
echo "ssh-ed25519 AAAA..." >> /home/deploy/.ssh/authorized_keys
chown -R deploy:deploy /home/deploy/.ssh
chmod 700 /home/deploy/.ssh
chmod 600 /home/deploy/.ssh/authorized_keys

# 6. Instalar Nginx
apt install -y nginx certbot python3-certbot-nginx

# 7. Criar diretório da aplicação
mkdir -p /opt/myapp
chown deploy:deploy /opt/myapp

# 8. Configurar firewall (veja vps-hardening.md para hardening completo)
ufw allow ssh
ufw allow 80/tcp
ufw allow 443/tcp
ufw --force enable
```

### Exemplo 2: Docker Compose de Produção

`/opt/myapp/compose.yml`:
```yaml
name: myapp

services:
  api:
    image: ghcr.io/yourusername/myapp:${VERSION:-latest}
    restart: unless-stopped
    expose:
      - "3000"
    environment:
      NODE_ENV: production
      DATABASE_URL: postgres://app:${POSTGRES_PASSWORD}@db:5432/app
      REDIS_URL: redis://cache:6379
      PORT: "3000"
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_healthy
    networks:
      - internal
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: app
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 10s
    networks:
      - internal
    logging:
      driver: "json-file"
      options:
        max-size: "5m"
        max-file: "3"

  cache:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --maxmemory 128mb --maxmemory-policy allkeys-lru --save ""
    volumes:
      - redisdata:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    networks:
      - internal

volumes:
  pgdata:
  redisdata:

networks:
  internal:
    driver: bridge
```

`/opt/myapp/.env.production` (permissões: 600, nunca commitado no git):
```bash
POSTGRES_PASSWORD=use_a_long_random_string_here
```

### Exemplo 3: Configuração do Nginx

`/etc/nginx/sites-available/myapp`:
```nginx
# HTTP → HTTPS redirect
server {
    listen 80;
    listen [::]:80;
    server_name myapp.com www.myapp.com;
    return 301 https://$host$request_uri;
}

# HTTPS
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name myapp.com www.myapp.com;

    ssl_certificate     /etc/letsencrypt/live/myapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/myapp.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;

    add_header Strict-Transport-Security "max-age=63072000" always;
    add_header X-Frame-Options SAMEORIGIN always;
    add_header X-Content-Type-Options nosniff always;

    # Logging
    access_log /var/log/nginx/myapp.access.log;
    error_log  /var/log/nginx/myapp.error.log;

    # Proxy da API
    location / {
        proxy_pass         http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_set_header   Connection        "";

        proxy_connect_timeout 10s;
        proxy_send_timeout    60s;
        proxy_read_timeout    60s;

        client_max_body_size 10m;
    }
}
```

**Nota:** A porta 3000 do container não é publicada diretamente. Precisamos de um mapeamento de porta no host para o Nginx alcançá-la. Atualize o compose.yml:

```yaml
api:
  ports:
    - "127.0.0.1:3000:3000"   # bind apenas no loopback — Nginx alcança, a internet não
```

Habilitar e obter certificado:
```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
sudo certbot --nginx -d myapp.com -d www.myapp.com
```

### Exemplo 4: Workflow de Deploy com GitHub Actions

`.github/workflows/deploy.yml`:
```yaml
name: Deploy

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run build
      - run: npm test

  build:
    name: Build & Push Image
    runs-on: ubuntu-latest
    needs: test
    permissions:
      contents: read
      packages: write
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}
    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=raw,value=latest

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          target: production

  deploy:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: production
      url: https://myapp.com
    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.VPS_HOST }}
          username: deploy
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            set -e
            cd /opt/myapp

            # Autenticar com o registry
            echo "${{ secrets.GITHUB_TOKEN }}" | \
              docker login ghcr.io -u "${{ github.actor }}" --password-stdin

            # Baixar nova imagem
            docker compose pull api

            # Rodar migrations (antes de chavear o tráfego)
            docker compose run --rm \
              -e DATABASE_URL="$(grep DATABASE_URL .env.production | cut -d= -f2)" \
              api node dist/migrate.js

            # Reiniciar com nova imagem
            docker compose up -d --remove-orphans

            # Aguardar health check
            sleep 10
            curl -f http://localhost:3000/health || (docker compose logs api && exit 1)

            # Limpar imagens antigas
            docker image prune -f
```

Secrets necessários no GitHub:
- `VPS_HOST`: IP ou hostname do seu VPS
- `VPS_SSH_KEY`: Chave SSH privada (a chave pública correspondente está no `authorized_keys` do usuário `deploy`)

### Exemplo 5: Script de Backup do Banco de Dados

`/opt/myapp/scripts/backup.sh`:
```bash
#!/usr/bin/env bash
set -euo pipefail

BACKUP_DIR="/opt/myapp/backups"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/pg_${DATE}.sql.gz"
KEEP_DAYS=7

mkdir -p "$BACKUP_DIR"

# Carregar env
source /opt/myapp/.env.production

# Dump do banco a partir do container postgres em execução
docker exec myapp-db-1 \
  pg_dump -U app app \
  | gzip > "$BACKUP_FILE"

echo "Backup criado: $BACKUP_FILE ($(du -sh $BACKUP_FILE | cut -f1))"

# Remover backups mais antigos que KEEP_DAYS
find "$BACKUP_DIR" -name "pg_*.sql.gz" -mtime +"$KEEP_DAYS" -delete
echo "Backups mais antigos que $KEEP_DAYS dias removidos"
```

```bash
chmod +x /opt/myapp/scripts/backup.sh

# Adicionar ao crontab (como usuário deploy)
crontab -e
# Adicionar:
0 2 * * * /opt/myapp/scripts/backup.sh >> /var/log/myapp-backup.log 2>&1
```

---

## Padrões e Boas Práticas

### Gerenciamento de Variáveis de Ambiente

```bash
# Gerar um secret aleatório
openssl rand -base64 32

# No VPS, .env.production contém os secrets
# Nunca commite .env.production no git
# Use um secrets manager (Doppler, Vault) para ambientes em equipe
```

### Rotação de Logs

Containers Docker escrevem logs em `/var/lib/docker/containers/<id>/<id>-json.log`. Configure limites no compose.yml:

```yaml
logging:
  driver: "json-file"
  options:
    max-size: "10m"   # máximo 10 MB por arquivo de log
    max-file: "5"     # manter 5 arquivos → máximo 50 MB por service
```

### Manter o SO Atualizado

```bash
# Habilitar atualizações de segurança automáticas
apt install -y unattended-upgrades
dpkg-reconfigure -pmedium unattended-upgrades
```

### Deploy com Zero Downtime e Health Checks

```bash
# No script de deploy — aguardar o novo container estar saudável antes de finalizar
docker compose up -d --remove-orphans

for i in {1..30}; do
  if curl -sf http://localhost:3000/health > /dev/null; then
    echo "Service saudável após ${i}s"
    exit 0
  fi
  sleep 1
done

echo "Service não ficou saudável em 30s"
docker compose logs api --tail 50
exit 1
```

---

## Anti-padrões a Evitar

### Rodar Aplicações Diretamente (Sem Docker)

Rodar Node.js diretamente no host mistura as dependências da aplicação com o SO, cria conflitos de porta entre aplicações e torna as atualizações trabalhosas. Use Docker.

### Expor Portas do Banco de Dados para a Internet

```yaml
# RUIM — Postgres acessível da internet
db:
  ports:
    - "5432:5432"

# BOM — acessível apenas dentro da rede Docker
db:
  expose:
    - "5432"
```

### Armazenar Secrets no Git

`.env.production` deve estar no `.gitignore`. Use GitHub Secrets para valores de CI/CD.

### Não Testar Antes de Deployar

Sempre rode os testes no CI antes do job de deploy. Um teste com falha deve impedir o deploy.

### Sem Backups

Seu banco de dados é o seu negócio. Backups diários automatizados com armazenamento externo (S3, B2) são inegociáveis para qualquer banco de dados em produção.

### Deployar como Root

Nunca faça SSH como root para deploys. Crie um usuário `deploy` com permissões mínimas. Se for comprometido, o raio de impacto fica contido.

---

## Debug & Troubleshooting

### Verificar Status dos Containers

```bash
cd /opt/myapp
docker compose ps
docker compose logs api --tail 100
docker compose logs db --tail 50
```

### Container Não Inicia

```bash
docker compose up api   # rodar em foreground para ver erros
docker compose run --rm api node --version   # testar o básico da imagem
```

### Problemas de Conexão com o Banco

```bash
# Testar de dentro do container api
docker compose exec api sh
# Então:
nc -zv db 5432    # conseguimos alcançar o postgres?
env | grep DATABASE_URL
```

### Erros do Nginx

```bash
sudo nginx -t                              # verificar sintaxe
sudo tail -f /var/log/nginx/myapp.error.log
sudo tail -f /var/log/nginx/myapp.access.log | grep " 5[0-9][0-9] "
```

### Espaço em Disco

```bash
df -h              # uso geral de disco
docker system df   # uso específico do Docker
docker system prune -f    # remover containers, networks e imagens não usadas
du -sh /var/lib/docker/   # uso total de disco do Docker
```

---

## Cenários do Mundo Real

### Cenário 1: Reverter um Deploy Problemático

```bash
# No VPS — reverter para a versão anterior da imagem
cd /opt/myapp

# Listar imagens recentes
docker images ghcr.io/yourusername/myapp --format "{{.Tag}}\t{{.CreatedAt}}"

# Reverter para um SHA específico
VERSION=sha-abc1234 docker compose up -d api
```

### Cenário 2: Rodar Migrations de Banco com Segurança

```bash
# Testar migration em um snapshot do banco primeiro
# 1. Fazer backup
./scripts/backup.sh

# 2. Rodar migration
docker compose run --rm api node dist/migrate.js

# 3. Se falhar, restaurar
gunzip -c backups/pg_YYYYMMDD_HHMMSS.sql.gz | \
  docker exec -i myapp-db-1 psql -U app app
```

### Cenário 3: Monitorar Saúde dos Containers

```bash
# Visão rápida de saúde
watch -n 5 'docker compose ps && echo "---" && docker stats --no-stream'

# Verificar se containers estão usando muita memória
docker stats --format "table {{.Name}}\t{{.MemUsage}}\t{{.CPUPerc}}" --no-stream
```

---

## Leitura Complementar

- [DigitalOcean Tutorials](https://www.digitalocean.com/community/tutorials)
- [Hetzner Cloud Docs](https://docs.hetzner.com/cloud/)
- [Docker in Production](https://docs.docker.com/config/containers/start-containers-automatically/)
- [SSH Key Management](https://www.ssh.com/academy/ssh/keygen)

---

## Resumo

| Etapa | Comando / Ação |
|------|----------------|
| Configuração inicial | Criar usuário `deploy`, instalar Docker + Nginx, configurar UFW |
| Diretório da aplicação | `/opt/myapp/compose.yml` + `.env.production` (chmod 600) |
| Nginx | Reverse proxy na 443, redirect HTTP, certbot SSL |
| CI/CD | GitHub Actions: test → build image → push → SSH deploy |
| Secrets | VPS_HOST + VPS_SSH_KEY no GitHub Secrets |
| Backups | Cron job, `pg_dump` via docker exec, manter 7 dias |
| Logging | Driver json-file do Docker com limites de tamanho/rotação |
| Rollback | `docker compose up -d` com a tag da imagem anterior |

Um VPS é uma tela em branco. Os padrões deste capítulo fornecem um deploy de nível produção que é repetível, seguro e automatizável. Comece simples, automatize tudo e adicione complexidade apenas quando necessário.
