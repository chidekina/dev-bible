# Docker Fundamentals

## Visão Geral

Docker é uma plataforma de containerização que empacota aplicações e suas dependências em unidades portáteis e isoladas chamadas containers. Ao contrário das máquinas virtuais, os containers compartilham o kernel do sistema operacional do host, o que os torna leves, rápidos para iniciar e consistentes entre ambientes.

A promessa central do Docker: **"funciona na minha máquina" vira "funciona em qualquer lugar."** Você define o ambiente exato que sua aplicação precisa — bibliotecas do SO, versão do runtime, configuração — e o Docker garante que esse ambiente seja idêntico no desenvolvimento, CI, staging e produção.

Este capítulo cobre o Docker desde os primeiros princípios: imagens, containers, o sistema de build, networking, volumes e os modelos mentais que você precisa para usar Docker efetivamente em produção.

---

## Pré-requisitos

- Linha de comando Linux básica (veja `01-foundations/linux-shell.md`)
- Entendimento básico de processos e sistemas de arquivos
- Familiaridade com Node.js/TypeScript para os exemplos

Instalar Docker:
```bash
# Ubuntu/Debian
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker

# Verificar
docker --version
docker run hello-world
```

---

## Conceitos Fundamentais

### Imagens vs Containers

Uma **imagem** é um template somente-leitura — um snapshot em camadas do sistema de arquivos com metadados. Um **container** é uma instância em execução de uma imagem, com uma camada gravável por cima.

```
Imagem (camadas somente-leitura, de baixo para cima):
  Camada 4: COPY . .          ← código da sua aplicação
  Camada 3: RUN npm install   ← node_modules
  Camada 2: FROM node:22      ← runtime do Node.js
  Camada 1: FROM debian:slim  ← base do SO (baixada do Docker Hub)

Container = Camadas da imagem + camada gravável + PID namespace + network namespace
```

Múltiplos containers podem rodar a partir da mesma imagem simultaneamente. Cada um tem sua própria camada gravável isolada, mas todos compartilham as camadas somente-leitura da imagem. É por isso que containers iniciam em milissegundos — sem boot completo do SO, sem copiar camadas.

### O Union Filesystem

Docker usa overlay filesystems (OverlayFS no Linux moderno). Cada instrução em um Dockerfile cria uma nova camada somente-leitura. As camadas são endereçadas por conteúdo (SHA256) e cacheadas — se o conteúdo da camada não mudou, o Docker a reutiliza durante os builds.

```
Regras de cache de camadas:
- FROM        → cacheado a menos que o digest da imagem base mude
- RUN         → cacheado se o comando for idêntico
- COPY/ADD    → cacheado se o checksum do conteúdo do arquivo for idêntico
- ARG         → invalida todas as camadas subsequentes se o valor mudar
```

Entender a invalidação de cache é fundamental para builds rápidos. Uma vez que uma camada é invalidada, todas as camadas subsequentes são reconstruídas.

### O Dockerfile

Um Dockerfile é um script ordenado que define como construir uma imagem. Cada instrução vira uma camada:

```dockerfile
# Dockerfile Node.js mínimo
FROM node:22-alpine

WORKDIR /app

# Copiar manifests primeiro — cacheado até package.json mudar
COPY package*.json ./

# Instalar deps — cacheado se os manifests não mudaram
RUN npm ci --only=production

# Copiar o código da aplicação — sempre re-executa em mudanças de código
COPY src/ ./src/

EXPOSE 3000

USER node

CMD ["node", "src/index.js"]
```

### Namespaces e Cgroups

Containers não são VMs — são processos Linux com restrições:

| Mecanismo | O que isola |
|-----------|-------------|
| PID namespace | Árvore de processos (o container não vê processos do host) |
| Network namespace | Interfaces de rede, roteamento, portas |
| Mount namespace | Visão do sistema de arquivos |
| UTS namespace | Hostname e nome de domínio |
| User namespace | Mapeamento de UID/GID |
| Cgroups | Limites de CPU, memória e I/O de disco |
| Seccomp | Chamadas de sistema permitidas |

### Ciclo de Vida do Container

```
docker create → Criado
docker start  → Rodando ←→ Pausado (docker pause/unpause)
docker stop   → Parado (SIGTERM + 10s de graça + SIGKILL)
docker kill   → Parado (SIGKILL imediato)
docker rm     → Removido
```

