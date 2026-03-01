# Bancos de Dados: NoSQL

## 1. O Que é e Por Que Usar

"NoSQL" não significa "sem SQL" — significa "não apenas SQL." É um termo guarda-chuva para bancos de dados que divergem do modelo relacional tradicional em troca de diferentes trade-offs: schemas flexíveis, escalabilidade horizontal, otimização para padrões de acesso específicos ou modelos de dados especializados (documentos, key-value, wide-column, grafo).

Entender NoSQL não é sobre saber qual é "melhor que SQL" — é sobre escolher a ferramenta certa. PostgreSQL trata 90% da maioria das aplicações perfeitamente. Os 10% restantes frequentemente têm requisitos específicos: lookups de latência ultra-baixa (Redis), throughput de escrita massivo com queries simples (Cassandra), schemas flexíveis em evolução (MongoDB) ou dados de relacionamentos profundamente conectados (Neo4j).

Este arquivo cobre as quatro principais famílias de NoSQL, quando usar cada uma, o teorema CAP que governa seus trade-offs fundamentais e os padrões práticos que importam em produção.

---

## 2. Conceitos Fundamentais

### Teorema CAP

O teorema CAP afirma que um data store distribuído só pode fornecer **duas** das três garantias a seguir simultaneamente — especialmente durante uma partição de rede:

- **Consistência (C):** Cada leitura recebe a escrita mais recente ou um erro. Todos os nós veem os mesmos dados ao mesmo tempo.
- **Disponibilidade (A):** Cada requisição recebe uma resposta (não necessariamente os dados mais recentes). O sistema continua funcionando mesmo se alguns nós estiverem fora.
- **Tolerância a Partição (P):** O sistema continua operando mesmo quando partições de rede (perda de mensagens entre nós) ocorrem.

Na prática, partições de rede são inevitáveis em sistemas distribuídos. Então a escolha real é: **durante uma partição, você sacrifica Consistência (AP) ou Disponibilidade (CP)?**

```
Sistemas CP (consistência sobre disponibilidade):
  - MongoDB (padrão — primário recusa leituras/escritas se não puder confirmar quorum)
  - HBase, Zookeeper
  - Escolha quando: correção dos dados não é negociável (bancário, inventário)

Sistemas AP (disponibilidade sobre consistência):
  - Cassandra, CouchDB, DynamoDB
  - Escolha quando: disponibilidade importa mais que ver os dados absolutamente mais recentes
    (carrinhos de compra, preferências de usuário, métricas)
  - Eventualmente consistente: todos os nós convergirão para o mesmo estado com o tempo

Sistemas CA (sem tolerância a partição):
  - RDBMS single-node tradicional (PostgreSQL, MySQL sem replicação)
  - Válido apenas em ambientes de um datacenter com rede controlada
```

> CAP é frequentemente simplificado demais. Sistemas reais fazem escolhas mais matizadas: MongoDB permite configurar read/write concerns por operação, Cassandra permite ajustar nível de consistência por query. PACELC estende CAP para também modelar trade-offs de latência vs. consistência.

### BASE vs ACID

| Propriedade | ACID | BASE |
|-------------|------|------|
| Modelo | Consistência forte | Consistência eventual |
| Estado | Sempre consistente | Estado soft (pode ser temporariamente inconsistente) |
| Disponibilidade | Pode sacrificar pela consistência | Prioriza disponibilidade |
| Transações | ACID multi-registro | Frequentemente operações de registro único |
| Caso de uso | Financeiro, transacional | Feeds sociais, métricas, atividade de usuário |

**BASE:** Basically Available (Basicamente Disponível), Soft state (Estado Soft), Eventually consistent (Eventualmente consistente).

---

## 3. Document Stores (MongoDB)

MongoDB armazena dados como documentos BSON (Binary JSON) agrupados em coleções. Documentos na mesma coleção podem ter schemas diferentes.

