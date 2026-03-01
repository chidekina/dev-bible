# Cache e Redis

## 1. O Que é e Por Que Usar

Cache é a prática de armazenar uma cópia de dados em uma camada de armazenamento mais rápida para atender requisições futuras com mais rapidez. Uma consulta ao banco que demora 50ms pode ser servida do cache em 0,1ms — uma aceleração de 500x. Cache é uma das otimizações de performance de maior alavancagem disponíveis, e também uma das mais perigosas se feita incorretamente.

Redis (Remote Dictionary Server) é a camada de cache padrão da indústria: single-threaded, in-memory, latência submilissegundo, com estruturas de dados ricas que habilitam muito mais do que simples cache key-value. Redis é usado para sessões, rate limiting, locks distribuídos, leaderboards, pub/sub e filas de background jobs.

A citação mais famosa da ciência da computação: "Existem apenas duas coisas difíceis em Ciência da Computação: invalidação de cache e nomenclatura de coisas." — Phil Karlton. Este arquivo aborda ambas.

---

## 2. Conceitos Fundamentais

### Estratégias de Cache

**Cache-Aside (Lazy Loading) — mais comum:**

O código da aplicação gerencia o cache. Em caso de cache miss, a aplicação lê do banco e popula o cache.

```
Fluxo de leitura:
  1. Verifica o cache pela chave
  2. Se HIT: retorna o valor em cache
  3. Se MISS: lê do banco, escreve no cache com TTL, retorna o valor

Fluxo de escrita:
  1. Escreve no banco
  2. Invalida ou atualiza a chave do cache
```

Prós: apenas dados requisitados são cacheados; falhas do cache não são catastróficas (app lê do banco). Contras: primeiras requisições sempre resultam em miss (cold start); janela de inconsistência entre escrita no banco e invalidação do cache.

**Write-Through:**

Toda escrita vai tanto para o cache quanto para o banco simultaneamente.

```
Fluxo de escrita:
  1. Escreve no cache
  2. Escreve no banco (sincronamente)
```

Prós: cache sempre consistente com o banco; sem janela de inconsistência. Contras: toda escrita tem latência dupla; cache se enche com dados que podem nunca ser lidos (desperdício).

**Write-Behind (Write-Back):**

Escritas vão primeiro para o cache; escritas no banco são assíncronas (em buffer).

```
Fluxo de escrita:
  1. Escreve no cache imediatamente (retorna para o cliente)
  2. Async: escreve mudanças do cache no banco em lote
```

Prós: escritas muito rápidas; batch de escritas no banco reduz carga. Contras: risco de perda de dados se o cache travar antes da escrita no banco; complexo de implementar corretamente.

**Read-Through:**

Cache fica na frente do banco; em caso de miss, o próprio cache busca no banco.

```
Fluxo de leitura:
  1. App lê do cache
  2. Cache miss: cache busca do banco, armazena, retorna
  3. App sempre fala apenas com o cache
```

Prós: transparente para o código da aplicação. Contras: requer que o cache tenha credenciais de acesso ao banco; menos flexível.

> Cache-aside é o padrão certo para a maioria das aplicações. Use write-through quando consistência é crítica. Write-behind apenas para throughput de escrita muito alto onde alguma perda de dados é aceitável (analytics, métricas).

---

## 3. Como Funciona — Tipos de Dados do Redis

### Strings

```typescript
import Redis from 'ioredis';
const redis = new Redis(process.env.REDIS_URL!);

// Cache básico com TTL
await redis.setex(`user:${id}`, 300, JSON.stringify(userData)); // TTL 5 min
const cached = await redis.get(`user:${id}`);
const user = cached ? JSON.parse(cached) : null;

// Contador atômico (INCR é atômico — sem race conditions)
const views = await redis.incr(`post:${postId}:views`);
await redis.incrby(`user:${userId}:credits`, 50);
await redis.decrby(`user:${userId}:credits`, 10);

// GET e SET atomicamente (troca atômica de valor, retorna o antigo)
const prev = await redis.getset(`config:maintenance`, 'true');

// SET com opções
await redis.set('lock:resource', 'owner-id', 'EX', 30, 'NX'); // EX=TTL, NX=apenas se não existir
```

