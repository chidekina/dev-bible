# Desafios de Backend

> Desafios práticos de backend em contexto Node.js/TypeScript usando Fastify (ou Express onde indicado). Cada cenário reflete uma situação real de produção. Foque na corretude primeiro, depois segurança, depois performance. Stack assumida: Node.js 20+, TypeScript, Fastify, Zod, Prisma (ou SQL direto), Redis.

---

## Exercício 1 — Projetar uma API REST de Recursos (Fácil)

**Cenário:** Você está construindo o backend de um app de gerenciamento de tarefas. Você precisa expor um recurso `tasks` com operações CRUD completas.

**Requisitos:**
- Endpoints: `GET /tasks`, `GET /tasks/:id`, `POST /tasks`, `PUT /tasks/:id`, `DELETE /tasks/:id`.
- Uma tarefa tem: `id`, `title` (obrigatório), `description` (opcional), `status` (`todo | in_progress | done`), `dueDate` (string ISO, opcional), `createdAt`.
- `GET /tasks` suporta filtragem por `status` e paginação (query params `page`, `limit`, limit padrão 20, máximo 100).
- Todos os corpos de requisição são validados com schemas Zod.
- Retornar os códigos HTTP adequados (201 ao criar, 204 ao deletar, 404 quando não encontrado, 422 em falha de validação).

**Critérios de Aceite:**
- [ ] `POST /tasks` com `title` ausente retorna `422` com um corpo de erro estruturado (erros por campo).
- [ ] `GET /tasks?status=invalid` retorna `422`, não `500`.
- [ ] `DELETE /tasks/:id` em um recurso inexistente retorna `404`.
- [ ] A resposta de `GET /tasks` inclui `{ data: Task[], meta: { page, limit, total } }`.
- [ ] Todos os endpoints têm schema Zod registrado no nível da rota no Fastify (sem validação ad-hoc dentro dos handlers).

**Dicas:**
1. Registre um schema Zod para cada rota via `fastify.route({ schema: { body: zodToJsonSchema(taskSchema) } })`.
2. Use uma camada `taskService` — as rotas chamam métodos do service, o service chama o repositório. Sem lógica de negócio nos handlers de rota.
3. Para paginação: `SELECT * FROM tasks LIMIT $limit OFFSET ($page - 1) * $limit`. Sempre rode uma query `COUNT(*)` para o total.
4. Centralize respostas 404/422 em um helper compartilhado `reply.badRequest` / `reply.notFound` ou em uma classe de erro customizada.

---

## Exercício 2 — Fluxo de Autenticação com JWT (Médio)

**Cenário:** Implemente um sistema completo de auth: registro, login, refresh token e logout.

**Requisitos:**
- `POST /auth/register`: cria um usuário com senha hasheada (bcrypt, custo 12). Retorna `201`.
- `POST /auth/login`: valida credenciais, retorna `{ accessToken, refreshToken }`. Access token expira em 15 minutos, refresh token em 7 dias.
- `POST /auth/refresh`: aceita um refresh token válido no corpo da requisição, retorna um novo access token.
- `POST /auth/logout`: invalida o refresh token (blacklist no Redis com o TTL restante do token).
- Rotas protegidas exigem um header `Authorization: Bearer <token>` válido.

**Critérios de Aceite:**
- [ ] Senhas nunca são armazenadas em texto puro — verifique com uma query direta ao banco.
- [ ] Access tokens são assinados com `HS256` usando um `JWT_SECRET` fornecido por variável de ambiente.
- [ ] Refresh tokens são armazenados no Redis com TTL de 7 dias.
- [ ] Após `POST /auth/logout`, o refresh token não pode ser usado para obter um novo access token.
- [ ] Login falho (senha errada) retorna `401`, não `403`.
- [ ] Proteção contra força bruta: após 5 logins falhos para o mesmo email em 15 minutos, retornar `429`.

**Dicas:**
1. Use `jsonwebtoken` (ou `@fastify/jwt`) para assinatura e verificação de tokens.
2. Armazene o refresh token no Redis com `SET rt:<userId>:<tokenId> 1 EX 604800`. No refresh, verifique se a chave existe. No logout, delete-a.
3. Para o contador de força bruta: `INCR login_fail:<email>` com `EXPIRE login_fail:<email> 900`. Verifique a contagem antes de validar a senha.
4. Nunca compare senhas com `===` — sempre use `bcrypt.compare`.