### Design de Schema: Embed vs Reference

Esta é a decisão de design mais importante no MongoDB.

**Embed quando:**
- Dados são sempre acessados juntos (sem necessidade de join)
- Relacionamento é 1:1 ou 1:poucos (não 1:muitos-milhares)
- Dados embutidos não mudam independentemente com alta frequência
- Tamanho total do documento fica < 16MB (limite do MongoDB)

```javascript
// Embutido: post + comentários (acessados juntos, contagem de comentários gerenciável)
{
  "_id": ObjectId("..."),
  "title": "Getting Started with MongoDB",
  "author": { "id": "user_42", "name": "Alice" },  // snapshot embutido
  "tags": ["mongodb", "nosql"],
  "comments": [
    { "author": "Bob", "text": "Great post!", "createdAt": ISODate("...") },
    { "author": "Carol", "text": "Very helpful", "createdAt": ISODate("...") }
  ]
}
```

**Reference quando:**
- Relacionamentos muitos:muitos
- Documentos referenciados são grandes ou mudam independentemente
- A contagem do relacionamento é ilimitada (1:milhões)
- Dados precisam ser consultados independentemente

```javascript
// Referenciado: perfil de usuário → pedidos (pedidos podem ser milhões)
// Coleção Users:
{ "_id": ObjectId("user_42"), "name": "Alice", "email": "alice@example.com" }

// Coleção Orders:
{ "_id": ObjectId("order_1"), "userId": ObjectId("user_42"), "total": 149.99 }
{ "_id": ObjectId("order_2"), "userId": ObjectId("user_42"), "total": 89.00 }
```

> Diferente do SQL, o MongoDB NÃO impõe integridade referencial. Se você deletar um usuário, os pedidos dele ainda referenciam o ID de usuário deletado. Seu código de aplicação deve gerenciar essa consistência.

### Aggregation Pipeline

```javascript
// Analytics complexo: receita por categoria de produto nos últimos 30 dias
db.orders.aggregate([
  // Estágio 1: Filtro
  { $match: {
    status: 'completed',
    createdAt: { $gte: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000) }
  }},

  // Estágio 2: Unwind array (um doc por item)
  { $unwind: '$items' },

  // Estágio 3: Join com coleção de produtos
  { $lookup: {
    from: 'products',
    localField: 'items.productId',
    foreignField: '_id',
    as: 'product'
  }},
  { $unwind: '$product' },

  // Estágio 4: Agrupa por categoria
  { $group: {
    _id: '$product.category',
    totalRevenue: { $sum: { $multiply: ['$items.price', '$items.quantity'] } },
    orderCount: { $sum: 1 },
    avgOrderValue: { $avg: { $multiply: ['$items.price', '$items.quantity'] } }
  }},

  // Estágio 5: Formata a saída
  { $project: {
    category: '$_id',
    totalRevenue: { $round: ['$totalRevenue', 2] },
    orderCount: 1,
    avgOrderValue: { $round: ['$avgOrderValue', 2] },
    _id: 0
  }},

  // Estágio 6: Ordena por receita descendente
  { $sort: { totalRevenue: -1 } }
])
```

### Índices no MongoDB

```javascript
// Índice de campo único
db.users.createIndex({ email: 1 });  // 1 = ascendente, -1 = descendente

// Índice composto (a regra do prefixo mais à esquerda se aplica)
db.orders.createIndex({ userId: 1, createdAt: -1 });

// Índice único
db.users.createIndex({ email: 1 }, { unique: true });

// Índice parcial (MongoDB 3.2+)
db.orders.createIndex(
  { userId: 1, createdAt: -1 },
  { partialFilterExpression: { status: 'active' } }
);

// Índice de texto (full-text search)
db.articles.createIndex({ title: 'text', body: 'text' });
db.articles.find({ $text: { $search: 'mongodb performance' } });

// Índice geoespacial
db.stores.createIndex({ location: '2dsphere' });
db.stores.find({
  location: { $near: {
    $geometry: { type: 'Point', coordinates: [-73.9857, 40.7484] },
    $maxDistance: 5000  // metros
  }}
});
```

