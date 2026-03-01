# Entrevistas de System Design

## Visão Geral

Entrevistas de system design avaliam sua capacidade de arquitetar um sistema de grande escala a partir de um enunciado vago. Diferente de entrevistas de coding, não há uma única resposta certa — o objetivo é demonstrar raciocínio estruturado, conhecimento de trade-offs e a capacidade de ter uma conversa técnica produtiva. Este capítulo fornece um framework repetível, técnicas comuns de estimativa e walkthroughs das perguntas de system design mais frequentes.

---

## Pré-requisitos

- Todos os capítulos anteriores do Track 08
- Track 05 — DevOps: containers, rede, fundamentos de CDN
- Track 07 — Security: auth, fundamentos de rate limiting

---

## Conceitos Fundamentais

### O framework de entrevista (RADISH)

Uma abordagem estruturada para qualquer problema de system design:

| Etapa | Ação | Tempo |
|-------|------|-------|
| **R** — Requirements | Esclareça requisitos funcionais e não-funcionais | 5 min |
| **A** — API | Defina a superfície da API | 5 min |
| **D** — Data model | Schema, escolhas de storage | 5 min |
| **I** — Infrastructure | Componentes de alto nível | 5 min |
| **S** — Scale | Lide com o volume de tráfego/dados declarado | 10 min |
| **H** — Harden | Gargalos, modos de falha, monitoramento | 5 min |

### Cheat sheet de estimativas

Memorize esses números para cálculos de back-of-envelope:

```
Números de latência:
  Cache L1:          ~1 ns
  Cache L2:          ~4 ns
  Acesso à RAM:      ~100 ns
  Leitura de SSD:    ~100 µs
  Leitura de HDD:    ~10 ms
  RTT de rede:       ~20 ms (mesma região)
  RTT de rede:       ~150 ms (cross-continente)

Throughput:
  Rede 1 Gbps:       ~125 MB/s
  SSD:               ~500 MB/s de leitura
  Postgres:          ~1.000–10.000 QPS (dependendo da complexidade da query)
  Redis:             ~100.000 ops/s

Storage:
  1 milhão de usuários × 1KB de perfil = 1 GB
  1 milhão de fotos × 3 MB = 3 TB
  1 char = 1 byte | 1 int = 4 bytes | 1 UUID = 16 bytes

Conversão de tempo:
  1 dia = 86.400 segundos ≈ 100.000 segundos (aproximação prática)
  1 ano = 31,5 milhões de segundos
```

---

## Walkthroughs de Entrevista

### Design de um encurtador de URLs (bit.ly)

**1. Requisitos**

Funcionais:
- Dado uma URL longa, gera uma URL curta
- Redireciona URL curta para a URL original
- (Opcional) Analytics: contagem de cliques, distribuição geográfica

Não-funcionais:
- 100M URLs criadas por dia
- Proporção Leitura:Escrita = 100:1 (leituras dominantes)
- URLs devem ficar disponíveis por 10 anos

**2. Estimativas**

```
Escritas: 100M / dia = 1.000 escritas/segundo
Leituras: proporção 100:1 = 100.000 leituras/segundo

Storage por URL: long_url (500 bytes) + short_code (7 bytes) + metadata (100 bytes) ≈ 700 bytes
Storage total: 100M URLs/dia × 10 anos × 700 bytes ≈ 255 TB
```

**3. Design da API**

```typescript
// Criar URL curta
POST /api/urls
Body: { longUrl: string, customAlias?: string, expiresAt?: string }
Response: { shortCode: string, shortUrl: string }

// Redirecionar
GET /:code → 301/302 para a URL original

// Analytics
GET /api/urls/:code/stats → { clicks: number, ... }
```

**4. Modelo de dados**

```sql
-- Tabela de URLs
CREATE TABLE urls (
  id          BIGSERIAL PRIMARY KEY,
  short_code  VARCHAR(8)   UNIQUE NOT NULL,
  long_url    TEXT         NOT NULL,
  user_id     UUID,
  created_at  TIMESTAMPTZ  DEFAULT now(),
  expires_at  TIMESTAMPTZ,
  click_count BIGINT       DEFAULT 0
);

CREATE INDEX ON urls (short_code);  -- caminho principal de busca
```

**5. Geração do short code**