---

## Exercício 3 — Middleware de Rate Limiting (Médio)

**Cenário:** Implemente um rate limiter de sliding window como plugin Fastify usando Redis. Deve suportar limites por IP e por usuário com diferentes thresholds.

**Requisitos:**
- Requisições anônimas (não autenticadas): 60 requisições por minuto por IP.
- Requisições autenticadas: 300 requisições por minuto por ID de usuário.
- Retornar `429 Too Many Requests` com header `Retry-After` quando o limite for excedido.
- Os limites são armazenados e verificados no Redis.
- O plugin deve ser configurável: `{ windowMs, maxRequests, keyPrefix }`.

**Critérios de Aceite:**
- [ ] Uma rajada de 61 requisições anônimas em 60 segundos resulta na 61ª sendo rejeitada com `429`.
- [ ] O valor do header `Retry-After` (em segundos) está correto.
- [ ] Usuários autenticados têm um limite separado e maior do que usuários anônimos.
- [ ] Falha do Redis (erro de conexão) abre o caminho (permite a requisição) com um aviso registrado em log — NÃO falhe toda a requisição.
- [ ] Teste unitário da lógica do rate limiter independentemente do Fastify.

**Dicas:**
1. Sliding window com Redis: use um pipeline `ZADD` + `ZREMRANGEBYSCORE` + `ZCOUNT`. Adicione o timestamp atual como score e membro, remova entradas mais antigas que `windowMs`, conte as restantes.
2. `ZADD key NX score member` seguido de `EXPIRE key windowSeconds` em um script Lua garante atomicidade.
3. Formato da chave: `rl:<keyPrefix>:<identifier>` onde identifier é IP ou userId.
4. Fail open: envolva todas as chamadas Redis em `try/catch`. Em erro, registre com `logger.warn` e chame `next()`.

---

## Exercício 4 — Handler de Webhook com Verificação de Assinatura (Médio)

**Cenário:** Seu app recebe webhooks de um provedor de pagamento (estilo Stripe). Cada requisição inclui um header `X-Webhook-Signature` que você deve verificar antes de processar.

**Requisitos:**
- `POST /webhooks/payment`: recebe eventos de pagamento.
- Assinatura: HMAC-SHA256 do corpo bruto da requisição usando um segredo compartilhado (variável de ambiente `WEBHOOK_SECRET`). O formato do header é `sha256=<hex_digest>`.
- Se a assinatura for inválida, retornar `401`. Se a estrutura do payload for inválida, retornar `422`.
- Processar estes tipos de evento: `payment.succeeded`, `payment.failed`, `payment.refunded`.
- Em `payment.succeeded`: atualizar o status do pedido no banco, enviar email de confirmação (assíncrono, não-bloqueante).
- Idempotência: se o mesmo ID de evento for recebido duas vezes, pular o processamento e retornar `200`.

**Critérios de Aceite:**
- [ ] A verificação de assinatura usa comparação timing-safe (`crypto.timingSafeEqual`) — não `===`.
- [ ] O corpo bruto é lido antes de qualquer análise JSON (Fastify analisa o corpo por padrão — configure para preservar os bytes brutos).
- [ ] A chave de idempotência por ID de evento é armazenada no Redis com TTL de 24 horas.
- [ ] A falha no envio de email não faz o webhook retornar resposta não-200.
- [ ] Cada tipo de evento é tratado por uma função separada em um módulo `webhookHandlers`.

**Dicas:**
1. No Fastify: registre `addContentTypeParser('application/json', { parseAs: 'buffer' }, ...)` para obter o corpo bruto para verificação de assinatura, depois analise manualmente.
2. Verificação de assinatura: `crypto.createHmac('sha256', secret).update(rawBody).digest('hex')`. Compare com `crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(received))`.
3. Idempotência: `SET webhook:<eventId> 1 NX EX 86400`. Se `SET` retornar `null`, o evento já foi processado.
4. Fire-and-forget no email: chame `sendEmail(...)` sem `await`. Capture e registre qualquer erro internamente.

---

## Exercício 5 — Otimização de Query no Banco de Dados (Médio)

**Cenário:** Um endpoint de relatório está levando 8–12 segundos para responder. Seu trabalho é diagnosticar e corrigir.