### Transações Multi-Documento (MongoDB 4.0+)

```javascript
const session = await client.startSession();
try {
  await session.withTransaction(async () => {
    await db.accounts.updateOne(
      { _id: fromId },
      { $inc: { balance: -amount } },
      { session }
    );
    await db.accounts.updateOne(
      { _id: toId },
      { $inc: { balance: amount } },
      { session }
    );
    await db.transactions.insertOne(
      { fromId, toId, amount, createdAt: new Date() },
      { session }
    );
  });
} finally {
  await session.endSession();
}
```

> Transações multi-documento no MongoDB têm overhead de performance significativo. Projete seu schema para evitar precisar delas (use documentos embutidos para atualizações atômicas quando possível). Use transações apenas quando o invariante genuinamente abrange múltiplos documentos/coleções.

---

## 4. Key-Value Stores (Redis)

Redis armazena dados na memória com persistência opcional. Operações são O(1) a O(log n) para a maioria dos tipos de dados.

### Tipos de Dados e Casos de Uso

```typescript
import Redis from 'ioredis';
const redis = new Redis(process.env.REDIS_URL);

// --- Strings: cache simples, contadores ---
await redis.set('user:42:name', 'Alice');
await redis.setex('session:abc123', 3600, JSON.stringify(sessionData)); // TTL 1hr
await redis.incr('page:views:/home'); // contador atômico
await redis.incrby('user:42:credits', 50);

// --- Hashes: objetos, sessões de usuário ---
await redis.hset('user:42', 'name', 'Alice', 'email', 'alice@example.com');
await redis.hset('user:42', { plan: 'pro', logins: '0' });
const user = await redis.hgetall('user:42');
await redis.hincrby('user:42', 'logins', 1);

// --- Lists: filas, feeds de atividade (FIFO/LIFO) ---
await redis.lpush('notifications:42', JSON.stringify({ type: 'like', postId: '1' }));
const notifications = await redis.lrange('notifications:42', 0, 9); // últimas 10
await redis.ltrim('notifications:42', 0, 99); // mantém apenas as 100 mais recentes

// --- Sets: visitantes únicos, tags, relacionamentos ---
await redis.sadd('post:1:viewers', userId);
const viewCount = await redis.scard('post:1:viewers');
const commonFriends = await redis.sinter('user:1:friends', 'user:2:friends');

// --- Sorted Sets: leaderboards, rate limiting ---
await redis.zadd('leaderboard:season1', score, userId);
const top10 = await redis.zrevrange('leaderboard:season1', 0, 9, 'WITHSCORES');
const userRank = await redis.zrevrank('leaderboard:season1', userId);

// --- Streams: log de eventos, pub/sub com persistência ---
const id = await redis.xadd('events', '*', 'type', 'page_view', 'path', '/home', 'userId', '42');
const events = await redis.xrange('events', '-', '+', 'COUNT', 100);
```

### Persistência

```
RDB (Redis Database): Snapshots em point-in-time
  save 900 1    # snapshot se 1+ chaves mudaram em 15 minutos
  save 300 10   # snapshot se 10+ chaves mudaram em 5 minutos
  save 60 10000 # snapshot se 10000+ chaves mudaram em 1 minuto
  Prós: compacto, restarts rápidos, impacto mínimo de I/O
  Contras: potencial perda de dados entre snapshots

AOF (Append-Only File): registra toda operação de escrita
  appendonly yes
  appendfsync everysec  # fsync a cada segundo (compromisso)
  # appendfsync always  # mais seguro mas mais lento
  # appendfsync no      # mais rápido, OS decide quando fazer flush
  Prós: perda mínima de dados (no máximo 1 segundo com everysec)
  Contras: arquivos maiores, restarts mais lentos

Combinado: use ambos para melhor durabilidade + restart rápido
```

