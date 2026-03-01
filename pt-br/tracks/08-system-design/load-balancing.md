# Load Balancing

## Visão Geral

Load balancing distribui requisições recebidas entre múltiplos servidores para maximizar o throughput, minimizar a latência e evitar sobrecarregar um único nó. É o mecanismo que permite escalonamento horizontal — adicionar mais servidores para lidar com mais tráfego. Sem um load balancer, você tem um único ponto de falha e um teto artificial de tráfego. Este capítulo cobre algoritmos de load balancing, health checks, afinidade de sessão e a diferença entre load balancing na camada 4 e na camada 7.

---

## Pré-requisitos

- Fundamentos de rede (TCP/IP, HTTP)
- Familiaridade com Docker e reverse proxies
- Track 05 — DevOps: containers e conceitos de nginx

---

## Conceitos Fundamentais

### Camada 4 vs Camada 7

**Camada 4 (Transporte):** Load balancing em nível TCP/UDP. O balancer roteia pacotes com base em endereço IP e porta, sem inspecionar o conteúdo HTTP. Rápido, baixo overhead, mas não pode tomar decisões de roteamento baseadas em conteúdo.

**Camada 7 (Aplicação):** Load balancing em nível HTTP. O balancer inspeciona o conteúdo da requisição (URL, headers, cookies) e pode rotear com base em path, host ou identidade do usuário. Mais flexível, levemente mais overhead.

| Característica | Camada 4 | Camada 7 |
|----------------|----------|----------|
| Base de roteamento | IP + porta | URL, host, headers |
| Terminação TLS | Pass-through ou terminar | Terminar (e re-encriptar) |
| Sticky sessions | Por IP | Por cookie |
| Inspeção de conteúdo | Não | Sim |
| Exemplos | AWS NLB, HAProxy (TCP) | Nginx, Traefik, AWS ALB |

### Algoritmos de load balancing

| Algoritmo | Como funciona | Ideal para |
|-----------|--------------|-----------|
| Round Robin | Requisições ciclam pelos servidores em ordem | Servidores homogêneos, apps sem estado |
| Round Robin Ponderado | Como round robin, mas alguns servidores recebem mais requisições | Capacidade heterogênea de servidores |
| Least Connections | Próxima requisição vai ao servidor com menos conexões ativas | Requisições de duração variável |
| Menor Tempo de Resposta | Roteia ao servidor com menor latência | Aplicações sensíveis à latência |
| IP Hash | Faz hash do IP do cliente para determinar o servidor | Sticky sessions sem cookies |
| Aleatório | Escolhe um servidor aleatório | Simples, surpreendentemente eficaz |

---

## Exemplos Práticos

### Nginx como load balancer HTTP

```nginx
# /etc/nginx/nginx.conf
http {
    upstream api_servers {
        # Padrão: round robin
        server api-1:3000;
        server api-2:3000;
        server api-3:3000;

        # Health check (recurso do nginx plus; para open source, verifique externamente)
        # keepalive 32; — reutiliza conexões com os backends
    }

    server {
        listen 80;

        location / {
            proxy_pass http://api_servers;
            proxy_http_version 1.1;
            proxy_set_header Connection "";           # habilita keepalive para upstreams
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # Timeouts
            proxy_connect_timeout 5s;
            proxy_read_timeout 30s;
        }
    }
}
```

**Algoritmo least connections:**

```nginx
upstream api_servers {
    least_conn;
    server api-1:3000;
    server api-2:3000;
    server api-3:3000;
}
```

**Round robin ponderado:**

```nginx
upstream api_servers {
    server api-1:3000 weight=3;  # recebe 3x mais requisições
    server api-2:3000 weight=1;
}
```

**IP hash (sticky sessions):**

```nginx
upstream api_servers {
    ip_hash;
    server api-1:3000;
    server api-2:3000;
    server api-3:3000;
    # O mesmo IP de cliente sempre roteia para o mesmo servidor
}
```

