# API Gateways

## Visão Geral

Um API gateway é um servidor que atua como ponto de entrada único para todas as requisições de clientes em um sistema. Ele lida com responsabilidades transversais — autenticação, rate limiting, roteamento, transformação de requisições, logging e terminação SSL — para que os serviços de backend não precisem implementá-las individualmente. Em arquiteturas de microsserviços, o gateway é essencial. Em arquiteturas monolíticas, ele ainda agrega valor como reverse proxy e camada de segurança. Este capítulo cobre o que os gateways fazem, como construir lógica de gateway leve em Fastify e como configurar gateways de nível produção.

---

## Pré-requisitos

- Fundamentos HTTP (headers, métodos, proxying)
- Entendimento básico de conceitos de microsserviços
- Track 07 — Security: conceitos de autenticação e rate limiting

---

## Conceitos Fundamentais

### O que um API gateway faz

```
Cliente → [API Gateway] → Serviço A
                       → Serviço B
                       → Serviço C
```

**Responsabilidades do gateway:**

| Responsabilidade | Detalhes |
|-----------------|---------|
| Roteamento | Mapeia `/api/users` para `users-service`, `/api/orders` para `orders-service` |
| Autenticação | Verifica JWTs ou API keys antes de encaminhar requisições |
| Rate limiting | Limita requisições por cliente ou por IP |
| Load balancing | Distribui requisições entre instâncias de um serviço |
| Terminação SSL | Lida com HTTPS; encaminha HTTP para serviços internos |
| Transformação de requisições | Modifica headers, body ou URL antes de encaminhar |
| Transformação de respostas | Agrega, filtra ou reformata respostas |
| Caching | Faz cache de respostas em nível de gateway |
| Observabilidade | Logging centralizado, métricas, tracing |

### Gateway vs reverse proxy vs load balancer

- **Reverse proxy** (Nginx): roteia requisições, termina TLS, serve arquivos estáticos. Simples.
- **Load balancer** (Nginx upstream, AWS ALB): distribui tráfego. Focado em disponibilidade.
- **API gateway** (Kong, AWS API Gateway, Fastify customizado): tudo acima, mais auth, rate limiting e transformação. Ciente da aplicação.

---

## Exemplos Práticos

### Gateway simples em Fastify com @fastify/http-proxy

```bash
npm install @fastify/http-proxy
```

```typescript
// src/gateway.ts
import Fastify from 'fastify';
import proxy from '@fastify/http-proxy';
import helmet from '@fastify/helmet';
import rateLimit from '@fastify/rate-limit';
import jwt from '@fastify/jwt';

const gateway = Fastify({ logger: true });

await gateway.register(helmet);
await gateway.register(jwt, { secret: process.env.JWT_SECRET! });

// Rate limiting — aplicado globalmente antes do roteamento
await gateway.register(rateLimit, {
  max: 100,
  timeWindow: '1 minute',
  keyGenerator: (request) => request.headers.authorization ?? request.ip,
});

// Middleware de autenticação — aplicado em rotas protegidas
async function requireAuth(request: any, reply: any) {
  try {
    await request.jwtVerify();
  } catch {
    return reply.status(401).send({ error: 'Unauthorized' });
  }
}

// Rota: serviço de usuários (requer auth)
await gateway.register(proxy, {
  upstream: 'http://users-service:3001',
  prefix: '/api/users',
  preHandler: requireAuth,
  rewritePrefix: '/users',
  // Adiciona header X-User-Id para que o serviço upstream não precise re-verificar o JWT
  replyOptions: {
    rewriteRequestHeaders: (request, headers) => ({
      ...headers,
      'x-user-id': (request as any).user?.id,
      'x-user-role': (request as any).user?.role,
    }),
  },
});

// Rota: endpoint de busca público (sem auth)
await gateway.register(proxy, {
  upstream: 'http://search-service:3002',
  prefix: '/api/search',
  rewritePrefix: '/search',
});

// Rota: serviço de pedidos (requer auth + admin)
await gateway.register(proxy, {
  upstream: 'http://orders-service:3003',
  prefix: '/api/orders',
  preHandler: [requireAuth, async (request: any, reply: any) => {
    if (request.user.role !== 'admin') {
      return reply.status(403).send({ error: 'Forbidden' });
    }
  }],
  rewritePrefix: '/orders',
});

await gateway.listen({ port: 8080, host: '0.0.0.0' });
```

### Estratégias de rate limiting

```typescript
// Rate limit por usuário (rotas autenticadas)
await gateway.register(rateLimit, {
  max: 1000,
  timeWindow: '1 hour',
  keyGenerator: (request) => {
    // Autenticado: rate limit por ID de usuário
    const user = (request as any).user;
    if (user) return `user:${user.id}`;
    // Não autenticado: rate limit por IP
    return `ip:${request.ip}`;
  },
  errorResponseBuilder: (request, context) => ({
    code: 'RATE_LIMIT_EXCEEDED',
    error: 'Too Many Requests',
    message: `Rate limit exceeded. Try again in ${context.after}`,
    retryAfter: context.after,
  }),
});

// Rate limits por nível de assinatura
const RATE_LIMITS = {
  free: { max: 100, window: '1 hour' },
  pro: { max: 5000, window: '1 hour' },
  enterprise: { max: 100000, window: '1 hour' },
};

gateway.addHook('preHandler', async (request, reply) => {
  const user = (request as any).user;
  const tier = user?.tier ?? 'free';
  const limits = RATE_LIMITS[tier as keyof typeof RATE_LIMITS];

  // Verifica rate limit customizado no store (Redis)
  const key = `ratelimit:${user?.id ?? request.ip}`;
  const count = await redis.incr(key);
  if (count === 1) await redis.expire(key, 3600); // define TTL na primeira requisição

  if (count > limits.max) {
    return reply.status(429).send({
      error: 'Rate limit exceeded',
      limit: limits.max,
      reset: await redis.ttl(key),
    });
  }

  reply.header('X-RateLimit-Limit', limits.max);
  reply.header('X-RateLimit-Remaining', Math.max(0, limits.max - count));
});
```