### Políticas de Eviction

```
noeviction:      Retorna erro quando memória cheia. Bom para data store primário.
allkeys-lru:     Evita chaves menos recentemente usadas. Bom para cache puro.
volatile-lru:    LRU apenas nas chaves com TTL. Evita cache, mantém dados persistentes.
allkeys-lfu:     Evita o menos frequentemente usado. Melhor que LRU para acesso inclinado.
volatile-ttl:    Evita chaves com menor TTL restante primeiro.
allkeys-random:  Eviction aleatório. Raramente útil.
```

> Para um cache puro (todas as chaves têm TTL), use `allkeys-lru` ou `allkeys-lfu`. Para um store misto (alguns em cache, alguns persistentes), use `volatile-lru` para proteger chaves persistentes de eviction.

---

## 5. Wide-Column Stores (Cassandra)

Cassandra é otimizado para throughput de escrita massivo em múltiplos datacenters sem ponto único de falha.

### Modelo de Dados

O modelo de dados do Cassandra é orientado a padrões de query, não a relacionamentos de domínio. Projete tabelas para as queries que você vai executar.

```cql
-- Partition key: determina qual nó armazena os dados (consistent hashing)
-- Clustering key: determina a ordem de classificação dentro de uma partição
-- Regra: queries devem sempre especificar a partition key

CREATE TABLE events_by_user (
  user_id     UUID,
  occurred_at TIMESTAMP,
  event_id    UUID,
  event_type  TEXT,
  payload     TEXT,
  PRIMARY KEY (user_id, occurred_at, event_id)
)
WITH CLUSTERING ORDER BY (occurred_at DESC);

-- Eficiente: partition key especificada
SELECT * FROM events_by_user
WHERE user_id = ? AND occurred_at >= ? AND occurred_at <= ?;

-- Ineficiente / não permitido: sem partition key — scan de todo o cluster
-- SELECT * FROM events_by_user WHERE event_type = 'click'; ← ALLOW FILTERING necessário
```

### Níveis de Consistência

```cql
-- Escrita com quorum (maioria das réplicas deve confirmar)
INSERT INTO events_by_user ... USING CONSISTENCY QUORUM;

-- Leitura com quorum
SELECT ... FROM events_by_user WHERE ... USING CONSISTENCY QUORUM;

-- Consistência eventual (mais rápido, pode ler dados stale)
SELECT ... USING CONSISTENCY ONE;

-- Consistência forte: write QUORUM + read QUORUM garante dados mais recentes
-- (requer: reads + writes > fator de replicação)
```

Cassandra é **otimizado para escrita**: escritas são anexadas a um commit log e a um memtable (estrutura in-memory), tornando escritas O(1). Leituras são mais caras — os dados podem estar no memtable, em SSTables ou requerem verificação de bloom filter.

---

## 6. Graph Databases (Neo4j)

Graph databases armazenam dados como nós (entidades) e relacionamentos (arestas entre entidades), cada um com propriedades.

```cypher
// Cria nós e relacionamento
CREATE (alice:User {id: '1', name: 'Alice', city: 'NYC'})
CREATE (bob:User {id: '2', name: 'Bob', city: 'LA'})
CREATE (alice)-[:FOLLOWS {since: date('2024-01-01')}]->(bob)

// Encontra amigos de amigos (travessia de 2 hops)
MATCH (me:User {id: $userId})-[:FOLLOWS]->(friend)-[:FOLLOWS]->(fof)
WHERE fof <> me AND NOT (me)-[:FOLLOWS]->(fof)
RETURN fof.name, COUNT(*) as mutual_friends
ORDER BY mutual_friends DESC
LIMIT 10

// Caminho mais curto entre dois usuários
MATCH path = shortestPath(
  (alice:User {id: '1'})-[:FOLLOWS*]-(target:User {id: '99'})
)
RETURN path

// Recomendação: produtos comprados por usuários que compraram o que eu comprei
MATCH (me:User {id: $userId})-[:PURCHASED]->(p:Product)<-[:PURCHASED]-(similar:User)
      -[:PURCHASED]->(recommended:Product)
WHERE NOT (me)-[:PURCHASED]->(recommended)
RETURN recommended.name, COUNT(*) as score
ORDER BY score DESC LIMIT 10
```