**Query inicial (PostgreSQL):**
```sql
SELECT
  u.name,
  u.email,
  COUNT(o.id) AS order_count,
  SUM(o.total_amount) AS lifetime_value,
  MAX(o.created_at) AS last_order_date
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.created_at > NOW() - INTERVAL '1 year'
  AND o.status = 'completed'
GROUP BY u.id
ORDER BY lifetime_value DESC
LIMIT 100;
```

**Requisitos:**
- Identifique por que essa query é lenta (índices ausentes, ordem de join ruim, posicionamento de filtro, etc.).
- Reescreva ou otimize para que rode em menos de 200ms em uma tabela com 1M de usuários e 5M de pedidos.
- Proponha os índices necessários.
- Envolva o resultado em uma materialized view ou camada de cache se necessário.

**Critérios de Aceite:**
- [ ] `EXPLAIN ANALYZE` mostra Seq Scans substituídos por Index Scans/Index Only Scans.
- [ ] O filtro `WHERE o.status = 'completed'` é aplicado antes do JOIN (empurre o predicado para baixo ou use um partial index).
- [ ] No mínimo, proponha índices em: `users(created_at)`, `orders(user_id, status)`, e explique os trade-offs.
- [ ] Se você introduzir uma materialized view, explique a estratégia de refresh (sob demanda? agendada?).
- [ ] A correção é testada em um dataset semeado com 1M+ linhas.

**Dicas:**
1. A query filtra em `o.status = 'completed'` mas faz o join primeiro — isso significa que todos os pedidos são carregados antes da filtragem. Um CTE ou subquery que filtra pedidos primeiro pode ajudar.
2. `LEFT JOIN` com cláusula `WHERE` na tabela direita torna-se implicitamente um `INNER JOIN` — torne isso explícito para clareza e dicas ao otimizador.
3. Um partial index `CREATE INDEX ON orders (user_id) WHERE status = 'completed'` reduz drasticamente o tamanho do índice e acelera o join.
4. Para dashboards acessados frequentemente: uma materialized view atualizada a cada 15 minutos costuma ser a resposta pragmática. `REFRESH MATERIALIZED VIEW CONCURRENTLY` evita bloqueios.

---

## Exercício 6 — Camada de Cache para uma API Cara (Médio)

**Cenário:** Seu app busca taxas de câmbio de uma API externa. A API externa tem um rate limit de 100 chamadas/hora. Seu app serve 10.000 requisições/minuto que precisam das taxas mais recentes.

**Requisitos:**
- Implemente uma camada de cache usando Redis que serve as taxas em cache para todos os clientes.
- As taxas são armazenadas em cache por 5 minutos (a API externa atualiza a cada 5 minutos).
- Em cache miss: buscar da API externa, armazenar no Redis, retornar ao cliente.
- Em falha da API externa: servir as últimas taxas conhecidas (stale-while-error), se disponíveis. Se não houver dados em cache, retornar `503`.
- Expor um endpoint `GET /rates` e um endpoint admin `POST /rates/refresh` para forçar a atualização do cache.

**Critérios de Aceite:**
- [ ] Apenas uma requisição chega à API externa mesmo que 1.000 requisições concorrentes cheguem simultaneamente em um cache miss (prevenção de cache stampede).
- [ ] As taxas desatualizadas são servidas (com um header de aviso `X-Cache-Stale: true`) em falha da API externa.
- [ ] Formato da chave de cache: `rates:v1:current`. Mudar a forma dos dados incrementa a versão.
- [ ] `POST /rates/refresh` exige um header `Admin-Key` correspondendo a uma variável de ambiente.
- [ ] Um teste unitário faz mock do Redis e da API externa, verificando o comportamento de fallback com dados desatualizados.

**Dicas:**
1. Cache stampede: use um lock Redis (`SET lock:rates 1 NX EX 10`) antes de buscar. Se o lock estiver tomado, faça polling (com backoff) até o cache ser preenchido por quem detém o lock.
2. Fallback com dados stale: use uma segunda chave Redis `rates:v1:stale` com TTL de 1 hora. Sempre atualize junto com a chave primária. Em erro, leia da chave stale.
3. Resposta de `GET /rates`: inclua `Cache-Control: public, max-age=60` para que CDNs e navegadores também possam fazer cache.
4. A versão na chave de cache permite invalidar todos os caches em formato antigo sem flush manual — apenas incremente `v1` para `v2`.