### Hashes

```typescript
// Armazenar objetos sem overhead de serialização JSON
await redis.hset(`session:${sessionId}`, {
  userId: user.id,
  role: user.role,
  ip: req.ip,
  createdAt: Date.now().toString(),
});
await redis.expire(`session:${sessionId}`, 3600);

const session = await redis.hgetall(`session:${sessionId}`);
// { userId: '...', role: 'admin', ip: '...', createdAt: '...' }

// Incrementa um campo do hash atomicamente
await redis.hincrby(`stats:${date}`, 'pageViews', 1);
await redis.hincrby(`stats:${date}`, 'uniqueVisitors', 1);
```

### Lists

```typescript
// Fila (FIFO): RPUSH para enfileirar, LPOP para desenfileirar
await redis.rpush('job:queue', JSON.stringify(job));
const item = await redis.lpop('job:queue');

// Pilha (LIFO): RPUSH para empurrar, RPOP para retirar
await redis.rpush('undo:stack', JSON.stringify(action));
const last = await redis.rpop('undo:stack');

// Feed de atividades: mantém os últimos N itens
await redis.lpush(`feed:${userId}`, JSON.stringify(activity));
await redis.ltrim(`feed:${userId}`, 0, 99); // mantém apenas os 100 mais recentes
const feed = await redis.lrange(`feed:${userId}`, 0, 19); // primeiros 20

// Blocking pop (para workers de jobs — bloqueia até ter um item disponível)
const [queue, value] = await redis.blpop('job:queue', 0); // 0 = bloqueia indefinidamente
```

### Sets

```typescript
// Coleções únicas — O(1) para adicionar, remover, verificar pertencimento
await redis.sadd(`post:${postId}:likers`, userId);
await redis.srem(`post:${postId}:likers`, userId);
const liked = await redis.sismember(`post:${postId}:likers`, userId); // 0 ou 1
const likeCount = await redis.scard(`post:${postId}:likers`);

// Operações de conjunto: união, interseção, diferença
const commonFollowers = await redis.sinter(`user:1:followers`, `user:2:followers`);
const allFollowers = await redis.sunion(`user:1:followers`, `user:2:followers`);
```

### Sorted Sets

```typescript
// Leaderboard: O(log n) para adicionar e consultar rank
await redis.zadd('leaderboard:season5', score, userId);

// Top 10 jogadores (score mais alto primeiro)
const top10 = await redis.zrevrange('leaderboard:season5', 0, 9, 'WITHSCORES');
// ['userId1', '9850', 'userId2', '9720', ...]

// Rank de um jogador específico (0-indexado, maior = melhor)
const rank = await redis.zrevrank('leaderboard:season5', userId);
// null se não estiver no set; 0 = primeiro lugar

// Jogadores ao redor de um usuário (para leaderboard "perto de você")
const userRank = await redis.zrevrank('leaderboard:season5', userId);
const nearby = await redis.zrevrange('leaderboard:season5',
  Math.max(0, userRank - 2), userRank + 2, 'WITHSCORES');

// Rate limiting com sorted set (janela deslizante)
const now = Date.now();
const windowStart = now - 60_000; // janela de 1 minuto
await redis.zremrangebyscore(`rate:${userId}`, 0, windowStart);
const count = await redis.zcard(`rate:${userId}`);
if (count >= 100) throw new RateLimitError();
await redis.zadd(`rate:${userId}`, now, `${now}-${Math.random()}`);
await redis.expire(`rate:${userId}`, 60);
```

---

## 4. Exemplos de Código (TypeScript / ioredis)

### Padrão Cache-Aside