**Quando graph databases se destacam:**
- Redes sociais (recomendações de amigos, caminhos de conexão)
- Detecção de fraude (detectando padrões incomuns em grafos de transação)
- Knowledge graphs
- Controle de acesso com hierarquias de roles complexas
- Planejamento de rotas e logística

> Se a travessia de relacionamentos é o padrão de query central — não apenas um JOIN ocasional — um graph database elimina o custo exponencial de JOINs multi-hop em SQL. Uma travessia de 6 hops em SQL exigiria 6 JOINs; Cypher trata isso naturalmente.

---

## 7. Design de Schema MongoDB em Profundidade

```typescript
// Padrão: Bucket pattern para dados de time-series
// Ao invés de um documento por evento (milhões de documentos), agrupa eventos por bucket de tempo

// Ruim: um documento por medição (10.000 documentos/dia por sensor)
{
  "_id": ObjectId("..."),
  "sensorId": "sensor_42",
  "temperature": 22.5,
  "recordedAt": ISODate("2024-01-15T14:30:00Z")
}

// Bom: padrão bucket — um documento por hora por sensor (288 documentos/dia)
{
  "_id": ObjectId("..."),
  "sensorId": "sensor_42",
  "date": ISODate("2024-01-15T14:00:00Z"),
  "measurements": [
    { "minute": 0, "temperature": 22.5 },
    { "minute": 1, "temperature": 22.6 },
    // ... até 60 medições
  ],
  "stats": {
    "min": 22.1,
    "max": 23.4,
    "avg": 22.7,
    "count": 60
  }
}

// Padrão: Computed pattern — pré-agrega estatísticas lidas frequentemente
{
  "_id": ObjectId("post_123"),
  "title": "Getting Started",
  "commentCount": 47,    // ← desnormalizado, atualizado ao criar/deletar comentário
  "likeCount": 312,
  "viewCount": 9841
}
// Evita COUNT(*) a cada carregamento de página
```

---

## 8. Erros Comuns e Armadilhas

**Arrays embutidos ilimitados no MongoDB:**

```javascript
// ERRADO: comentários crescem sem limite — documento atinge o limite de 16MB
{ _id: '...', title: 'Viral Post', comments: [/* milhões de comentários */] }

// CORRETO: referencia comentários separadamente quando a contagem é ilimitada
// Documento Post: commentCount: 15420 (pré-computado)
// Coleção Comments: { postId: '...', text: '...', createdAt: ... }
```

**Ignorar o requisito de partition key do Cassandra:**

```cql
-- ERRADO: vai falhar ou exigir ALLOW FILTERING (scan completo do cluster)
SELECT * FROM events WHERE event_type = 'purchase';

-- CORRETO: sempre projete tabelas em torno dos padrões de query
-- Crie uma tabela separada para este padrão de query
CREATE TABLE events_by_type (
  event_type TEXT,
  occurred_at TIMESTAMP,
  event_id UUID,
  user_id UUID,
  PRIMARY KEY (event_type, occurred_at, event_id)
);
```

**Usar Redis como banco primário sem persistência:**

```
Se Redis é seu único data store e você usa as configurações padrão (sem AOF, sem RDB),
um restart perde TODOS os dados.

Mínimo para store primário:
  appendonly yes
  appendfsync everysec
  requirepass <senha-forte>

Melhor: use Redis como cache + um banco durável como fonte da verdade
```