```typescript
// Opção A: código base62 aleatório (7 chars = 62^7 = ~3,5 trilhões de combinações)
function generateCode(length = 7): string {
  const chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
  let result = '';
  const bytes = crypto.randomBytes(length);
  for (const byte of bytes) {
    result += chars[byte % chars.length];
  }
  return result;
}

// Opção B: base62 de ID auto-incremental (sem necessidade de verificação de colisão)
function encodeId(id: number): string {
  const chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
  let result = '';
  while (id > 0) {
    result = chars[id % 62] + result;
    id = Math.floor(id / 62);
  }
  return result.padStart(7, 'a');
}
```

**6. Arquitetura**

```
Cliente → CDN (cacheia redirecionamentos)
        → Load Balancer
        → Servidores de API (sem estado, escalonamento horizontal)
        → Redis (cache de URLs quentes, TTL=1h)
        → Postgres (fonte de verdade primária, read replica para leituras)
        → Kafka (eventos de clique para processamento assíncrono de analytics)
```

**7. Gargalos de escala**

- **Throughput de leitura:** 100K leituras/s. Solução: cache Redis com hit rate de 99%+. Fallback para read replica.
- **Throughput de escrita:** 1K escritas/s. Solução: um único primary Postgres lida com isso confortavelmente.
- **Escritas de analytics:** 100K eventos de clique/s. Solução: Kafka → inserção em batch, não atualização síncrona no DB.
- **URLs quentes:** 1% das URLs geram 99% do tráfego. Cache Redis com LRU lida com isso automaticamente.

---

### Design de um aplicativo de chat em tempo real (Slack/WhatsApp)

**1. Requisitos**

Funcionais:
- Enviar e receber mensagens em tempo real
- Chats em grupo (até 500 membros)
- Presença online/offline
- Histórico de mensagens
- Notificações push para usuários offline

Não-funcionais:
- 100M usuários ativos diários
- Cada usuário envia ~10 mensagens/dia
- Mensagens entregues em < 100ms

**2. Estimativas**

```
Mensagens/segundo: 100M usuários × 10 msgs / 86.400 segundos ≈ 12.000 msgs/segundo
Storage de mensagens: 12.000 msgs/s × 100 bytes × 86.400s × 365 dias ≈ 37 TB/ano
```

**3. Componentes**

```
Cliente (WebSocket)
  ↓
WebSocket Gateway (mantém conexões persistentes)
  ↓
Serviço de Mensagens
  ↓
Kafka (eventos de mensagem) → Serviço de Fan-out → Conexões WebSocket dos destinatários
                           → Serviço de Push Notification (Firebase/APNs)
                           → Message Store (Cassandra — write-heavy, time-series)

Serviço de Presença (Redis — status online baseado em TTL)
Media Storage (S3 + CDN para imagens, vídeos)
```

**4. Arquitetura WebSocket**

```typescript
// WebSocket gateway — mantém conexões de usuários
const connections = new Map<string, WebSocket>();

wss.on('connection', (ws, request) => {
  const userId = extractUserIdFromRequest(request);
  connections.set(userId, ws);

  // Atualiza presença
  await redis.setEx(`presence:${userId}`, 30, 'online');

  ws.on('message', async (data) => {
    const message = JSON.parse(data.toString());
    // Valida, persiste e faz fan-out
    await kafka.producer.send({
      topic: 'messages',
      messages: [{ value: JSON.stringify(message) }],
    });
  });

  ws.on('close', () => {
    connections.delete(userId);
    redis.del(`presence:${userId}`);
  });
});

// Worker de fan-out: entrega às conexões dos destinatários
kafka.consumer.subscribe({ topic: 'messages' });
await kafka.consumer.run({
  eachMessage: async ({ message }) => {
    const msg = JSON.parse(message.value.toString());
    for (const recipientId of msg.recipients) {
      const ws = connections.get(recipientId);
      if (ws?.readyState === WebSocket.OPEN) {
        ws.send(JSON.stringify(msg));
      } else {
        await sendPushNotification(recipientId, msg);
      }
    }
  },
});
```

**5. Escalando servidores WebSocket**

O desafio: conexões WebSocket são stateful. O usuário A está conectado ao Servidor 1, o usuário B ao Servidor 2. Uma mensagem de A para B deve ser roteada do Servidor 1 ao Servidor 2.

Solução: Redis Pub/Sub como barramento de mensagens entre servidores WebSocket:

```typescript
// Servidor 1: publica mensagem quando A envia para B
await redis.publish(`user:${recipientId}`, JSON.stringify(message));

// Servidor 2: inscrito no canal de B
redis.subscribe(`user:${userId}`, (message) => {
  const ws = localConnections.get(userId);
  ws?.send(message);
});
```

---

### Design do timeline do Twitter

**1. O problema central: fan-out**

Quando um usuário com 10M de seguidores twitta, como você mostra esse tweet nos 10M de timelines?

**Fan-out na escrita (modelo push):**
- Na criação do tweet, escreve no cache do timeline de cada seguidor
- Pro: leituras são O(1) — basta ler o cache do timeline
- Contra: 10M de seguidores × 1 escrita = muito lento para celebridades

**Fan-out na leitura (modelo pull):**
- Na leitura do timeline, busca tweets de todos os seguidos e faz merge
- Pro: sem amplificação de escrita
- Contra: leituras são lentas (merge de N timelines de seguidos)

**Abordagem híbrida (solução real do Twitter):**
- Contas pequenas (<100K seguidores): fan-out na escrita
- Contas de celebridades (>100K seguidores): fan-out na leitura, injetado nos timelines no momento da leitura

```typescript
async function getTimeline(userId: string): Promise<Tweet[]> {
  // 1. Obtém timeline pré-computada do cache (tweets de usuários regulares já injetados)
  const cachedTimeline = await redis.zRange(`timeline:${userId}`, 0, 20, { REV: true });

  // 2. Busca tweets de celebridades que não foram propagados por fan-out (merge na leitura)
  const followedCelebrities = await db.follow.findMany({
    where: { followerId: userId, followee: { followerCount: { gte: 100000 } } },
    select: { followeeId: true },
  });

  const celebrityTweets = await Promise.all(
    followedCelebrities.map((f) =>
      redis.zRange(`user-tweets:${f.followeeId}`, 0, 20, { REV: true })
    )
  );

  // 3. Faz merge e ordena por timestamp
  return mergeAndSort([...cachedTimeline, ...celebrityTweets.flat()]).slice(0, 20);
}
```

---

## Erros Comuns em Entrevistas

- **Não esclarecer requisitos** — pular para a solução sem entender a escala
- **Projetar para a escala errada** — perguntar "quantos usuários?" muda tudo
- **Ignorar modos de falha** — o que acontece quando o Redis cai? Quando o Kafka fica lento?
- **Não discutir trade-offs** — toda escolha tem prós e contras; articule-os
- **Over-engineering** — um cron job pode ser mais simples do que Kafka para sua escala
- **Sub-especificar o modelo de dados** — o schema frequentemente revela complexidade oculta

---

## Cenários do Mundo Real

**Como lidar com a pressão de "design X em 45 minutos":**

1. (5 min) Faça 4–5 perguntas de esclarecimento para estabelecer escala: DAU, proporção leitura/escrita, tamanho dos dados, requisitos de latência, requisitos de consistência
2. (5 min) Defina a API: 2–3 endpoints principais
3. (5 min) Desenhe o modelo de dados: 3–4 tabelas/coleções com índices-chave
4. (10 min) Diagrama de componentes de alto nível: cliente, load balancer, API, cache, DB, queue, CDN
5. (15 min) Deep dive na parte mais difícil (geralmente o problema de escalonamento)
6. (5 min) Discuta modos de falha e monitoramento

Sempre conduza a conversa. O entrevistador está avaliando se você consegue arquitetar um sistema, não se você tem uma resposta perfeita.

---

## Leitura Complementar

- [System Design Primer — GitHub](https://github.com/donnemartin/system-design-primer)
- [Designing Data-Intensive Applications](https://dataintensive.net/)
- [Blog High Scalability](http://highscalability.com/)
- [Newsletter ByteByteGo](https://blog.bytebytego.com/)
- Todos os capítulos anteriores do Track 08

---

## Resumo

Entrevistas de system design recompensam raciocínio estruturado mais do que conhecimento enciclopédico. Siga o framework RADISH: esclareça requisitos, defina a API, modele os dados, desenhe a infraestrutura, lide com a escala e fortifique contra falhas. Habilidades de estimativa são críticas — elas determinam se você precisa de Redis ou de uma simples coluna de banco de dados, se precisa de Kafka ou de um cron job. A habilidade mais importante é articular trade-offs: por que essa abordagem e não aquela, o que você ganha e o que você abre mão. Todo componente que você adiciona tem um custo; os melhores arquitetos minimizam componentes enquanto atendem aos requisitos.