```bash
docker run nginx             # create + start (mais comum)
docker run -d nginx          # detached (background)
docker run --rm nginx        # remove o container quando sair
docker stop mycontainer      # parada graciosa
docker rm mycontainer        # remove container parado
docker rm -f mycontainer     # força parada + remoção
```

---

## Exemplos Práticos

### Exemplo 1: Containerizando uma API Node.js

Estrutura do projeto:
```
myapi/
  src/
    index.ts
  package.json
  tsconfig.json
  Dockerfile
  .dockerignore
```

`src/index.ts`:
```typescript
import Fastify from 'fastify';

const app = Fastify({ logger: true });

app.get('/health', async () => ({
  status: 'ok',
  uptime: process.uptime(),
  version: process.env.APP_VERSION ?? 'dev',
}));

app.get('/users', async () => [
  { id: 1, name: 'Alice' },
  { id: 2, name: 'Bob' },
]);

const start = async () => {
  await app.listen({ port: 3000, host: '0.0.0.0' });
};

start().catch(console.error);
```

`.dockerignore` (crie antes do seu primeiro COPY):
```
node_modules
dist
.git
*.log
.env
.env.*
coverage
.nyc_output
*.test.ts
*.spec.ts
```

`Dockerfile` usando build multi-stage:
```dockerfile
# ─── Estágio 1: Build ───────────────────────────────────────────────
FROM node:22-alpine AS builder

WORKDIR /app

COPY package*.json tsconfig.json ./
RUN npm ci

COPY src/ ./src/
RUN npm run build

# ─── Estágio 2: Produção ──────────────────────────────────────────
FROM node:22-alpine AS production

# Segurança: usuário não-root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

COPY --from=builder /app/dist ./dist

RUN chown -R appuser:appgroup /app

USER appuser

EXPOSE 3000

# Orquestradores usam isso para decidir quando o container está saudável
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "dist/index.js"]
```

Build e execução:
```bash
# Build
docker build -t myapi:latest .

# Rodar com limites de recursos
docker run -d \
  --name myapi \
  -p 3000:3000 \
  -e APP_VERSION=1.0.0 \
  --memory=256m \
  --cpus=0.5 \
  --restart=unless-stopped \
  myapi:latest

# Verificar
curl http://localhost:3000/health

# Logs
docker logs -f myapi
```

### Exemplo 2: Variáveis de Ambiente

```bash
# Injetar uma variável
docker run -e DATABASE_URL="postgres://user:pass@host/db" myapi:latest

# Injetar de arquivo (uma KEY=VALUE por linha, sem aspas)
docker run --env-file .env.production myapi:latest

# Repassar do ambiente do host
export DATABASE_URL="postgres://..."
docker run -e DATABASE_URL myapi:latest   # passa o valor de $DATABASE_URL
```

Nunca insira secrets nas imagens — eles aparecem no `docker history` e nos registries.

### Exemplo 3: Volumes

```bash
# Volume nomeado — Docker gerencia a localização no host
docker volume create appdata
docker run -v appdata:/app/data myapi:latest

# Bind mount — caminho exato do host mapeado para o container
docker run -v $(pwd)/uploads:/app/uploads myapi:latest

# Bind mount somente-leitura — o container não pode modificar
docker run -v $(pwd)/config:/app/config:ro myapi:latest

# tmpfs — em memória, não persiste, ideal para secrets/arquivos temporários
docker run --tmpfs /tmp:size=100m myapi:latest
```

Inspecionar volumes:
```bash
docker volume ls
docker volume inspect appdata
docker volume rm appdata
```

### Exemplo 4: Networking

```bash
# Criar rede isolada definida pelo usuário
docker network create backend

# Containers na mesma rede se resolvem pelo nome do service
docker run -d --name postgres --network backend \
  -e POSTGRES_PASSWORD=secret postgres:16-alpine

docker run -d --name myapi --network backend \
  -p 3000:3000 \
  -e DATABASE_URL="postgres://postgres:secret@postgres:5432/app" \
  myapi:latest

# postgres é alcançável dentro de myapi pelo hostname "postgres"
```

Tipos de rede:
| Modo | Descrição |
|------|-----------|
| `bridge` | Padrão. Rede NAT isolada por container |
| Bridge definida pelo usuário | Rede customizada; containers se resolvem por nome |
| `host` | Compartilha o stack de rede do host. Sem isolamento, mais rápido |
| `none` | Sem networking |