```typescript
interface CacheOptions {
  ttl?: number; // segundos
  staleWhileRevalidate?: number; // segundos — serve dado stale enquanto atualiza
}

class CacheService {
  constructor(private redis: Redis) {}

  async getOrSet<T>(
    key: string,
    fetchFn: () => Promise<T>,
    options: CacheOptions = {}
  ): Promise<T> {
    const { ttl = 300 } = options;

    const cached = await this.redis.get(key);
    if (cached !== null) {
      return JSON.parse(cached) as T;
    }

    const value = await fetchFn();
    await this.redis.setex(key, ttl, JSON.stringify(value));
    return value;
  }

  async invalidate(key: string): Promise<void> {
    await this.redis.del(key);
  }

  async invalidatePattern(pattern: string): Promise<void> {
    // SCAN ao invés de KEYS — KEYS bloqueia o Redis
    let cursor = '0';
    do {
      const [nextCursor, keys] = await this.redis.scan(cursor, 'MATCH', pattern, 'COUNT', 100);
      cursor = nextCursor;
      if (keys.length > 0) {
        await this.redis.del(...keys);
      }
    } while (cursor !== '0');
  }
}

// Uso em um service
class UserService {
  constructor(private db: Database, private cache: CacheService) {}

  async getUser(id: string): Promise<User> {
    return this.cache.getOrSet(
      `user:${id}`,
      () => this.db.users.findUniqueOrThrow({ where: { id } }),
      { ttl: 300 }
    );
  }

  async updateUser(id: string, data: Partial<User>): Promise<User> {
    const updated = await this.db.users.update({ where: { id }, data });
    await this.cache.invalidate(`user:${id}`); // invalida após a escrita
    return updated;
  }
}
```

### Rate Limiting com Janela Deslizante

```typescript
// Rate limiter atômico usando script Lua para prevenir race conditions
const RATE_LIMIT_SCRIPT = `
  local key = KEYS[1]
  local now = tonumber(ARGV[1])
  local window = tonumber(ARGV[2])
  local limit = tonumber(ARGV[3])
  local unique = ARGV[4]

  -- Remove entradas expiradas
  redis.call('ZREMRANGEBYSCORE', key, 0, now - window * 1000)

  -- Conta entradas atuais
  local count = redis.call('ZCARD', key)

  if count >= limit then
    -- Obtém a entrada mais antiga para calcular o tempo de reset
    local oldest = redis.call('ZRANGE', key, 0, 0, 'WITHSCORES')
    local resetAt = tonumber(oldest[2]) + window * 1000
    return {0, count, resetAt}
  end

  -- Adiciona a requisição atual
  redis.call('ZADD', key, now, unique)
  redis.call('EXPIRE', key, window)

  return {1, count + 1, 0}
`;

async function rateLimit(
  redis: Redis,
  key: string,
  limit: number,
  windowSecs: number
): Promise<{ allowed: boolean; remaining: number; resetAt: number }> {
  const now = Date.now();
  const unique = `${now}-${Math.random()}`;

  const result = await redis.eval(
    RATE_LIMIT_SCRIPT, 1, key, now, windowSecs, limit, unique
  ) as [number, number, number];

  const [allowed, count, resetAt] = result;
  return {
    allowed: allowed === 1,
    remaining: Math.max(0, limit - count),
    resetAt: resetAt || now + windowSecs * 1000,
  };
}

// Middleware Fastify
app.addHook('preHandler', async (req, reply) => {
  const userId = req.user?.id ?? req.ip;
  const { allowed, remaining, resetAt } = await rateLimit(
    redis, `rate:api:${userId}`, 100, 60
  );

  reply.header('X-RateLimit-Limit', '100');
  reply.header('X-RateLimit-Remaining', remaining.toString());
  reply.header('X-RateLimit-Reset', Math.ceil(resetAt / 1000).toString());

  if (!allowed) {
    return reply.status(429).send({
      type: 'https://api.example.com/errors/rate-limited',
      title: 'Too Many Requests',
      status: 429,
      detail: 'Rate limit exceeded. Please retry after the reset time.',
    });
  }
});
```

### Lock Distribuído (Redlock)

