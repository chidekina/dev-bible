# Docker Compose

## Visão Geral

Docker Compose é uma ferramenta para definir e executar aplicações multi-container usando um único arquivo YAML. Em vez de rodar longos comandos `docker run` com dezenas de flags, você declara toda a sua stack — API, banco de dados, cache, reverse proxy — e gerencia tudo com comandos simples.

O Compose é a ponte entre "rodar um único container" e "rodar uma aplicação de verdade." Aplicações reais têm dependências: uma API Node.js precisa de um banco Postgres, um cache Redis e talvez um worker em background. O Compose conecta tudo isso, gerencia a ordem de inicialização, cuida do networking e torna a stack inteira reproduzível.

Este capítulo cobre o Compose v2 (a versão moderna, integrada ao CLI do Docker como `docker compose`).

---

## Pré-requisitos

- Docker instalado e funcionando (veja `docker-fundamentals.md`)
- Entendimento básico de imagens, containers, volumes e networks do Docker

```bash
# Verificar se o Compose v2 está disponível
docker compose version
# Docker Compose version v2.x.x
```

---

## Conceitos Fundamentais

### O Arquivo Compose

Um arquivo `docker-compose.yml` (ou `compose.yml`) declara services, volumes e networks:

```yaml
name: myapp

services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgres://app:secret@db:5432/app
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: app
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  pgdata:
```

### Comandos Principais

```bash
docker compose up              # iniciar todos os services (foreground)
docker compose up -d           # iniciar em detached (background)
docker compose up --build      # rebuild das imagens antes de iniciar
docker compose down            # parar e remover containers
docker compose down -v         # também remover volumes
docker compose ps              # listar services em execução
docker compose logs api        # logs de um service
docker compose logs -f         # seguir todos os logs
docker compose exec api sh     # shell dentro de um service em execução
docker compose run api node migrate.js  # executar comando único
docker compose restart api     # reiniciar um service
docker compose pull            # baixar imagens mais recentes
```

### Chaves de Definição de Service

```yaml
services:
  myservice:
    # Fonte da imagem (uma destas):
    image: node:22-alpine          # baixar do registry
    build: .                       # build do Dockerfile em .
    build:
      context: .
      dockerfile: Dockerfile.prod
      args:
        NODE_ENV: production
      target: production           # alvo de build multi-stage

    # Portas: HOST:CONTAINER
    ports:
      - "3000:3000"
      - "127.0.0.1:3000:3000"    # bind apenas no loopback (mais seguro)

    # Variáveis de ambiente
    environment:
      NODE_ENV: production
      PORT: 3000
    env_file:
      - .env.production            # arquivo com linhas KEY=VALUE

    # Volumes
    volumes:
      - ./src:/app/src             # bind mount
      - nodemodules:/app/node_modules  # volume nomeado

    # Dependências
    depends_on:
      db:
        condition: service_healthy  # aguardar o healthcheck passar
      redis:
        condition: service_started  # aguardar apenas o container iniciar

    # Política de restart
    restart: unless-stopped        # em caso de falha, exceto parada manual

    # Limites de recursos (sintaxe Compose v2 + Docker Swarm)
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 256M

    # Networks
    networks:
      - backend
      - frontend
```

### Networking no Compose

Por padrão, o Compose cria uma única rede para o projeto. Todos os services nessa rede se alcançam pelo nome do service:

```yaml
services:
  api:
    image: myapi:latest
    # Pode alcançar "db" pelo hostname "db"

  db:
    image: postgres:16-alpine
    # Pode alcançar "api" pelo hostname "api"
```

Networks customizadas permitem segmentar o tráfego:

```yaml
services:
  nginx:
    networks: [frontend, backend]
  api:
    networks: [backend]
  db:
    networks: [backend]
  # nginx pode falar com api e db
  # api e db podem falar entre si
  # nada externo acessa db diretamente

networks:
  frontend:
  backend:
    internal: true  # sem acesso à internet de saída
```

---

## Exemplos Práticos

### Exemplo 1: Stack Node.js Completa

```yaml
# compose.yml
name: myapp

services:
  api:
    build:
      context: .
      target: production
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: production
      DATABASE_URL: postgres://app:${POSTGRES_PASSWORD}@db:5432/app
      REDIS_URL: redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - backend

  db:
    image: postgres:16-alpine
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
    restart: unless-stopped
    networks:
      - backend

  cache:
    image: redis:7-alpine
    command: redis-server --maxmemory 128mb --maxmemory-policy allkeys-lru
    volumes:
      - redisdata:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    restart: unless-stopped
    networks:
      - backend

volumes:
  pgdata:
  redisdata:

networks:
  backend:
```

Arquivo `.env` (referenciado por `${POSTGRES_PASSWORD}`):
```bash
POSTGRES_PASSWORD=changeme_in_production
```

```bash
docker compose up -d
docker compose logs -f api
curl http://localhost:3000/health
```

### Exemplo 2: Overrides de Desenvolvimento vs Produção

Use um arquivo base + arquivos de override para compartilhar configuração comum:

`compose.yml` (base):
```yaml
name: myapp

services:
  api:
    build: .
    environment:
      NODE_ENV: development
    networks:
      - backend

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: app
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 3s
      retries: 5
    networks:
      - backend

networks:
  backend:
```

`compose.override.yml` (desenvolvimento — carregado automaticamente):
```yaml
services:
  api:
    build:
      target: development
    volumes:
      - ./src:/app/src
    ports:
      - "3000:3000"
      - "9229:9229"    # debugger do Node.js
    environment:
      NODE_ENV: development
    command: tsx watch src/index.ts

  db:
    ports:
      - "5432:5432"    # expor ao host para ferramentas de DB
```

`compose.prod.yml` (produção):
```yaml
services:
  api:
    image: myapi:${VERSION:-latest}
    ports:
      - "127.0.0.1:3000:3000"   # apenas loopback, nginx faz o proxy
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '1.0'

  db:
    # Sem exposição de porta em produção
    restart: unless-stopped
```

Uso:
```bash
# Desenvolvimento (compose.yml + compose.override.yml mesclados automaticamente)
docker compose up

# Produção (seleção explícita de arquivos)
docker compose -f compose.yml -f compose.prod.yml up -d
```

### Exemplo 3: Migrations de Banco como Init Container

```yaml
services:
  migrate:
    image: myapi:latest
    command: node dist/migrate.js
    environment:
      DATABASE_URL: postgres://app:secret@db:5432/app
    depends_on:
      db:
        condition: service_healthy
    restart: "no"   # executar uma vez e sair
    networks:
      - backend

  api:
    image: myapi:latest
    depends_on:
      migrate:
        condition: service_completed_successfully
    ports:
      - "3000:3000"
    networks:
      - backend

  db:
    image: postgres:16-alpine
    # ... config de healthcheck
    networks:
      - backend
```

O Compose aguarda `migrate` sair com código 0 antes de iniciar `api`.

### Exemplo 4: Ambiente de Desenvolvimento Multi-Service com Ferramentas

```yaml
name: myapp-dev

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile.dev
    volumes:
      - .:/app
      - node_modules:/app/node_modules
    ports:
      - "3000:3000"
      - "9229:9229"
    environment:
      DATABASE_URL: postgres://app:dev@db:5432/app
      REDIS_URL: redis://cache:6379
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: app
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 3s
      retries: 5

  cache:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  mailpit:
    image: axllent/mailpit:latest
    ports:
      - "1025:1025"    # SMTP
      - "8025:8025"    # Web UI
    environment:
      MP_SMTP_AUTH_ALLOW_INSECURE: true

  pgadmin:
    image: dpage/pgadmin4:latest
    ports:
      - "5050:80"
    environment:
      PGADMIN_DEFAULT_EMAIL: dev@local.dev
      PGADMIN_DEFAULT_PASSWORD: dev

volumes:
  pgdata:
  node_modules:
```

Isso entrega: API com hot reload, Postgres, Redis, captura de e-mails (Mailpit) e uma GUI de banco (pgAdmin) — tudo com um único `docker compose up`.

---

## Padrões e Boas Práticas

### Sempre Use Healthchecks

`depends_on` sem `condition: service_healthy` apenas aguarda o container iniciar, não o service interno estar pronto. O Postgres leva alguns segundos para aceitar conexões após a inicialização.

```yaml
db:
  image: postgres:16-alpine
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
    interval: 5s
    timeout: 3s
    retries: 10
    start_period: 10s  # período de graça antes da primeira verificação
```

### Use Volumes Nomeados para Dados Persistentes

```yaml
volumes:
  pgdata:        # Docker gerencia a localização
  redisdata:

services:
  db:
    volumes:
      - pgdata:/var/lib/postgresql/data
```

Volumes nomeados sobrevivem ao `docker compose down`. Use `docker compose down -v` apenas quando quiser apagar os dados.

### Separe Arquivos .env por Ambiente

```
.env              # padrões (seguro para commitar — sem secrets)
.env.local        # overrides locais (no .gitignore)
.env.production   # valores de produção (gerenciados via secrets manager)
```

```yaml
# compose.yml
env_file:
  - .env
  - .env.local    # sobrescreve .env
```

### Services Condicionais por Profile

```yaml
services:
  api:
    # sempre roda
    image: myapi:latest

  pgadmin:
    profiles: [tools]       # roda apenas com --profile tools
    image: dpage/pgadmin4

  mailpit:
    profiles: [tools]
    image: axllent/mailpit
```

```bash
# Inicialização normal — apenas api
docker compose up -d

# Com ferramentas de desenvolvimento
docker compose --profile tools up -d
```

### Fixe Versões de Imagens

```yaml
# RUIM
image: postgres:latest

# BOM
image: postgres:16.2-alpine3.19
```

---

## Anti-padrões a Evitar

### Secrets Hardcoded no compose.yml

```yaml
# RUIM — commitado no git
environment:
  DATABASE_URL: postgres://root:HARDCODED_SECRET@db/app

# BOM — do ambiente ou arquivo .env (no .gitignore)
environment:
  DATABASE_URL: postgres://app:${POSTGRES_PASSWORD}@db/app
```