**MongoDB sem indexação:**

```javascript
// ERRADO: querying uma coleção de 10M documentos sem índice
db.orders.find({ userId: 'user_42', status: 'pending' }).sort({ createdAt: -1 });
// → Seq scan, potencialmente segundos

// CORRETO: índice nos campos da query
db.orders.createIndex({ userId: 1, status: 1, createdAt: -1 });
// → Index scan, milissegundos

// Sempre execute explain() para verificar
db.orders.find({ userId: 'user_42', status: 'pending' })
  .explain('executionStats');
// Procure por: COLLSCAN (ruim) vs IXSCAN (bom)
```

---

## 9. Quando NoSQL vs SQL: Matriz de Decisão

| Cenário | Recomendação | Raciocínio |
|---------|-------------|------------|
| Contas de usuário + pedidos + produtos | PostgreSQL | Relacionamentos complexos, transações ACID |
| Catálogo de produtos com atributos variáveis | MongoDB | Schema flexível, schema varia por tipo de produto |
| Sessões de usuário e cache | Redis | In-memory, ops O(1), suporte a TTL |
| Dados de sensores IoT (bilhões de eventos) | Cassandra ou TimescaleDB | Write-heavy, time-series, escala horizontal |
| Grafo social, detecção de fraude | Neo4j | Travessia é o padrão de query principal |
| Leaderboards em tempo real | Redis sorted sets | Ranking O(log n), atualizações atômicas |
| Full-text search | Elasticsearch ou PostgreSQL GIN | Índice invertido, scoring de relevância |
| E-commerce com inventário | PostgreSQL | ACID para decremento de estoque, queries complexas |
| Gestão de conteúdo, CMS | MongoDB ou PostgreSQL | Modelo de documento OU JSONB |
| Analytics/relatórios | PostgreSQL ou ClickHouse | Window functions, agregações |

---

## 10. Cenário Real

**Problema:** Uma plataforma de mídia social armazena eventos de atividade de usuário (curtidas, comentários, compartilhamentos, visualizações) em uma única tabela PostgreSQL. Com 500 milhões de eventos diários, a tabela cresceu para 2 bilhões de linhas. Escritas estão lentas, leituras atingem timeout e o VACUUM não consegue acompanhar.

**Análise:**
- Eventos são apenas append — nunca atualizados.
- Query principal: "me dá todos os eventos do usuário X nos últimos 7 dias" (partition key = userId + date).
- Retenção: 90 dias depois arquiva para armazenamento frio.
- Sem JOINs complexos necessários.

**Migração para Cassandra:**

```cql
CREATE TABLE user_activity (
  user_id     UUID,
  bucket_date DATE,
  occurred_at TIMESTAMP,
  event_id    UUID,
  event_type  TEXT,   -- 'like', 'comment', 'share', 'view'
  target_id   TEXT,
  metadata    TEXT,   -- JSON blob
  PRIMARY KEY ((user_id, bucket_date), occurred_at, event_id)
)
WITH CLUSTERING ORDER BY (occurred_at DESC)
AND default_time_to_live = 7776000;  -- TTL de 90 dias, auto-delete

-- Escrita (da aplicação)
INSERT INTO user_activity (user_id, bucket_date, occurred_at, event_id, event_type, target_id)
VALUES (?, ?, toTimestamp(now()), uuid(), ?, ?)
USING TTL 7776000;

-- Leitura: últimos 7 dias de atividade de um usuário
SELECT * FROM user_activity
WHERE user_id = ? AND bucket_date IN (?, ?, ?, ?, ?, ?, ?)
ORDER BY occurred_at DESC
LIMIT 100;
```

**Resultado:** Escritas passam de 800ms em média para 2ms. Latência de leitura cai de 30 segundos (com timeouts) para 8ms. TTL automático cuida da expiração de dados sem manutenção.