```typescript
// Implementação simplificada de Redlock para Redis single-node
// Para deploy multi-node em produção, use o pacote npm 'redlock'

async function acquireLock(
  redis: Redis,
  resource: string,
  ttlMs: number
): Promise<string | null> {
  const lockId = crypto.randomUUID();
  const key = `lock:${resource}`;

  // SET NX EX: define apenas se não existir, com validade
  const result = await redis.set(key, lockId, 'PX', ttlMs, 'NX');
  return result === 'OK' ? lockId : null;
}

async function releaseLock(
  redis: Redis,
  resource: string,
  lockId: string
): Promise<boolean> {
  // Script Lua: verifica propriedade + deleta atomicamente
  // Sem isso, você pode deletar um lock que não é seu (race condition)
  const RELEASE_SCRIPT = `
    if redis.call('GET', KEYS[1]) == ARGV[1] then
      return redis.call('DEL', KEYS[1])
    else
      return 0
    end
  `;
  const result = await redis.eval(RELEASE_SCRIPT, 1, `lock:${resource}`, lockId);
  return result === 1;
}

async function withLock<T>(
  redis: Redis,
  resource: string,
  ttlMs: number,
  fn: () => Promise<T>
): Promise<T> {
  const lockId = await acquireLock(redis, resource, ttlMs);
  if (!lockId) {
    throw new Error(`Could not acquire lock for resource: ${resource}`);
  }
  try {
    return await fn();
  } finally {
    await releaseLock(redis, resource, lockId);
  }
}

// Uso: prevenir cobrança dupla em um pagamento
await withLock(redis, `payment:${orderId}`, 30_000, async () => {
  const order = await db.orders.findUnique({ where: { id: orderId } });
  if (order?.status === 'PAID') return; // idempotente
  await paymentProvider.charge(order.amount);
  await db.orders.update({ where: { id: orderId }, data: { status: 'PAID' } });
});
```

### Armazenamento de Sessão

```typescript
// Armazenar sessões de usuário no Redis
interface Session {
  userId: string;
  role: string;
  ip: string;
  createdAt: number;
  lastAccessedAt: number;
}

class SessionStore {
  private SESSION_TTL = 7 * 24 * 3600; // 7 dias

  constructor(private redis: Redis) {}

  async create(userId: string, role: string, ip: string): Promise<string> {
    const sessionId = crypto.randomBytes(32).toString('base64url');
    const session: Session = {
      userId, role, ip,
      createdAt: Date.now(),
      lastAccessedAt: Date.now(),
    };

    await this.redis.hset(`session:${sessionId}`, session as any);
    await this.redis.expire(`session:${sessionId}`, this.SESSION_TTL);

    return sessionId;
  }

  async get(sessionId: string): Promise<Session | null> {
    const data = await this.redis.hgetall(`session:${sessionId}`);
    if (!data || Object.keys(data).length === 0) return null;

    // Expiração deslizante: estende o TTL a cada acesso
    await this.redis.expire(`session:${sessionId}`, this.SESSION_TTL);
    await this.redis.hset(`session:${sessionId}`, 'lastAccessedAt', Date.now());

    return data as unknown as Session;
  }

  async destroy(sessionId: string): Promise<void> {
    await this.redis.del(`session:${sessionId}`);
  }

  // Revoga todas as sessões de um usuário (ex.: após mudança de senha)
  async destroyAllForUser(userId: string): Promise<void> {
    let cursor = '0';
    do {
      const [next, keys] = await this.redis.scan(cursor, 'MATCH', 'session:*', 'COUNT', 100);
      cursor = next;
      for (const key of keys) {
        const uid = await this.redis.hget(key, 'userId');
        if (uid === userId) await this.redis.del(key);
      }
    } while (cursor !== '0');
  }
}
```

### Pipeline e Transaction

```typescript
// Pipeline: agrupa múltiplos comandos, reduz round trips (não atômico)
const pipeline = redis.pipeline();
pipeline.get('key1');
pipeline.get('key2');
pipeline.set('key3', 'value3', 'EX', 300);
pipeline.incr('counter');
const results = await pipeline.exec();
// results: [[null, 'value1'], [null, 'value2'], [null, 'OK'], [null, 1]]

// Transaction (MULTI/EXEC): lote atômico — tudo tem sucesso ou nada se aplica
const multi = redis.multi();
multi.set('account:1:balance', '900');
multi.set('account:2:balance', '1100');
multi.incrby('audit:transfers', 1);
const txResults = await multi.exec(); // retorna null se WATCH detectou uma mudança
```

---

## 5. Erros Comuns e Armadilhas

**Cache stampede (thundering herd):**