### Health checks passivos no Nginx

```nginx
upstream api_servers {
    server api-1:3000 max_fails=3 fail_timeout=30s;
    server api-2:3000 max_fails=3 fail_timeout=30s;
    # Após 3 falhas em 30s, o servidor é removido por 30s
}
```

### Traefik como load balancer (Docker)

O Traefik descobre serviços backend via labels do Docker:

```yaml
# docker-compose.yml
services:
  api:
    image: myapp:latest
    deploy:
      replicas: 3       # 3 instâncias da API
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.api.rule=Host(`api.example.com`)"
      - "traefik.http.services.api.loadbalancer.server.port=3000"
      # Padrão: round robin entre as 3 instâncias

  traefik:
    image: traefik:v3.0
    command:
      - "--providers.docker=true"
      - "--providers.docker.swarmMode=true"  # para Docker Swarm
      - "--entrypoints.web.address=:80"
    ports:
      - "80:80"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
```

### Sticky sessions com cookies (Traefik)

```yaml
labels:
  - "traefik.http.services.api.loadbalancer.sticky=true"
  - "traefik.http.services.api.loadbalancer.sticky.cookie.name=lb_sticky"
  - "traefik.http.services.api.loadbalancer.sticky.cookie.httponly=true"
  - "traefik.http.services.api.loadbalancer.sticky.cookie.secure=true"
```

### Endpoint de health check em Node.js

Todo backend deve expor um endpoint de health para o load balancer verificar:

```typescript
// src/routes/health.ts
import { FastifyPluginAsync } from 'fastify';
import { db } from '../lib/db.js';
import { redis } from '../lib/redis.js';

const healthRoute: FastifyPluginAsync = async (fastify) => {
  fastify.get('/health', { logLevel: 'silent' }, async (request, reply) => {
    const checks: Record<string, 'ok' | 'error'> = {};

    // Verifica conectividade com o banco de dados
    try {
      await db.$queryRaw`SELECT 1`;
      checks.database = 'ok';
    } catch {
      checks.database = 'error';
    }

    // Verifica conectividade com o Redis
    try {
      await redis.ping();
      checks.redis = 'ok';
    } catch {
      checks.redis = 'error';
    }

    const healthy = Object.values(checks).every((v) => v === 'ok');

    return reply.status(healthy ? 200 : 503).send({
      status: healthy ? 'healthy' : 'unhealthy',
      checks,
      version: process.env.APP_VERSION ?? 'unknown',
    });
  });

  // Liveness check — o processo está rodando? (sem verificações externas)
  fastify.get('/health/live', { logLevel: 'silent' }, async () => {
    return { status: 'alive' };
  });
};

export default healthRoute;
```

### Simulando load balancing em Node.js (entendendo o padrão)

```typescript
// Implementação conceitual — entendendo o round robin
class RoundRobinBalancer {
  private servers: string[];
  private index = 0;

  constructor(servers: string[]) {
    this.servers = servers;
  }

  next(): string {
    const server = this.servers[this.index % this.servers.length];
    this.index++;
    return server;
  }
}

class LeastConnectionsBalancer {
  private connections = new Map<string, number>();

  constructor(servers: string[]) {
    for (const server of servers) {
      this.connections.set(server, 0);
    }
  }

  acquire(): string {
    let min = Infinity;
    let chosen = '';
    for (const [server, count] of this.connections) {
      if (count < min) { min = count; chosen = server; }
    }
    this.connections.set(chosen, (this.connections.get(chosen) ?? 0) + 1);
    return chosen;
  }

  release(server: string): void {
    this.connections.set(server, Math.max(0, (this.connections.get(server) ?? 1) - 1));
  }
}
```

---

## Padrões e Boas Práticas

