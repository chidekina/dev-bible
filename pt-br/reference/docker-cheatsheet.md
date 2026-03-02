# Docker Cheatsheet

> Referência rápida — use Ctrl+F para encontrar o que precisa.

---

## Comandos de Image

```bash
docker images                         # listar images locais
docker pull nginx:alpine              # baixar do registry
docker build -t myapp:1.0 .           # build a partir do Dockerfile
docker build -t myapp:1.0 -f Dockerfile.prod .
docker tag myapp:1.0 registry/myapp:1.0
docker push registry/myapp:1.0
docker rmi myapp:1.0                  # remover image
docker image prune                    # remover images sem tag
docker image prune -a                 # remover todas as images não utilizadas
docker history myapp:1.0              # histórico de layers
docker inspect myapp:1.0             # metadados completos em JSON
```

---

## Comandos de Container

```bash
# Executar
docker run nginx                                # em primeiro plano
docker run -d nginx                             # em segundo plano (detached)
docker run -it ubuntu bash                      # TTY interativo
docker run --rm ubuntu echo hello               # remove automaticamente ao sair
docker run -p 8080:80 nginx                     # porta host:container
docker run -p 127.0.0.1:8080:80 nginx          # vincular apenas ao localhost
docker run -e ENV_VAR=value nginx               # variável de ambiente
docker run --env-file .env nginx                # arquivo de variáveis
docker run -v /host/path:/container/path nginx  # bind mount
docker run -v myvolume:/data nginx              # volume nomeado
docker run --name mycontainer nginx             # container nomeado
docker run --network mynet nginx                # rede customizada
docker run --memory 512m --cpus 1.5 nginx      # limites de recursos
docker run --restart unless-stopped nginx       # política de restart

# Ciclo de vida
docker start mycontainer
docker stop mycontainer           # SIGTERM, depois SIGKILL após 10s
docker stop -t 30 mycontainer     # timeout customizado
docker restart mycontainer
docker kill mycontainer           # SIGKILL imediato
docker pause mycontainer
docker unpause mycontainer
docker rm mycontainer             # remover container parado
docker rm -f mycontainer          # forçar remoção de container em execução

# Informações
docker ps                         # containers em execução
docker ps -a                      # todos os containers
docker logs mycontainer           # stdout/stderr
docker logs -f mycontainer        # acompanhar logs
docker logs --tail 100 mycontainer
docker logs --since 10m mycontainer
docker stats                      # uso de recursos em tempo real
docker top mycontainer            # processos dentro do container
docker inspect mycontainer        # metadados completos em JSON
docker diff mycontainer           # alterações no sistema de arquivos

# Executar comandos
docker exec mycontainer ls /app
docker exec -it mycontainer bash      # shell interativo
docker exec -u root mycontainer whoami

# Copiar arquivos
docker cp mycontainer:/app/logs ./logs    # container → host
docker cp ./config.json mycontainer:/app  # host → container
```

---

## Diretivas do Dockerfile

```dockerfile
# Image base
FROM node:22-alpine AS base

# Argumentos de build (disponíveis em tempo de build)
ARG NODE_ENV=production

# Variáveis de ambiente (disponíveis em tempo de execução)
ENV PORT=3000 \
    NODE_ENV=production

# Diretório de trabalho
WORKDIR /app

# Copiar arquivos (COPY <origem> <destino>)
COPY package*.json ./
COPY --chown=node:node . .

# Executar comandos (cria uma nova layer)
RUN npm ci --only=production

# Expor porta (apenas documentação — não publica a porta)
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# Ponto de montagem de volume
VOLUME ["/data"]

# Usuário para executar (nunca execute como root em produção)
USER node

# Comando padrão (pode ser sobrescrito)
CMD ["node", "dist/index.js"]

# Entrypoint (não é facilmente sobrescrito — use para executáveis)
ENTRYPOINT ["docker-entrypoint.sh"]
```

### Build Multi-Stage

```dockerfile
# Estágio 1: build
FROM node:22 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Estágio 2: runtime (image enxuta)
FROM node:22-alpine AS runtime
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
USER node
CMD ["node", "dist/index.js"]
```

### .dockerignore

```
node_modules
.git
.env
*.log
dist
coverage
.DS_Store
```

---

## Docker Compose