---

## Exercício 7 — Fila de Jobs em Background (Médio)

**Cenário:** Seu app precisa enviar emails de boas-vindas, redimensionar imagens enviadas e gerar relatórios PDF de forma assíncrona. Isso não deve bloquear as respostas HTTP.

**Requisitos:**
- Implemente uma fila de jobs usando BullMQ (backed por Redis).
- Três tipos de job: `welcome-email`, `image-resize`, `pdf-report`.
- Cada tipo de job tem um worker dedicado com sua própria configuração de concorrência: email (5), image (2), pdf (1).
- Jobs com falha são reprocessados até 3 vezes com backoff exponencial (1s, 2s, 4s).
- Após 3 falhas, o job vai para uma dead-letter queue (estado "failed" do BullMQ).
- Um endpoint `GET /jobs/:id` retorna o status do job (waiting, active, completed, failed).

**Critérios de Aceite:**
- [ ] O handler HTTP enfileira o job e retorna `202 Accepted` com `{ jobId }` imediatamente.
- [ ] Os workers são registrados em um processo separado (ou arquivo worker) — não no processo do servidor Fastify.
- [ ] O backoff exponencial é configurado via opções `attempts` e `backoff` do BullMQ, não com `setTimeout` manual.
- [ ] `GET /jobs/:id` funciona para todos os tipos de job, independentemente de qual fila pertencem.
- [ ] Um teste simula falha de job e verifica o comportamento de retry sem precisar de Redis real (use `ioredis-mock`).

**Dicas:**
1. Setup BullMQ: uma `Queue` por tipo de job, um `Worker` por tipo de job. Use `{ connection }` de um client `ioredis` compartilhado.
2. Config de backoff: `{ attempts: 3, backoff: { type: 'exponential', delay: 1000 } }`.
3. `GET /jobs/:id`: use `Job.fromId(queue, jobId)` do BullMQ. Para buscar em todas as filas, tente cada fila em sequência até encontrar um match.
4. Mantenha o código dos workers em arquivos `src/workers/<type>.worker.ts` — cada arquivo é um registro standalone de `Worker`.

---

## Exercício 8 — API Multi-Tenant com Row-Level Security (Difícil)

**Cenário:** Construa uma API onde múltiplos tenants (organizações) compartilham as mesmas tabelas do banco, mas seus dados são estritamente isolados. Um usuário pertence a um tenant e nunca deve ver os dados de outro tenant.

**Requisitos:**
- Tabelas: `tenants`, `users` (com FK `tenantId`), `projects` (com FK `tenantId`).
- Cada requisição à API é autenticada com um JWT que contém `userId` e `tenantId`.
- Todas as queries fazem scope automaticamente para o tenant autenticado — nenhum método de service deve precisar passar `tenantId` manualmente.
- Implementar usando Row Level Security (RLS) do PostgreSQL.
- Usuários admin (role: `admin`) podem ver dados de todos os tenants; usuários comuns veem apenas seu tenant.

**Critérios de Aceite:**
- [ ] Query direta ao banco como role `app_user` com `SET app.tenant_id = X` retorna apenas linhas do tenant X.
- [ ] Um usuário do tenant A não pode recuperar um projeto do tenant B por nenhum endpoint da API, mesmo adivinhando o `id`.
- [ ] O `tenantId` do JWT é definido na conexão ao banco via `SET LOCAL app.tenant_id` antes de cada query.
- [ ] As políticas RLS são criadas em um arquivo de migration (não aplicadas manualmente).
- [ ] Teste de integração: dois tenants, dois usuários, verifique o isolamento entre tenants.

**Dicas:**
1. No hook `onRequest` do Fastify: após a verificação do JWT, execute `SET LOCAL app.tenant_id = '<tenantId>'` na conexão.
2. Política RLS: `CREATE POLICY tenant_isolation ON projects USING (tenant_id = current_setting('app.tenant_id')::uuid)`.
3. Habilite RLS: `ALTER TABLE projects ENABLE ROW LEVEL SECURITY; ALTER TABLE projects FORCE ROW LEVEL SECURITY;`
4. O role de banco `app_user` NÃO deve ser superuser — superusers ignoram o RLS por padrão.
5. Override de admin: crie uma `POLICY admin_override ... USING (current_setting('app.user_role') = 'admin')` separada.