```typescript
// Problema: todas as requisições batem no banco simultaneamente quando uma chave expira
// Imagine 1000 req/s e uma chave popular expirar às 14:00:00.000
// Todas as 1000 requisições recebem cache miss simultaneamente → 1000 queries no banco

// Fix 1: Baseado em lock (apenas uma requisição busca, outras aguardam)
async function getWithStampedeProtection<T>(
  redis: Redis,
  key: string,
  fetchFn: () => Promise<T>,
  ttl: number
): Promise<T> {
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  // Tenta adquirir o lock "estou buscando"
  const lockKey = `${key}:fetching`;
  const acquired = await redis.set(lockKey, '1', 'EX', 10, 'NX');

  if (acquired) {
    // Conseguimos o lock — busca e popula
    const value = await fetchFn();
    await redis.setex(key, ttl, JSON.stringify(value));
    await redis.del(lockKey);
    return value;
  }

  // Outra requisição está buscando — aguarda e tenta novamente
  await new Promise((r) => setTimeout(r, 50));
  return getWithStampedeProtection(redis, key, fetchFn, ttl);
}
```

**Usar o comando KEYS em produção:**

```typescript
// ERRADO: KEYS bloqueia o Redis pelo tempo do scan — O(N) onde N = total de chaves
const keys = await redis.keys('user:*'); // congela o Redis por segundos em grandes datasets

// CORRETO: use SCAN — iteração não-bloqueante baseada em cursor
async function scanKeys(redis: Redis, pattern: string): Promise<string[]> {
  const results: string[] = [];
  let cursor = '0';
  do {
    const [nextCursor, keys] = await redis.scan(cursor, 'MATCH', pattern, 'COUNT', 100);
    cursor = nextCursor;
    results.push(...keys);
  } while (cursor !== '0');
  return results;
}
```

**Sem TTL em valores cacheados:**

```typescript
// ERRADO: cache cresce indefinidamente, dados stale nunca expiram
await redis.set(`user:${id}`, JSON.stringify(user));

// CORRETO: sempre defina TTL
await redis.setex(`user:${id}`, 300, JSON.stringify(user)); // 5 minutos
// OU
await redis.set(`user:${id}`, JSON.stringify(user), 'EX', 300);
```

**Cachear dados mutáveis por muito tempo:**

```typescript
// Perigo: cachear perfil de usuário por 1 hora
// Usuário atualiza o nome → nome antigo servido por até 1 hora
// Fix: invalide na escrita E use um TTL menor como fallback
await redis.setex(`user:${id}`, 60, JSON.stringify(user)); // máximo 60s de staleness

// Na atualização: invalide imediatamente
await db.users.update({ where: { id }, data: { name: newName } });
await redis.del(`user:${id}`); // invalida o cache imediatamente
```

---

## 6. Quando Usar / Não Usar

**Use cache Redis para:**
- Queries caras ao banco que são lidas frequentemente (perfis de usuário, detalhes de produto, configurações)
- Respostas de API para dados públicos (feed de notícias, listagens de produto)
- Dados computados que são caros de recalcular (agregações, scores de recomendação)
- Rate limiting, armazenamento de sessão, locks distribuídos

**Evite cachear:**
- Dados altamente voláteis onde leituras stale são inaceitáveis (saldos bancários, contagens de estoque)
- Dados raramente requisitados (baixa taxa de hit = poluição do cache)
- Queries muito pequenas e rápidas (overhead do cache pode superar o tempo da query)
- Dados personalizados com chaves de alta cardinalidade (uma chave por usuário por recurso = muitas chaves)

**Redis vs Memcached:**

| Fator | Redis | Memcached |
|-------|-------|-----------|
| Estruturas de dados | Ricas (strings, hashes, lists, sets, sorted sets, streams) | Apenas strings |
| Persistência | RDB + AOF | Nenhuma |
| Clustering | Redis Cluster (nativo) | Consistent hashing (client-side) |
| Pub/sub | Built-in | Não suportado |
| Scripting Lua | Suportado | Não suportado |
| Eficiência de memória | Ligeiramente menor | Ligeiramente maior |
| Escolha quando | Quase sempre | Cache mais simples possível, eficiência de memória crítica |