```yaml
# docker-compose.yml
version: "3.9"

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
      target: runtime
    image: myapp:latest
    container_name: myapp-api
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: production
      DATABASE_URL: postgresql://user:pass@db:5432/mydb
    env_file:
      - .env
    volumes:
      - ./uploads:/app/uploads
    depends_on:
      db:
        condition: service_healthy
    networks:
      - internal
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
    deploy:
      resources:
        limits:
          memory: 512m
          cpus: "1.0"

  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - internal
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "user"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  pgdata:

networks:
  internal:
    driver: bridge
```

### Comandos do Compose

```bash
docker compose up                    # iniciar todos os serviços
docker compose up -d                 # em segundo plano
docker compose up --build            # rebuild das images primeiro
docker compose up api                # iniciar serviço específico
docker compose down                  # parar e remover containers
docker compose down -v               # também remover volumes
docker compose ps                    # listar serviços
docker compose logs                  # todos os logs
docker compose logs -f api           # acompanhar serviço específico
docker compose exec api bash         # shell no serviço
docker compose run --rm api npm test # comando avulso
docker compose pull                  # baixar images mais recentes
docker compose build                 # build de todas as images
docker compose restart api           # reiniciar serviço
docker compose stop                  # parar sem remover
docker compose config                # validar e imprimir configuração
```

---

## Redes

```bash
# Redes
docker network ls
docker network create mynet
docker network create --driver bridge --subnet 172.20.0.0/16 mynet
docker network inspect mynet
docker network connect mynet mycontainer
docker network disconnect mynet mycontainer
docker network rm mynet
docker network prune                 # remover redes não utilizadas

# Conectar dois containers
docker run -d --name db --network mynet postgres
docker run -d --name api --network mynet myapp
# api alcança db pelo hostname "db"
```

### Drivers de Rede

| Driver | Caso de Uso |
|--------|-------------|
| `bridge` | Padrão — containers no mesmo host comunicam-se via switch virtual |
| `host` | Container compartilha a pilha de rede do host (apenas Linux) |
| `none` | Sem rede |
| `overlay` | Multi-host (Docker Swarm / Compose com modo Swarm) |
| `macvlan` | Container recebe seu próprio MAC/IP na rede física |

---

## Volumes

```bash
docker volume ls
docker volume create myvolume
docker volume inspect myvolume
docker volume rm myvolume
docker volume prune                  # remover volumes não utilizados

# Backup de um volume
docker run --rm \
  -v myvolume:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/myvolume.tar.gz /data

# Restaurar um volume
docker run --rm \
  -v myvolume:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/myvolume.tar.gz -C /
```

---

## Registry e Gerenciamento de Images

```bash
# Login
docker login
docker login registry.example.com

# Tag e push
docker tag myapp:latest registry.example.com/myapp:1.0
docker push registry.example.com/myapp:1.0

# Salvar / carregar (sem registry)
docker save myapp:1.0 | gzip > myapp.tar.gz
docker load < myapp.tar.gz

# Exportar / importar (sistema de arquivos do container)
docker export mycontainer | gzip > container.tar.gz
docker import container.tar.gz myimage:restored
```

---

## Limpeza

```bash
# Remover tudo não utilizado (seguro)
docker system prune

# Remover tudo, incluindo volumes
docker system prune --volumes -a

# Limpeza seletiva
docker container prune     # containers parados
docker image prune -a      # images não utilizadas
docker volume prune        # volumes não utilizados
docker network prune       # redes não utilizadas

# Verificar uso de disco
docker system df
docker system df -v        # detalhado
```

---

## Padrões de Produção

```bash
# Sempre fixe versões das images (nunca use :latest em produção)
FROM postgres:16.2-alpine

# Sistema de arquivos somente-leitura + tmpfs para escritas
docker run --read-only --tmpfs /tmp myapp

# Remover capabilities
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE myapp

# Sem novos privilégios
docker run --security-opt no-new-privileges myapp

# Usuário não-root (definido no Dockerfile)
USER 1001

# Limites de recursos
docker run --memory 256m --memory-swap 256m --cpus 0.5 myapp

# Política de restart para produção
restart: unless-stopped   # ou: always, on-failure:5
```

---

## One-Liners Úteis

```bash
# Parar todos os containers em execução
docker stop $(docker ps -q)

# Remover todos os containers parados
docker rm $(docker ps -aq -f status=exited)

# Obter IP do container
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' mycontainer

# Acompanhar logs com timestamps
docker logs -f --timestamps mycontainer

# Copiar arquivo de uma image (sem executar)
docker create --name tmp myapp && docker cp tmp:/app/config.json . && docker rm tmp

# Shell no container em execução como root
docker exec -u root -it mycontainer sh

# Ver estatísticas dos containers uma vez
docker stats --no-stream
```