---

## 11. Perguntas de Entrevista

**Q1: Explique o teorema CAP em termos práticos.**

Um sistema distribuído durante uma partição de rede deve escolher: ou recusar requisições (para manter consistência) ou servir dados potencialmente desatualizados (para manter disponibilidade). O MongoDB tem padrão CP — o primário para de aceitar escritas se não puder confirmar um quorum de réplicas, garantindo que todas as leituras obtenham os dados mais recentes. Cassandra é AP — continua aceitando escritas e leituras mesmo se alguns nós estiverem inacessíveis, com o trade-off de que algumas leituras podem retornar dados stale até os nós se reconciliarem.

**Q2: Quando você deve embed vs referenciar documentos no MongoDB?**

Embed quando: dados são sempre acessados juntos (post + seus 5 comentários mais recentes), a contagem do relacionamento é limitada e pequena (< algumas centenas), e os dados embutidos não mudam independentemente com alta frequência. Referenicie quando: o relacionamento é muitos-para-muitos, o array é ilimitado (pode crescer para milhares), o documento referenciado é grande e frequentemente consultado independentemente, ou você precisa consultar os documentos referenciados sem passar pelo pai. A pergunta chave: "Eu sempre preciso de ambos os dados juntos, ou às vezes preciso deles separadamente?"

**Q3: Quando você escolheria Cassandra ao invés de PostgreSQL?**

Cassandra é ideal quando: throughput de escrita é extremo (milhões de escritas por segundo), dados são apenas append com padrões de acesso simples (time-series, logs de atividade, dados de sensores IoT), distribuição geográfica multi-datacenter é necessária sem ponto único de falha, e você pode projetar suas tabelas em torno de padrões de query específicos. PostgreSQL é melhor quando você precisa de queries complexas, JOINs, transações ACID multi-registro ou analytics ad-hoc. A restrição chave do Cassandra: toda query deve especificar a partition key, então você pré-projeta tabelas para queries conhecidas.

**Q4: O que é consistência eventual e quais problemas ela resolve vs cria?**

Consistência eventual significa que todos os nós eventualmente convergirão para os mesmos dados se as escritas pararem, mas em qualquer momento diferentes nós podem retornar valores diferentes. Resolve: disponibilidade (o sistema continua funcionando mesmo quando alguns nós estão inacessíveis), distribuição global (nós podem aceitar escritas localmente sem coordenar com datacenters distantes em tempo real). Problemas que cria: ler suas próprias escritas pode falhar (você escreve no nó A, depois lê do nó B antes da replicação), e escritas concorrentes no mesmo registro podem conflitar (requer estratégia de resolução de conflito: last-write-wins, vector clocks ou merge no nível da aplicação).

**Q5: Para uma plataforma de e-commerce (usuários, produtos, pedidos, inventário), você usaria MongoDB ou PostgreSQL?**

PostgreSQL. E-commerce requer: transações ACID para decrementos de inventário (prevenindo oversell), JOINs complexos (histórico de pedidos com detalhes do produto, analytics de clientes), integridade referencial (foreign keys entre pedidos e usuários/produtos) e relatórios ad-hoc. MongoDB pode funcionar mas requer gerenciamento de transação no nível da aplicação e perde o benefício do SQL para analytics. A exceção: atributos de catálogo de produtos (variáveis por tipo de produto) são bem adequados para o schema flexível do MongoDB — muitas plataformas de e-commerce usam PostgreSQL para transações e MongoDB (ou JSONB no Postgres) para catálogo de produtos.

**Q6: Quando você escolheria um graph database?**

Quando o valor principal dos seus dados está nos relacionamentos entre entidades, não nas entidades em si. Redes sociais (recomendações de amigos, caminho mais curto de conexão), detecção de fraude (detectar anéis de transações suspeitas), hierarquias de controle de acesso com herança complexa e knowledge graphs. A pergunta decisiva: "Meu padrão de query central é travessia de relacionamentos multi-hop?" Se uma query SQL para seu caso de uso exigiria 5+ JOINs ou CTEs recursivas, um graph database vale a pena considerar.