> Redis efetivamente substituiu o Memcached para novos projetos. A única razão para escolher Memcached hoje é se você tem infraestrutura existente ou precisa de eficiência de memória ligeiramente melhor para cache puro de strings em escala extrema.

---

## 7. Estratégia de TTL

```typescript
// Diretrizes de TTL por tipo de dado
const TTL = {
  // Quase em tempo real (muda frequentemente)
  stockPrice: 5,           // 5 segundos
  liveScore: 10,           // 10 segundos

  // Curta duração (minutos)
  searchResults: 60,       // 1 minuto
  userActivity: 120,       // 2 minutos

  // Média duração (horas)
  userProfile: 3600,       // 1 hora
  productDetails: 1800,    // 30 minutos

  // Longa duração (dia+)
  staticConfig: 86400,     // 24 horas
  countryList: 86400 * 7,  // 1 semana

  // Baseado em sessão
  authSession: 1800,       // 30 minutos de idle timeout
  refreshToken: 86400 * 30, // 30 dias
};

// Jitter: adiciona aleatoriedade para prevenir expiração sincronizada
function ttlWithJitter(baseTtl: number, jitterPercent = 10): number {
  const jitter = baseTtl * (jitterPercent / 100);
  return Math.floor(baseTtl + (Math.random() * 2 - 1) * jitter);
}

// Ao invés de todos os perfis de usuário expirarem ao mesmo tempo
await redis.setex(`user:${id}`, ttlWithJitter(3600), JSON.stringify(user));
// Alguns expiram em 3240s, outros em 3760s — sem stampede sincronizado
```

---

## 8. Políticas de Eviction

Quando o Redis atinge seu limite de memória (`maxmemory`), ele evita chaves baseado na política configurada:

```
# redis.conf
maxmemory 512mb
maxmemory-policy allkeys-lru

Políticas:
  noeviction:      Erro de escrita quando memória cheia. Use para: data store primário (sem eviction aceitável)
  allkeys-lru:     Evita o menos recentemente usado entre TODAS as chaves. Use para: cache puro (toda chave é descartável)
  volatile-lru:    LRU apenas nas chaves COM TTL. Use para: mix de cache + dados persistentes
  allkeys-lfu:     Evita o menos frequentemente usado. Use para: cache com acesso power-law (algumas chaves 100x mais quentes)
  volatile-lfu:    LFU apenas nas chaves com TTL.
  allkeys-random:  Eviction aleatório. Raramente útil.
  volatile-ttl:    Evita chave com menor TTL restante. Use para: priorizar chaves de vida longa
  volatile-random: Eviction aleatório apenas nas chaves com TTL.
```

---

## 9. Cenário Real

**Problema:** Um site de notícias serve 50.000 requisições/minuto em uma matéria viral. A query "busca artigo com contagem de comentários + curtidas + detalhes do autor" demora 80ms no PostgreSQL. Sob essa carga, o banco está com 100% de CPU.

**Solução: Cache-aside com invalidação**

```typescript
class ArticleService {
  constructor(private db: PrismaClient, private redis: Redis) {}

  async getArticle(slug: string): Promise<ArticleDTO | null> {
    const cacheKey = `article:${slug}`;
    const cached = await this.redis.get(cacheKey);

    if (cached) {
      // Cache hit: 0,3ms
      return JSON.parse(cached);
    }

    // Cache miss: 80ms de query no banco
    const article = await this.db.article.findUnique({
      where: { slug },
      include: {
        author: { select: { name: true, avatarUrl: true } },
        _count: { select: { comments: true, likes: true } },
      },
    });

    if (!article) return null;

    const dto: ArticleDTO = {
      ...article,
      commentCount: article._count.comments,
      likeCount: article._count.likes,
    };

    // Cache por 5 minutos + jitter
    const ttl = ttlWithJitter(300);
    await this.redis.setex(cacheKey, ttl, JSON.stringify(dto));

    return dto;
  }

  // Invalida quando o artigo é atualizado
  async updateArticle(id: string, data: Partial<Article>): Promise<Article> {
    const updated = await this.db.article.update({ where: { id }, data });
    await this.redis.del(`article:${updated.slug}`);
    return updated;
  }

  // Invalida quando um comentário é adicionado (contagem de comentários mudou)
  async addComment(articleId: string, comment: CreateCommentInput): Promise<Comment> {
    const article = await this.db.article.findUniqueOrThrow({ where: { id: articleId } });
    const newComment = await this.db.comment.create({
      data: { ...comment, articleId },
    });
    await this.redis.del(`article:${article.slug}`); // invalida contagem stale
    return newComment;
  }
}
```