---

## Exercício 9 — Notificações em Tempo Real com WebSockets (Médio)

**Cenário:** Construa um sistema de notificação onde usuários recebem alertas em tempo real quando o status do pedido muda.

**Requisitos:**
- Endpoint WebSocket: `ws://host/ws` — clientes conectam com JWT no query param `Authorization`.
- Quando o status de um pedido muda (via endpoint REST separado), todos os clientes WebSocket conectados como proprietário daquele pedido recebem uma mensagem `{ type: "order_update", orderId, status }`.
- Suportar escalonamento horizontal: se o servidor rodar em 3 instâncias, uma mudança de status na instância 1 deve notificar um cliente conectado à instância 2.
- Clientes que desconectam são removidos do registro de conexões.
- Heartbeat: envie um ping a cada 30 segundos; feche conexões que não respondam com pong em 10 segundos.

**Critérios de Aceite:**
- [ ] Clientes WebSocket são armazenados em um `Map<userId, WebSocket>` em memória.
- [ ] Um canal Redis Pub/Sub (`order:updates`) faz a ponte entre instâncias.
- [ ] Ao atualizar pedido: publicar no Redis; o subscriber de cada instância verifica seu `Map` local e envia aos clientes conectados.
- [ ] Clientes desconectados são removidos do map no handler do evento `'close'`.
- [ ] Teste de carga: 500 conexões WebSocket concorrentes sem memory leaks após 5 minutos.

**Dicas:**
1. Use o pacote `ws` com Fastify (`@fastify/websocket`). Registre o plugin WebSocket e adicione a opção `ws: true` na rota.
2. JWT no query param: extraia de `req.query.token`, verifique com `jwt.verify`. Rejeite com close code `4001` se inválido.
3. Redis Pub/Sub: use duas conexões `ioredis` separadas — uma para `subscribe`, outra para todos os outros comandos (uma conexão com subscribe não pode emitir outros comandos).
4. Heartbeat: use `ws.ping()` em um intervalo. No evento `pong`, resete um flag `isAlive`. Antes de cada ping, verifique `isAlive` — se false, termine a conexão.

---

## Exercício 10 — Locking Distribuído para Operações Idempotentes (Difícil)

**Cenário:** Seu serviço de checkout processa pagamentos. Se um usuário clica duas vezes em "Pagar", duas requisições chegam simultaneamente. A segunda requisição não deve cobrar o cartão duas vezes.

**Requisitos:**
- Implemente um lock distribuído usando Redis (`SET NX EX`) em volta da lógica de processamento de pagamento.
- Chave do lock: `lock:payment:<orderId>`. TTL: 30 segundos (duração máxima esperada do pagamento).
- Se o lock não puder ser adquirido, retornar `409 Conflict` com `{ message: "Pagamento já em andamento" }`.
- Após o pagamento completar (sucesso ou falha), o lock é liberado imediatamente (sem esperar o TTL expirar).
- A liberação do lock deve ser atômica (use um script Lua para evitar liberar o lock de outro processo).

**Critérios de Aceite:**
- [ ] Duas requisições simultâneas para o mesmo `orderId` resultam em exatamente uma tentativa de pagamento bem-sucedida.
- [ ] O lock é liberado em sucesso, falha e exceção inesperada (use try/finally).
- [ ] O script Lua para liberação: apenas delete a chave se seu valor corresponder ao token do lock (UUID definido no momento da aquisição).
- [ ] Se o pagamento levar mais de 30 segundos (caso extremo), trate graciosamente — registre um aviso, não quebre.
- [ ] Teste de integração usando Redis real verifica o cenário de condição de corrida.

**Dicas:**
1. Adquirir: `SET lock:payment:<orderId> <uuid> NX EX 30`. Retorna `"OK"` em sucesso, `null` se já detido.
2. Script Lua para liberação:
   ```lua
   if redis.call("GET", KEYS[1]) == ARGV[1] then
     return redis.call("DEL", KEYS[1])
   else
     return 0
   end
   ```
   Chame com `redis.eval(script, 1, lockKey, lockToken)`.