---

## Padrões e Boas Práticas

### Builds Multi-Stage

Builds multi-stage produzem imagens de produção pequenas descartando as ferramentas de build:

```dockerfile
FROM node:22 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build && npm run test

FROM node:22-alpine AS production
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY --from=builder /app/dist ./dist
USER node
CMD ["node", "dist/index.js"]
```

Resultado: imagem do builder ~1,2 GB, imagem de produção ~120 MB.

### Ordem de Camadas para Máximo Cache

```dockerfile
# RUIM — mudança de código invalida o npm install
COPY . .
RUN npm ci

# BOM — npm install é cacheado até package.json mudar
COPY package*.json ./
RUN npm ci
COPY . .
```

A regra: copie primeiro o que muda com menos frequência.

### Usuários Não-Root

```dockerfile
# Alpine
RUN addgroup -S app && adduser -S app -G app
USER app

# Debian/Ubuntu
RUN groupadd --system app && useradd --system --gid app --no-create-home app
USER app
```

### HEALTHCHECK

Sempre defina um healthcheck. Orquestradores (Docker Compose, Swarm, Kubernetes) o usam para saber quando um container está pronto e para reiniciar os que estão com problemas:

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

### Tagging de Imagens

```bash
# Vincule as tags de imagem a refs do git para rastreabilidade
GIT_SHA=$(git rev-parse --short HEAD)
docker build \
  -t myapi:${GIT_SHA} \
  -t myapi:latest \
  .
```

### Limites de Recursos

```bash
docker run \
  --memory=512m \              # limite rígido de memória — morto por OOM se exceder
  --memory-reservation=256m \ # limite suave para agendamento
  --cpus=1.0 \                 # 1.0 = um núcleo de CPU completo
  --pids-limit=100 \           # previne fork bombs
  myapi:latest
```

---

## Anti-padrões a Evitar

### Rodar como Root

Um container comprometido rodando como root dá ao atacante capacidades equivalentes a root. Sempre troque para um usuário não-root antes do CMD.

### Secrets em Imagens

```dockerfile
# RUIM — visível no docker history, enviado ao registry
ENV DATABASE_URL="postgres://admin:secret@prod-db/app"

# BOM — injetado em runtime, nunca armazenado na imagem
# docker run -e DATABASE_URL=... ou --env-file
```

### Pinar em `latest`

```yaml
# RUIM — imprevisível, pode quebrar em qualquer deploy
image: postgres:latest

# BOM — determinístico
image: postgres:16.2-alpine
```

### Imagens Pesadas

```dockerfile
# RUIM
RUN apt-get install -y build-essential curl wget vim git

# BOM — instale apenas o que o container precisa em runtime
RUN apt-get install -y --no-install-recommends curl \
  && rm -rf /var/lib/apt/lists/*
```

### Sem .dockerignore

Sem `.dockerignore`, `COPY . .` envia `node_modules`, `.git`, `.env` e logs para o daemon de build, inflando o contexto de build e potencialmente vazando secrets.

### Problema do PID 1 com Sinais

Node.js não lida bem com SIGTERM quando é PID 1. Use `tini`:

```dockerfile
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "dist/index.js"]
```

Ou trate os sinais na sua aplicação:
```typescript
process.on('SIGTERM', () => {
  server.close(() => {
    console.log('Graceful shutdown completo');
    process.exit(0);
  });
});
```

---

## Debug & Troubleshooting

### Abrir um Shell Dentro de um Container em Execução

```bash
docker exec -it myapi sh      # alpine (sem bash)
docker exec -it myapi bash    # debian/ubuntu
docker exec myapi env         # imprimir variáveis de ambiente
docker exec myapi cat /etc/hosts
```

### Ler Logs

```bash
docker logs myapi               # todos os logs
docker logs myapi --tail 50     # últimas 50 linhas
docker logs myapi -f            # seguir em tempo real
docker logs myapi --since 10m   # últimos 10 minutos
docker logs myapi 2>&1 | grep ERROR
```

### Inspecionar Metadados

```bash
docker inspect myapi                                    # JSON completo
docker inspect myapi | jq '.[0].State'                 # info de estado
docker inspect myapi | jq '.[0].HostConfig.Memory'     # limite de memória
docker inspect myapi | jq '.[0].NetworkSettings'       # info de rede
```

### Debugar um Container Parado