**Resultado:** Taxa de cache hit atinge 98% em 30 segundos após a matéria viralizar. CPU do banco cai de 100% para 4%. Tempo de resposta para requisições em cache: 0,5ms. O banco trata apenas 1.000 requisições/minuto (2% de misses) ao invés de 50.000.

---

## 10. Perguntas de Entrevista

**Q1: Qual é a diferença entre as estratégias cache-aside e write-through?**

Cache-aside (lazy loading): a aplicação verifica o cache primeiro; em caso de miss, lê do banco e popula o cache. O cache só contém dados que foram requisitados pelo menos uma vez. Write-through: toda escrita vai tanto para o cache quanto para o banco sincronamente. O cache está sempre sincronizado mas pode conter dados que nunca são lidos novamente, e cada escrita tem latência maior. Cache-aside é mais comum porque evita poluir o cache com dados não lidos; write-through é usado quando consistência de leitura após escrita é crítica.

**Q2: Qual política de eviction você usaria para um session store e por quê?**

`volatile-lru` ou `noeviction`. Para um session store, as sessões são o dado primário — você não deve descartá-las para liberar espaço para outras chaves. Use `volatile-lru` se o Redis também serve como cache (sessões têm TTL definido; itens em cache também têm TTL; LRU garante que itens de cache menos usados recentemente sejam descartados primeiro). Use `noeviction` se o Redis é dedicado apenas para sessões (Redis retorna erros quando a memória está cheia ao invés de silenciosamente perder sessões). Nunca use `allkeys-lru` para um session store — descartaria sessões mesmo quando existem chaves que não são sessões.

**Q3: Quais são as estratégias para invalidação de cache?**

(1) **Baseada em TTL** — mais simples; define uma validade e aceita que dados podem ficar stale por até TTL segundos. (2) **Invalidação baseada em eventos** — quando dados mudam no banco, delete explicitamente a chave do cache (`redis.del(key)`). (3) **Write-through** — cache é sempre atualizado na escrita, nunca stale. (4) **Versionamento de cache** — inclua um token de versão na chave do cache; mudar a versão efetivamente invalida todas as chaves antigas sem deletá-las (útil para invalidação em massa). (5) **Invalidação baseada em tags** — marque chaves do cache com IDs de entidade; na atualização da entidade, delete todas as chaves marcadas.

**Q4: O que é cache stampede e como você o previne?**

Cache stampede (thundering herd) ocorre quando uma chave popular expira e muitas requisições concorrentes recebem cache miss simultaneamente, fazendo todas baterem no banco ao mesmo tempo. Estratégias de prevenção: (1) **Mutex/lock** — a primeira thread a receber um miss adquire um lock no Redis e busca; as outras aguardam e tentam novamente. (2) **Expiração probabilística antecipada** — algumas requisições começam a atualizar o cache antes de ele expirar, escalonadas aleatoriamente. (3) **Stale-while-revalidate** — serve o valor em cache stale imediatamente; atualiza assincronamente em background. (4) **TTL jitter** — adiciona aleatoriedade aos TTLs para que chaves de dados similares não expirem simultaneamente.

**Q5: O que é Redlock e quando você precisa dele?**

Redlock é um algoritmo de lock distribuído para Redis. O padrão básico: `SET key value NX EX ttl` (define apenas se não existir, com validade). Para um único nó Redis, isso é suficiente para a maioria dos casos. Redlock estende isso para múltiplos nós Redis: um lock é considerado adquirido se a maioria de N nós (N/2+1) o concede, prevenindo que a falha de um único nó torne o lock indisponível ou crie split-brain. Use locks distribuídos quando você precisa garantir que apenas um processo execute uma seção crítica por vez (ex.: prevenir pagamentos duplos, agendar jobs exclusivos).