- **Backends sem estado** — não armazene dados de sessão na memória; armazene no Redis. Servidores sem estado são intercambiáveis.
- **Separe liveness e readiness probes** — liveness: "o processo está vivo?" Readiness: "ele consegue servir tráfego?" (verifica DB, Redis, etc.)
- **Remova servidores lentos rapidamente** — use `max_fails` e `fail_timeout` para que um servidor doente não arraste o pool inteiro
- **Use conexões keepalive para backends** — reduz o overhead de handshake TCP
- **Monitore a distribuição de latência dos backends** — p50, p90, p99 de cada backend separadamente para detectar nós degradados
- **Circuit breaker** — pare de enviar requisições para um backend que está retornando erros em alta taxa

---

## Anti-Padrões a Evitar

- Estado de sessão armazenado na memória da aplicação — impede escalonamento horizontal
- Não ter endpoints de health check — o load balancer não consegue detectar backends com falha
- Rotear todo o tráfego para um servidor "só para testar" e chamar isso de load balancing
- Ignorar a diferença entre o load balancer estar fora do ar e um backend estar fora do ar
- Sticky sessions de longa duração sem fallback — se o servidor fixado morrer, a sessão é perdida

---

## Debugging e Troubleshooting

**"Um servidor está recebendo todo o tráfego"**
Verifique se sticky sessions (ip_hash ou cookie) estão roteando tudo para um nó. Se sim, a distribuição está funcionando corretamente para o algoritmo configurado — mas talvez ip_hash não seja o mais adequado aqui.

**"O load balancer está marcando servidores saudáveis como falhos"**
Verifique o tempo de resposta do endpoint de health. Se ele expira por timeout (devido a uma verificação lenta do DB), o load balancer considera o servidor falho. Acelere o health check ou aumente o limite de timeout.

**"Picos de tráfego sobrecarregam os backends um de cada vez"**
Os algoritmos least connections ou menor tempo de resposta distribuem melhor sob duração variável de requisições. O round robin envia o mesmo número de requisições para cada servidor, independentemente de quanto tempo cada uma leva.

---

## Cenários do Mundo Real

**Cenário: Deployar 3 instâncias de API atrás do Nginx em um VPS**

```bash
# Subir instâncias
docker compose up --scale api=3 -d

# O Nginx faz round robin automaticamente entre as 3 instâncias
# Quando uma falha max_fails vezes, o Nginx a remove do pool
# Após fail_timeout, o Nginx tenta novamente
```

**Cenário: Deploy sem downtime com load balancing**

```bash
# Deploy rolling: sobe nova instância, verifica health, então remove a antiga
# 1. Inicia nova instância (api-v2)
docker run -d --name api-v2 myapp:v2

# 2. Adiciona ao upstream, verifica se começa a receber tráfego
# 3. Remove instância antiga do upstream
# 4. Para instância antiga
docker stop api-v1
```

---

## Leitura Complementar

- [Documentação do módulo upstream do Nginx](https://nginx.org/en/docs/http/ngx_http_upstream_module.html)
- [Documentação do load balancer do Traefik](https://doc.traefik.io/traefik/routing/services/)
- [Comparação AWS ALB vs NLB](https://aws.amazon.com/elasticloadbalancing/features/)
- Track 08: [Database Scaling](database-scaling.md)
- Track 08: [API Gateways](api-gateways.md)

---

## Resumo

Load balancing é a tecnologia habilitadora do escalonamento horizontal. A escolha do algoritmo importa: round robin funciona para serviços sem estado homogêneos; least connections é melhor para requisições de duração variável. Health checks são inegociáveis — sem eles, o balancer continua enviando requisições para backends com falha. Sticky sessions são uma camada de compatibilidade para aplicações com estado, mas a solução mais adequada é externalizar o estado para o Redis para que todo backend seja idêntico. Load balancing na camada 7 (Nginx, Traefik) é a escolha certa para APIs HTTP — ele habilita terminação TLS, roteamento baseado em conteúdo e stickiness por cookie.