```bash
docker ps -a                          # listar todos incluindo parados
docker logs <container-id>            # última saída antes do crash
docker inspect <id> | jq '.[0].State.ExitCode'

# Códigos de saída comuns:
# 1   — erro da aplicação (verifique os logs)
# 126 — comando não executável (permissão negada)
# 127 — comando não encontrado (caminho do CMD errado)
# 137 — SIGKILL / morto por OOM (aumente --memory)
# 143 — SIGTERM (parada limpa)
```

### Analisar Camadas da Imagem

```bash
docker history myapi:latest            # tamanhos das camadas
docker history --no-trunc myapi:latest # comandos completos

# Instale dive para inspeção detalhada de camadas
# https://github.com/wagoodman/dive
dive myapi:latest
```

### Recuperar Espaço em Disco

```bash
docker system df              # mostrar uso de disco
docker system prune           # remover containers parados + imagens dangling
docker system prune -a        # também remover imagens não usadas
docker volume prune           # remover volumes não usados
```

---

## Cenários do Mundo Real

### Cenário 1: Desenvolvimento com Hot Reload

```dockerfile
# Dockerfile.dev
FROM node:22-alpine
WORKDIR /app
RUN npm install -g tsx
COPY package*.json ./
RUN npm install
# O código é montado via bind mount em runtime — não faça COPY src
CMD ["tsx", "watch", "src/index.ts"]
```

```bash
docker run -it --rm \
  -v $(pwd)/src:/app/src:ro \
  -p 3000:3000 \
  myapi:dev
```

Mudanças em `src/` no host são imediatamente refletidas no container sem rebuild.

### Cenário 2: Comandos Únicos (Migrations)

```bash
# Rodar migration e sair
docker run --rm \
  -e DATABASE_URL="$DATABASE_URL" \
  myapi:latest \
  node dist/migrate.js

# Abrir um REPL contra o banco de dados de produção (réplica somente-leitura)
docker run --rm -it \
  -e DATABASE_URL="$REPLICA_URL" \
  myapi:latest \
  node -e "require('./dist/repl')"
```

### Cenário 3: Argumentos de Build para Múltiplos Ambientes

```dockerfile
ARG APP_ENV=production
ENV APP_ENV=${APP_ENV}

RUN if [ "$APP_ENV" = "staging" ]; then \
      cp config.staging.json config.json; \
    else \
      cp config.production.json config.json; \
    fi
```

```bash
docker build --build-arg APP_ENV=staging -t myapi:staging .
docker build --build-arg APP_ENV=production -t myapi:prod .
```

### Cenário 4: Passando Credenciais NPM com Segurança

```dockerfile
# syntax=docker/dockerfile:1.4
FROM node:22-alpine AS builder
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm install
```

```bash
docker build --secret id=npmrc,src=$HOME/.npmrc .
```

O `.npmrc` é montado apenas durante o passo `RUN` e nunca armazenado em nenhuma camada.

---

## Leitura Complementar

- [Boas Práticas de Dockerfile — Docker Docs](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [dive — explorador de camadas de imagem](https://github.com/wagoodman/dive)
- [tini — init mínimo para containers](https://github.com/krallin/tini)
- [Docker Security Cheat Sheet — OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)
- [Documentação do BuildKit](https://docs.docker.com/build/buildkit/)

---

## Resumo

| Conceito | Ponto Principal |
|---------|-----------------|
| Imagens | Snapshots em camadas, somente-leitura, endereçados por conteúdo. Compartilhados entre containers. |
| Containers | Imagem em execução + camada gravável + namespaces Linux isolados |
| Dockerfile | Instruções ordenadas. A ordem determina a eficiência do cache. |
| Builds multi-stage | Mantenha imagens de produção pequenas — ferramentas de build ficam no estágio builder |
| Usuários não-root | Sempre rode como não-root em produção |
| HEALTHCHECK | Defina um para que orquestradores possam gerenciar o ciclo de vida do container |
| Limites de recursos | Sempre defina `--memory` e `--cpus` em produção |
| .dockerignore | Crie antes do seu primeiro COPY |
| Secrets | Injete em runtime — nunca insira nas camadas da imagem |
| PID 1 | Use tini ou trate SIGTERM explicitamente para habilitar graceful shutdown |

Docker elimina o "funciona na minha máquina" tornando a máquina parte do artefato. Domine o cache de camadas, mantenha imagens de produção mínimas e nunca rode como root.