### Transformação de requisição/resposta

```typescript
// Adiciona correlation ID a toda requisição para tracing
gateway.addHook('onRequest', async (request) => {
  const correlationId = request.headers['x-correlation-id'] ?? crypto.randomUUID();
  request.headers['x-correlation-id'] = correlationId;
});

// Remove headers internos das respostas upstream antes de enviar ao cliente
gateway.addHook('onSend', async (request, reply, payload) => {
  reply.removeHeader('x-internal-service-name');
  reply.removeHeader('x-db-query-count');
  return payload;
});
```

### Configuração do Kong API Gateway

O Kong é um gateway de nível produção com ecossistema de plugins. Configure via sua Admin API:

```bash
# Cria um serviço (backend)
curl -i -X POST http://kong:8001/services \
  --data name=users-service \
  --data url=http://users-service:3001

# Cria uma rota (mapeia URL para o serviço)
curl -i -X POST http://kong:8001/services/users-service/routes \
  --data paths[]=/api/users

# Adiciona plugin de autenticação JWT
curl -i -X POST http://kong:8001/services/users-service/plugins \
  --data name=jwt

# Adiciona plugin de rate limiting
curl -i -X POST http://kong:8001/services/users-service/plugins \
  --data name=rate-limiting \
  --data config.minute=60 \
  --data config.hour=1000 \
  --data config.policy=redis \
  --data config.redis_host=redis
```

### Nginx como gateway leve

```nginx
http {
  # Serviços backend
  upstream users_service { server users-service:3001; }
  upstream orders_service { server orders-service:3002; }
  upstream search_service { server search-service:3003; }

  # Zona de rate limiting
  limit_req_zone $binary_remote_addr zone=api_limit:10m rate=100r/m;

  server {
    listen 80;

    # Aplica rate limit
    limit_req zone=api_limit burst=20 nodelay;

    # Roteia por prefixo de URL
    location /api/users {
      auth_request /auth/verify;   # subrequisição para verificar JWT
      proxy_pass http://users_service;
      proxy_set_header X-Real-IP $remote_addr;
    }

    location /api/orders {
      auth_request /auth/verify;
      proxy_pass http://orders_service;
    }

    location /api/search {
      # Público — sem auth
      proxy_pass http://search_service;
    }

    # Endpoint interno de verificação de auth
    location = /auth/verify {
      internal;
      proxy_pass http://auth-service:3000/verify;
      proxy_pass_request_body off;
      proxy_set_header Content-Length "";
      proxy_set_header X-Original-URI $request_uri;
    }
  }
}
```

---

## Padrões e Boas Práticas

- **Centralize auth no gateway** — serviços downstream confiam nos headers `X-User-Id` e `X-User-Role` definidos pelo gateway; eles não verificam JWTs independentemente
- **Passe correlation IDs** — adicione `X-Correlation-ID` a toda requisição para tracing de ponta a ponta entre serviços
- **Logue em nível de gateway** — um stream de log centralizado vs. N streams de log de serviços
- **Use circuit breakers** — se um backend está com problemas, pare de encaminhar requisições para ele (fail fast)
- **Retorne respostas de erro consistentes** — traduza erros upstream para um formato de envelope consistente

---

## Anti-Padrões a Evitar

- Lógica de negócio no gateway — gateways lidam com responsabilidades transversais, não com lógica de domínio
- Gateway único para todo o tráfego sem fallback — o próprio gateway se torna um ponto único de falha (rode múltiplas instâncias atrás de um load balancer)
- Bloquear o gateway com chamadas upstream lentas — defina timeouts agressivos para upstreams
- Encaminhar headers internos ao cliente — remova headers `X-Internal-*` das respostas de saída

---

## Debugging e Troubleshooting

**"Clientes estão recebendo 502 Bad Gateway"**
O serviço upstream não está respondendo. Verifique se o serviço está rodando e se o gateway tem a URL upstream correta. Verifique os logs do upstream para erros.

**"O rate limiting não está funcionando"**
Se rodar múltiplas instâncias de gateway sem estado Redis compartilhado, cada instância tem seu próprio contador. Use rate limiting com backend Redis para que todas as instâncias compartilhem um único contador.

**"O JWT está inválido segundo o serviço downstream"**
O serviço downstream não deveria estar verificando JWTs — apenas o gateway deveria. Serviços downstream confiam no header `X-User-Id` definido pelo gateway. Remova a verificação de JWT dos serviços downstream.

---

## Leitura Complementar

- [Documentação do Kong API Gateway](https://docs.konghq.com/)
- [Documentação do AWS API Gateway](https://docs.aws.amazon.com/apigateway/)
- [@fastify/http-proxy](https://github.com/fastify/fastify-http-proxy)
- Track 08: [Load Balancing](load-balancing.md)
- Track 07: [Security Headers](../07-security/security-headers.md)

---

## Resumo

Um API gateway é o local natural para responsabilidades transversais em uma arquitetura multi-serviço: autenticação, rate limiting, roteamento, terminação SSL e observabilidade. Para times rodando microsserviços, um gateway é essencial — ele evita que todo serviço reimplemente auth e rate limiting independentemente. Para times com monolito, Nginx ou um gateway Fastify lida com roteamento e segurança com menor complexidade. O princípio-chave é que o gateway lida com preocupações de infraestrutura e passa a identidade para os serviços; os serviços focam em lógica de negócio.