**Q7: Compare BASE e ACID.**

ACID (típico de bancos SQL): Atomicidade (tudo ou nada), Consistência (restrições sempre mantidas), Isolamento (transações parecem seriais), Durabilidade (dados confirmados sobrevivem a crashes). Fornece garantias fortes ao custo de disponibilidade e throughput reduzidos em configurações distribuídas. BASE (típico de NoSQL): Basically Available (sistema permanece operacional), Soft state (dados podem ser temporariamente inconsistentes), Eventually consistent (todos os nós convergem dado o tempo). Troca garantias de correção por maior disponibilidade e performance. Use ACID para sistemas financeiros, médicos e de inventário; BASE para feeds sociais, logs de atividade e preferências de usuário.

---

## 12. Exercícios

**Exercício 1: Design de schema MongoDB para um blog**

Projete um schema MongoDB para um blog com:
- Posts (título, body, tags, informações do autor)
- Comentários (texto, autor, reações)
- Tags com contagens de posts

Requisitos: visualizar um post mostra seu conteúdo + autor + primeiros 10 comentários; navegar por tag mostra posts ordenados por popularidade; perfil do autor mostra seus posts recentes. Projete documentos, decisões de embedding e índices. Justifique cada decisão de embed vs reference.

*Dica:* Posts devem embed um snapshot do autor (não o objeto de usuário completo) e contagem de comentários como campo pré-computado. Comentários devem ser uma coleção separada (ilimitada). Tags devem ser uma coleção separada com campo `postCount`.

**Exercício 2: Rate limiter com Redis**

Implemente um rate limiter usando sorted sets do Redis que:
- Permite 100 requisições por usuário por minuto
- Usa uma janela deslizante (não janela fixa)
- Retorna cota restante e tempo de reset nos headers
- Trata falhas de conexão ao Redis com graciosidade (fail open)

*Dica:* Use `ZADD` com timestamp como score, `ZREMRANGEBYSCORE` para remover entradas antigas, `ZCARD` para contar entradas da janela atual. Envolva em script Lua para atomicidade.

**Exercício 3: Tabela de time-series no Cassandra**

Projete uma tabela Cassandra para armazenar eventos de analytics de website (page views, cliques, conversões) com requisitos:
- Query por website + intervalo de data (ex.: últimos 30 dias)
- Query por website + tipo de evento + data
- Armazena no máximo 1 ano de dados (use TTL)
- Suporta 100.000 escritas por segundo em 10 websites

Projete o schema da tabela, partition key, clustering key e queries de padrão de acesso. Explique por que você escolheu esta estratégia de partição ao invés de alternativas.

---

## 13. Leitura Complementar

- [Guia de modelagem de dados do MongoDB](https://www.mongodb.com/docs/manual/data-modeling/)
- [Modelagem de dados do Cassandra — DataStax](https://docs.datastax.com/en/cassandra-oss/3.0/cassandra/dml/dmlAboutDataModels.html)
- [Documentação de tipos de dados do Redis](https://redis.io/docs/data-types/)
- [Manual Cypher do Neo4j](https://neo4j.com/docs/cypher-manual/)
- [Teorema PACELC (estende o CAP)](https://dl.acm.org/doi/10.1145/2360276.2360337)
- [Martin Fowler — NoSQL Distilled (livro)](https://martinfowler.com/books/nosql.html)
- [Designing Data-Intensive Applications por Martin Kleppmann](https://dataintensive.net/) — capítulos sobre replicação e consistência
- [MongoDB University (cursos gratuitos)](https://learn.mongodb.com/)
- [The Little Redis Book (gratuito)](http://openmymind.net/redis.pdf)