**Q6: Como o Redis se compara ao Memcached para cache?**

Redis é a escolha certa para quase todos os novos projetos. Razões: Redis suporta estruturas de dados ricas (hashes, sorted sets, lists) habilitando casos de uso além de simples cache (leaderboards, rate limiting, sessões, pub/sub); Redis tem persistência (RDB, AOF) para que dados sobrevivam a reinicializações; Redis suporta scripting Lua para operações atômicas de múltiplos passos; Redis Cluster fornece sharding nativo. Memcached é mais simples, ligeiramente mais eficiente em memória para cache puro de strings e pode ser melhor para cargas de escrita multi-threaded. Na prática, a versatilidade do Redis vence em quase todos os cenários.

**Q7: O que são sorted sets no Redis e quais são os casos de uso típicos?**

Sorted sets armazenam membros de string únicos cada um associado a um score float. Membros são ordenados por score; empates são resolvidos lexicograficamente. Operações: `ZADD` (adicionar/atualizar), `ZRANK`/`ZREVRANK` (lookup de rank O(log n)), `ZRANGE`/`ZREVRANGE` (obter membros por intervalo de rank), `ZRANGEBYSCORE` (obter por intervalo de score). Casos de uso: (1) Leaderboards — `ZADD leaderboard score userId`, `ZREVRANGE leaderboard 0 9` para top 10. (2) Rate limiting — score = timestamp, queries de intervalo removem entradas expiradas. (3) Jobs agendados — score = timestamp de execução, workers processam jobs com score <= now(). (4) Autocomplete — score 0 para todas as entradas, query de intervalo por prefixo lexicográfico.

---

## 11. Exercícios

**Exercício 1: Cache warming**

Uma página de listagem de produtos demora 800ms para carregar do banco em cache frio. Construa uma estratégia de cache warming que:
- Pré-popula os 100 produtos mais vistos na inicialização da aplicação
- Os atualiza a cada 10 minutos sem causar stampede
- Usa script Lua para verificar o TTL atomicamente e atualizar condicionalmente

*Dica:* Use o comando `TTL` para verificar o TTL restante. Se < 60 segundos, atualize. Envolva em Lua para evitar race conditions.

**Exercício 2: Cache multi-tier**

Implemente um cache de três camadas para perfis de usuário:
- L1: Map in-process (sem rede, microssegundos)
- L2: Redis (submilissegundo)
- L3: PostgreSQL (fonte da verdade)
- Cache se popula de L3 → L2 → L1 em caso de miss
- Invalidação se propaga de L3 → L2 → L1

*Dica:* L1 precisa de limite de tamanho (eviction LRU) e TTL. Considere usar o pacote npm `lru-cache`.

**Exercício 3: Leaderboard em tempo real**

Construa uma API de leaderboard usando sorted sets do Redis:
- `POST /scores` — envia um score para um usuário
- `GET /leaderboard?limit=10` — obtém os top N usuários com scores
- `GET /leaderboard/me/:userId` — obtém o rank do usuário + 2 usuários acima e abaixo ("perto de você")
- `GET /leaderboard/percentile/:userId` — retorna o percentil do usuário
- Trate atualizações concorrentes de score atomicamente

*Dica:* Use `ZREVRANK` para rank, `ZCARD` para contagem total, `ZREVRANGE` com offset para usuários "próximos".

---

## 12. Leitura Complementar

- [Documentação do Redis](https://redis.io/docs/)
- [Guia de tipos de dados do Redis](https://redis.io/docs/data-types/)
- [Especificação do algoritmo Redlock](https://redis.io/docs/manual/patterns/distributed-locks/)
- [Documentação de persistência do Redis](https://redis.io/docs/manual/persistence/)
- [ioredis — cliente Redis em TypeScript](https://github.com/redis/ioredis)
- [Martin Fowler — Cache-Aside Pattern](https://martinfowler.com/patterns/cacheAside.html)
- [Melhores práticas de cache (AWS)](https://aws.amazon.com/caching/best-practices/)
- [Redis University — cursos gratuitos](https://university.redis.com/)
- Livro: *Redis in Action* por Josiah Carlson