3. O UUID (token do lock) garante que você libera apenas seu próprio lock, não um readquirido por outro processo após seu TTL expirar.
4. `try { await processPayment() } finally { await releaseLock() }` — nunca pule a liberação.

---

## Exercício 11 — Estratégia de Versionamento de API (Fácil)

**Cenário:** Sua API pública está em uso por 500 clientes externos. Você precisa introduzir mudanças breaking em três endpoints sem forçar todos os clientes a migrarem imediatamente.

**Requisitos:**
- Suportar duas versões simultaneamente: `v1` (atual) e `v2` (nova).
- Versionamento via prefixo de path na URL: `/api/v1/...` e `/api/v2/...`.
- Os endpoints alterados na v2: o shape da resposta de `GET /users` muda (`firstName`/`lastName` divididos em um objeto `name`), e `POST /orders` agora exige um campo `currencyCode`.
- Endpoints da v1 continuam funcionando sem alteração por pelo menos 6 meses.
- Adicionar um header de resposta `Deprecation` em todos os endpoints da v1.
- Documente o guia de migração em um comentário JSDoc no nível da rota.

**Critérios de Aceite:**
- [ ] `GET /api/v1/users` ainda retorna `{ name: "John Doe" }`.
- [ ] `GET /api/v2/users` retorna `{ name: { first: "John", last: "Doe" } }`.
- [ ] `POST /api/v1/orders` aceita um corpo sem `currencyCode`.
- [ ] `POST /api/v2/orders` retorna `422` se `currencyCode` estiver ausente.
- [ ] Respostas da v1 incluem os headers `Deprecation: true` e `Sunset: <data 6 meses a partir de agora>`.

**Dicas:**
1. Fastify: registre dois prefixos de router com `fastify.register(v1Routes, { prefix: '/api/v1' })` e `fastify.register(v2Routes, { prefix: '/api/v2' })`.
2. Compartilhe a camada de service — as rotas v1 e v2 chamam os mesmos métodos de service; a diferença está no parsing da requisição e na transformação da resposta.
3. Headers de deprecação: adicione via hook de plugin que roda em todas as rotas v1: `reply.header('Deprecation', 'true').header('Sunset', sunsetDate)`.
4. Data de sunset: `new Date(Date.now() + 180 * 24 * 60 * 60 * 1000).toUTCString()`.

---

## Exercício 12 — Logging Estruturado e Observabilidade (Fácil)

**Cenário:** Um microsserviço em produção não tem logs úteis. Requisições falham silenciosamente e depurar leva horas. Adicione logging estruturado, rastreamento de requisição e monitoramento de erros.

**Requisitos:**
- Use Pino como logger (está embutido no Fastify).
- Cada requisição registra: `{ method, url, statusCode, responseTime, requestId }`.
- Cada erro registra: `{ error: { message, stack, name }, requestId, userId (se autenticado) }`.
- Atribua um `X-Request-ID` único a cada requisição (use o header se fornecido por um gateway, caso contrário gere com `crypto.randomUUID()`).
- Adicione um endpoint de health check `GET /health` que retorna `{ status: "ok", uptime, version }` — este endpoint NÃO deve ser logado (muito ruidoso).
- Níveis de log: `info` para requisições, `warn` para 4xx, `error` para 5xx.

**Critérios de Aceite:**
- [ ] Cada linha de log é JSON válido (saída padrão do Pino).
- [ ] O `requestId` é o mesmo em todas as linhas de log de uma determinada requisição HTTP.
- [ ] `GET /health` não produz nenhuma saída de log.
- [ ] Um log de erro 500 inclui o stack trace completo.
- [ ] O nível de log pode ser alterado em tempo de execução via variável de ambiente `LOG_LEVEL` sem reiniciar.

**Dicas:**
1. Logger Pino embutido no Fastify: `fastify({ logger: { level: process.env.LOG_LEVEL ?? 'info' } })`.
2. Request ID: configure `fastify({ genReqId: () => crypto.randomUUID() })` e use `req.id` em todo lugar.
3. Suprimir logs do health check: no `fastify.addHook('onSend', ...)`, verifique `req.url === '/health'` e pule o log. Ou configure `disableRequestLogging: true` nessa rota específica.
4. Nível de log por status: no hook `onResponse`, use `req.log.warn(...)` para 4xx e `req.log.error(...)` para 5xx, baseado em `reply.statusCode`.