### Sem Healthchecks em Bancos de Dados

Sem healthchecks, `depends_on` é essencialmente inútil para bancos de dados — sua API inicia antes do Postgres estar pronto e falha com "connection refused."

### Armazenar node_modules em Bind Mounts

```yaml
# RUIM — node_modules do host vaza para o container ou vice-versa
volumes:
  - .:/app

# BOM — volume anônimo para node_modules oculta o diretório do host
volumes:
  - .:/app
  - /app/node_modules   # volume anônimo vazio mascara o bind mount
```

Ou ainda melhor — use um volume nomeado:
```yaml
volumes:
  - .:/app
  - nodemodules:/app/node_modules

volumes:
  nodemodules:
```

### Rodar Migrations Dentro do Service de API na Inicialização

Misturar "rodar migrations" com "iniciar servidor" no CMD cria race conditions quando você escala para múltiplas réplicas. Use um service `migrate` separado ou um hook de pré-deploy.

### Usar `docker-compose` (v1) em vez de `docker compose` (v2)

A v1 (`docker-compose`) está depreciada. A v2 está integrada ao CLI do Docker e é significativamente mais rápida. Use `docker compose` (com espaço).

---

## Debug & Troubleshooting

### Verificar Status dos Services

```bash
docker compose ps
docker compose ps --format json | jq '.[] | {name, state, health}'
```

### Ver Logs

```bash
docker compose logs                     # todos os services
docker compose logs api                 # service único
docker compose logs api db              # múltiplos services
docker compose logs -f --tail 50 api    # seguir últimas 50 linhas
docker compose logs --since 5m          # últimos 5 minutos
```

### Entrar em um Service

```bash
docker compose exec api sh
docker compose exec db psql -U app -d app
docker compose exec cache redis-cli
```

### Executar Comandos Únicos

```bash
# Roda um novo container e depois o remove
docker compose run --rm api node dist/seed.js
docker compose run --rm api sh -c "node dist/migrate.js && node dist/seed.js"
```

### Inspecionar Conectividade de Rede

```bash
# De dentro do container api, consegue alcançar db?
docker compose exec api sh
# Dentro do shell:
wget -qO- http://db:5432 || echo "can't reach"
nc -zv db 5432
```

### Resetar Tudo

```bash
docker compose down -v --remove-orphans  # parar, remover containers+volumes+orphans
docker compose build --no-cache          # forçar rebuild completo
docker compose up -d
```

---

## Cenários do Mundo Real

### Cenário 1: Deploy com Zero Downtime e Health Checks

```yaml
services:
  api:
    image: myapi:${VERSION}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 20s
    restart: unless-stopped
```

Com um reverse proxy (nginx) na frente, você pode fazer o deploy de uma nova versão ao lado da antiga, aguardar os healthchecks passarem e então chavear o tráfego.

### Cenário 2: Seed de Banco de Dados de Desenvolvimento

```yaml
services:
  db-seed:
    image: myapi:latest
    command: node dist/seed.js
    profiles: [seed]
    depends_on:
      db:
        condition: service_healthy
    environment:
      DATABASE_URL: postgres://app:dev@db/app
    restart: "no"
```

```bash
# Fazer seed após o primeiro start
docker compose --profile seed run --rm db-seed
```

### Cenário 3: Ambiente de CI

`compose.ci.yml`:
```yaml
services:
  test:
    build:
      context: .
      target: builder
    command: npm test
    environment:
      DATABASE_URL: postgres://app:ci@db/app_test
      NODE_ENV: test
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ci
      POSTGRES_DB: app_test
    tmpfs:
      - /var/lib/postgresql/data   # DB em memória para velocidade no CI
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 3s
      timeout: 3s
      retries: 10
```

```bash
# No pipeline de CI
docker compose -f compose.ci.yml run --rm test
docker compose -f compose.ci.yml down -v
```

---

## Leitura Complementar

- [Compose Specification](https://compose-spec.io/)
- [Referência do CLI do Docker Compose](https://docs.docker.com/compose/reference/)
- [Referência do arquivo Compose](https://docs.docker.com/compose/compose-file/)
- [Configurações multi-ambiente](https://docs.docker.com/compose/extends/)

---

## Resumo

| Conceito | Ponto Principal |
|---------|-----------------|
| `docker compose up -d` | Iniciar todos os services em detached |
| `depends_on` + healthcheck | Aguardar o banco realmente pronto, não apenas iniciado |
| Arquivos de override | Compartilhe config base; sobrescreva por ambiente |
| Volumes nomeados | Persistir dados após `docker compose down` |
| Profiles | Services condicionais (tools, seed, etc.) |
| `env_file` | Separar secrets do arquivo compose |
| Comandos únicos | `docker compose run --rm service comando` |
| Migrations | Service `migrate` separado, não dentro do CMD |
| CI | Use `tmpfs` para volumes de banco e acelere os testes |

O Compose é a base do desenvolvimento local e uma solução de produção válida para deploys em host único. Aprenda-o bem antes de partir para Kubernetes — a maioria das aplicações não precisa de um orquestrador.
