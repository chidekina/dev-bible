# Design de REST API

## 1. O Que é e Por Que Usar

REST (Representational State Transfer) é um estilo arquitetural para sistemas hipermídia distribuídos, definido por Roy Fielding em sua dissertação de 2000. Na prática, "REST API" refere-se a uma API HTTP que segue convenções que a tornam previsível, cacheável e fácil de consumir por qualquer cliente.

Um bom design de REST API funciona como multiplicador: uma API bem projetada é consumida corretamente na primeira vez, exige documentação mínima, falha de forma clara e informativa, e evolui sem quebrar os clientes existentes. Uma API mal projetada gera tickets de suporte, integrações incorretas e migrações de versionamento dolorosas.

O objetivo deste arquivo não é apenas regras de sintaxe, mas o raciocínio por trás de cada decisão — para que você possa aplicar julgamento quando as regras conflitam.

---

## 2. Conceitos Fundamentais

### Recursos, Não Ações

REST modela a API como uma coleção de **recursos** (substantivos), não uma coleção de operações (verbos).

```
# Errado — estilo RPC, orientado a ações
GET  /getUser?id=42
POST /createUser
POST /deleteUser?id=42
POST /activateUserAccount

# Correto — orientado a recursos
GET    /users/42
POST   /users
DELETE /users/42
POST   /users/42/activations  (sub-recurso representando o estado de ativação)
```

**Convenções de nomenclatura:**
- Use **substantivos no plural**: `/users`, `/orders`, `/products`
- Use **kebab-case** para recursos com múltiplas palavras: `/product-categories`
- Evite verbos nos paths; o método HTTP já expressa a ação
- Recursos aninhados expressam relacionamentos: `/users/42/orders` = pedidos do usuário 42
- Mantenha o aninhamento em **no máximo 2 níveis** — mais fundo acopla o cliente à estrutura interna

```
# Aceitável
GET /users/42/orders
GET /orders/99/items

# Fundo demais — evite
GET /users/42/orders/99/items/7/reviews
# Melhor: GET /items/7/reviews ou GET /reviews?itemId=7
```

### Métodos HTTP e Suas Semânticas

| Método | Seguro | Idempotente | Uso Comum |
|--------|--------|-------------|-----------|
| GET | Sim | Sim | Recuperar recurso(s) |
| HEAD | Sim | Sim | Igual ao GET mas sem body (verificar existência, ETags) |
| POST | Não | Não | Criar recurso, disparar ação |
| PUT | Não | Sim | Substituir recurso por completo |
| PATCH | Não | Não* | Atualização parcial |
| DELETE | Não | Sim | Remover recurso |

*PATCH não é inerentemente idempotente (ex.: incrementar um contador em 1), mas muitas implementações o tornam assim.

**Seguro** significa que a requisição não altera o estado do servidor — clientes e caches podem repeti-la livremente.
**Idempotente** significa que repetir a requisição N vezes produz o mesmo resultado que fazê-la uma vez — seguro para retentativas.

**PUT vs PATCH:**

```json
// PUT /users/42 — substitui o recurso inteiro
// Campos omitidos são definidos como null/padrão
{
  "name": "Alice",
  "email": "alice@example.com",
  "role": "admin"
}

// PATCH /users/42 — atualização parcial; apenas os campos fornecidos mudam
{
  "email": "newalice@example.com"
}
// name e role permanecem inalterados
```

> Use PUT quando o cliente constrói a representação completa do recurso. Use PATCH para atualizações em nível de campo. PATCH é mais comum em APIs reais porque raramente o cliente quer enviar todos os campos.

---

## 3. Como Funciona

### Status Codes: O Vocabulário da API

Status codes corretos são a primeira coisa que os clientes verificam. Usar 200 para erros é um dos anti-padrões mais prejudiciais de API.

```
2xx — Sucesso
  200 OK              Sucesso geral, body contém o resultado
  201 Created         Recurso criado; inclua o header Location apontando para ele
  204 No Content      Sucesso, sem body (DELETE, algumas respostas de PATCH)

3xx — Redirecionamento
  301 Moved Permanently  Recurso tem uma nova URL permanente
  304 Not Modified       ETag/Last-Modified coincide; use a versão em cache

4xx — Erros do Cliente
  400 Bad Request         Sintaxe malformada, campos obrigatórios ausentes
  401 Unauthorized        Autenticação necessária ou falhou (nome enganoso — significa "não autenticado")
  403 Forbidden           Autenticado mas não autorizado para este recurso
  404 Not Found           Recurso não existe
  405 Method Not Allowed  Método HTTP não suportado para este endpoint
  409 Conflict            Conflito de estado (ex.: email duplicado, conflito de edição)
  410 Gone                Recurso existia mas foi permanentemente deletado
  422 Unprocessable Entity Sintaticamente válido mas semanticamente inválido (erros de validação)
  429 Too Many Requests   Limite de requisições excedido

5xx — Erros do Servidor
  500 Internal Server Error Falha inesperada do servidor
  502 Bad Gateway           Serviço upstream retornou resposta inválida
  503 Service Unavailable   Sobrecarregado ou em manutenção
  504 Gateway Timeout       Serviço upstream atingiu timeout
```

> Nunca retorne 200 com `{ "status": "error" }` no body. Status codes na camada HTTP são verificáveis sem parsear o body. Muitos clientes HTTP, proxies e ferramentas de monitoramento dependem deles.

### Formato de Resposta de Erro: RFC 7807 Problem Details

A RFC 7807 define um formato JSON padrão para erros. Adotá-lo torna sua API interoperável com qualquer cliente que entenda o padrão.

```json
{
  "type": "https://api.example.com/errors/validation-error",
  "title": "Validation Error",
  "status": 422,
  "detail": "The request body contains invalid fields.",
  "instance": "/api/users",
  "errors": [
    { "field": "email", "message": "Must be a valid email address" },
    { "field": "age", "message": "Must be a positive integer" }
  ]
}
```

Campos:
- `type` — URI que identifica o tipo de erro (dereferenciável = documentação legível por máquina)
- `title` — resumo legível por humanos (o mesmo para todas as instâncias deste tipo de erro)
- `status` — código de status HTTP (redundante mas útil para logging)
- `detail` — explicação legível por humanos para esta ocorrência específica
- `instance` — URI da requisição específica que causou o erro

### Estratégias de Versionamento de API

```
Estratégia 1: Versionamento no path da URL (mais comum)
  /v1/users
  /v2/users

  Prós: Explícito, fácil de rotear, fácil de testar no browser
  Contras: Tecnicamente não é RESTful (versão não faz parte da identidade do recurso)

Estratégia 2: Versionamento via header Accept
  GET /users
  Accept: application/vnd.myapi.v2+json

  Prós: URLs mais limpas, semanticamente correto
  Contras: Mais difícil de testar no browser/curl, requer configuração cuidadosa de cache

Estratégia 3: Query parameter
  GET /users?version=2

  Prós: Simples de implementar
  Contras: Fácil de omitir, polui query strings, problemas de cache em proxies

Estratégia 4: Header customizado
  GET /users
  API-Version: 2024-01-01

  Prós: Versionamento por data é claro sobre quando as mudanças aconteceram
  Contras: Não padrão, exige documentação
```

**Recomendação:** Versionamento no path da URL (`/v1/`) para APIs públicas (previsível, fácil de documentar). Versionamento por header para APIs internas onde a limpeza da URL importa. Headers baseados em data (estilo Stripe) para APIs com mudanças incrementais que quebram compatibilidade.

### Paginação

**Paginação por offset/limit:**

```
GET /posts?offset=0&limit=20   → primeira página
GET /posts?offset=20&limit=20  → segunda página
GET /posts?offset=40&limit=20  → terceira página
```

Problemas:
- Resultados inconsistentes quando itens são inseridos/deletados entre páginas (linhas se deslocam)
- Performance degrada em offsets grandes (ex.: `OFFSET 100000` no PostgreSQL escaneia 100.000 linhas antes de retornar 20)

**Paginação baseada em cursor (preferida para grandes datasets):**

```
GET /posts?limit=20
→ retorna itens + nextCursor

GET /posts?limit=20&cursor=eyJpZCI6IjEwMCJ9
→ retorna próxima página
```

O cursor codifica uma posição estável (tipicamente o ID ou timestamp do último item), não um offset numérico.

**Envelope de resposta:**

```json
{
  "data": [...],
  "meta": {
    "nextCursor": "eyJpZCI6IjEwMCJ9",
    "hasMore": true,
    "totalCount": 4200
  }
}
```

> Evite retornar `totalCount` quando isso exige um scan completo da tabela. Omita, compute separadamente sob demanda ou use contagens aproximadas (`SELECT reltuples FROM pg_class`).

### Filtros, Ordenação, Sparse Fieldsets

```
# Filtros
GET /users?status=active&role=admin&createdAfter=2024-01-01

# Ordenação (prefixo - para descendente)
GET /posts?sort=-createdAt,title

# Sparse fieldsets (GraphQL-lite para REST)
GET /users?fields=id,name,email

# Combinado
GET /orders?status=pending&sort=-createdAt&limit=10&fields=id,total,customerId
```

---

## 4. Exemplos de Código (TypeScript / Fastify)

### Validação de Requisição com Zod

```typescript
import Fastify from 'fastify';
import { z } from 'zod';

const app = Fastify({ logger: true });

// Schemas
const CreateUserBody = z.object({
  name: z.string().min(1).max(100),
  email: z.string().email(),
  role: z.enum(['user', 'admin']).default('user'),
  age: z.number().int().positive().optional(),
});

const UserParams = z.object({
  id: z.string().uuid(),
});

const ListUsersQuery = z.object({
  status: z.enum(['active', 'inactive']).optional(),
  limit: z.coerce.number().int().min(1).max(100).default(20),
  cursor: z.string().optional(),
  sort: z.string().optional(),
});

type CreateUserBody = z.infer<typeof CreateUserBody>;
```

### GET com Paginação por Cursor

```typescript
app.get('/v1/users', async (req, reply) => {
  const query = ListUsersQuery.safeParse(req.query);
  if (!query.success) {
    return reply.status(422).send({
      type: 'https://api.example.com/errors/validation',
      title: 'Validation Error',
      status: 422,
      detail: 'Invalid query parameters',
      errors: query.error.errors.map((e) => ({
        field: e.path.join('.'),
        message: e.message,
      })),
    });
  }

  const { limit, cursor, status, sort } = query.data;

  // Decodifica o cursor (JSON em base64url com o último id visto)
  let cursorId: string | undefined;
  if (cursor) {
    try {
      const decoded = JSON.parse(Buffer.from(cursor, 'base64url').toString('utf8'));
      cursorId = decoded.id;
    } catch {
      return reply.status(400).send({
        type: 'https://api.example.com/errors/invalid-cursor',
        title: 'Invalid Cursor',
        status: 400,
        detail: 'The provided cursor is malformed or expired.',
      });
    }
  }

  const users = await userService.list({ limit: limit + 1, cursorId, status, sort });

  const hasMore = users.length > limit;
  const items = hasMore ? users.slice(0, limit) : users;

  const nextCursor = hasMore
    ? Buffer.from(JSON.stringify({ id: items[items.length - 1].id })).toString('base64url')
    : null;

  return reply.status(200).send({
    data: items,
    meta: {
      nextCursor,
      hasMore,
      count: items.length,
    },
  });
});
```

### POST com 201 + Header Location

```typescript
app.post('/v1/users', async (req, reply) => {
  const body = CreateUserBody.safeParse(req.body);
  if (!body.success) {
    return reply.status(422).send({
      type: 'https://api.example.com/errors/validation',
      title: 'Validation Error',
      status: 422,
      detail: 'The request body contains invalid fields.',
      errors: body.error.errors.map((e) => ({
        field: e.path.join('.'),
        message: e.message,
      })),
    });
  }

  try {
    const user = await userService.create(body.data);

    return reply
      .status(201)
      .header('Location', `/v1/users/${user.id}`)
      .send({ data: user });
  } catch (err) {
    if (err instanceof DuplicateEmailError) {
      return reply.status(409).send({
        type: 'https://api.example.com/errors/conflict',
        title: 'Conflict',
        status: 409,
        detail: `A user with email ${body.data.email} already exists.`,
      });
    }
    throw err; // Deixa o handler de erros do Fastify tratar erros inesperados
  }
});
```

### PATCH Atualização Parcial

```typescript
const UpdateUserBody = z.object({
  name: z.string().min(1).max(100).optional(),
  email: z.string().email().optional(),
  role: z.enum(['user', 'admin']).optional(),
}).refine(
  (data) => Object.keys(data).length > 0,
  { message: 'At least one field must be provided' }
);

app.patch('/v1/users/:id', async (req, reply) => {
  const params = UserParams.safeParse(req.params);
  if (!params.success) {
    return reply.status(400).send({
      type: 'https://api.example.com/errors/validation',
      title: 'Invalid ID',
      status: 400,
      detail: 'User ID must be a valid UUID.',
    });
  }

  const body = UpdateUserBody.safeParse(req.body);
  if (!body.success) {
    return reply.status(422).send({ /* RFC 7807 */ });
  }

  const user = await userService.update(params.data.id, body.data);
  if (!user) {
    return reply.status(404).send({
      type: 'https://api.example.com/errors/not-found',
      title: 'Not Found',
      status: 404,
      detail: `User ${params.data.id} does not exist.`,
    });
  }

  return reply.status(200).send({ data: user });
});
```

### DELETE com 204 No Content

```typescript
app.delete('/v1/users/:id', async (req, reply) => {
  const params = UserParams.safeParse(req.params);
  if (!params.success) {
    return reply.status(400).send({ /* erro de validação */ });
  }

  const deleted = await userService.delete(params.data.id);
  if (!deleted) {
    return reply.status(404).send({ /* não encontrado */ });
  }

  return reply.status(204).send(); // Sem body no sucesso
});
```

### Chaves de Idempotência para POST

```typescript
// Crítico para APIs de pagamento — POST /payments com chave de idempotência
// Mesma chave = mesmo resultado; seguro para retentativas em caso de falha de rede

app.post('/v1/payments', async (req, reply) => {
  const idempotencyKey = req.headers['idempotency-key'];
  if (!idempotencyKey || typeof idempotencyKey !== 'string') {
    return reply.status(400).send({
      type: 'https://api.example.com/errors/missing-idempotency-key',
      title: 'Missing Idempotency Key',
      status: 400,
      detail: 'The Idempotency-Key header is required for payment requests.',
    });
  }

  // Verifica se já processamos esta chave
  const cached = await redis.get(`idempotency:${idempotencyKey}`);
  if (cached) {
    return reply
      .status(200)
      .header('Idempotency-Replayed', 'true')
      .send(JSON.parse(cached));
  }

  const result = await paymentService.charge(req.body);

  // Armazena o resultado com TTL de 24h
  await redis.setex(
    `idempotency:${idempotencyKey}`,
    86400,
    JSON.stringify(result)
  );

  return reply.status(201).send(result);
});
```

### Exemplo de HATEOAS

```json
// Resposta de GET /v1/orders/99
{
  "data": {
    "id": "99",
    "status": "pending",
    "total": 149.99,
    "_links": {
      "self": { "href": "/v1/orders/99" },
      "cancel": { "href": "/v1/orders/99/cancellations", "method": "POST" },
      "pay": { "href": "/v1/payments", "method": "POST" },
      "customer": { "href": "/v1/users/42" }
    }
  }
}
```

> HATEOAS permite que clientes naveguem na API sem hardcodear URLs — eles seguem links de resposta em resposta. Raramente é implementado por completo na prática, mas o conceito é valioso: inclua URLs de ações relevantes nas respostas para que os clientes não precisem de conhecimento externo da estrutura de URLs.

---

## 5. Erros Comuns e Armadilhas

**Usar GET para operações com efeitos colaterais:**

```
# Errado — GET deve ser seguro
GET /api/users/42/deactivate
GET /api/emails/send?to=user@example.com

# Correto
POST /api/users/42/deactivations
POST /api/emails
```

**Status codes errados:**

```typescript
// Errado — 200 para not-found esconde o erro do monitoramento
res.status(200).json({ error: 'User not found' });

// Errado — 500 para input inválido do cliente
res.status(500).json({ error: 'Email already taken' }); // deveria ser 409

// Correto
res.status(404).json({ type: '...', title: 'Not Found', status: 404 });
res.status(409).json({ type: '...', title: 'Conflict', status: 409 });
```

**Vazar implementação interna em mensagens de erro:**

```json
// Errado — expõe o schema do banco e erros internos
{
  "error": "ERROR:  duplicate key value violates unique constraint \"users_email_key\""
}

// Correto — abstraia para uma mensagem em nível de domínio
{
  "type": "https://api.example.com/errors/conflict",
  "title": "Conflict",
  "status": 409,
  "detail": "An account with this email address already exists."
}
```

**Aninhamento profundo de recursos:**

```
# Evite — acopla o cliente à estrutura interna
GET /companies/5/departments/12/employees/88/projects/3

# Melhor — use query parameters para filtros
GET /projects?employeeId=88
GET /employees/88/projects
```

**Pluralização inconsistente:**

```
# Errado — misto de singular/plural
GET /user        vs GET /orders
GET /product/5   vs GET /carts/5/items

# Correto — sempre no plural
GET /users
GET /orders
GET /products/5
GET /carts/5/items
```

---

## 6. Quando Usar / Não Usar

**REST é uma boa escolha quando:**
- Construindo uma API pública com consumidores diversos (mobile, web, terceiros)
- Operações simples de CRUD dominam o caso de uso
- Cache HTTP é importante (CDN, cache do browser)
- A superfície da API é estável e evolui lentamente
- Consumidores precisam navegar na API sem documentação de SDK

**Considere alternativas quando:**
- Múltiplos clientes precisam de formatos de dados muito diferentes (→ GraphQL)
- Comunicação bidirecional de alta frequência (→ WebSockets/gRPC)
- Comunicação interna entre serviços com schemas rígidos (→ gRPC)
- Performance com protocolo binário é necessária (→ gRPC, MessagePack)
- Streaming de eventos em tempo real (→ Server-Sent Events, Kafka + gateway HTTP)

---

## 7. Cenário Real

**Problema:** Uma API de e-commerce tem um endpoint POST `/api/order/checkout` que cobra o cartão do cliente. Quando o app mobile sofre um timeout de rede, ele tenta novamente — resultando em cobranças duplicadas.

**Causa raiz:** O endpoint não é idempotente. Na retentativa, uma nova cobrança é criada.

**Solução: Chaves de idempotência com deduplicação via Redis:**

```typescript
// Cliente envia: POST /v1/checkouts
// Header: Idempotency-Key: <uuid gerado por tentativa de checkout>

// Servidor verifica: já processamos esta chave?
app.post('/v1/checkouts', async (req, reply) => {
  const key = req.headers['idempotency-key'] as string;
  if (!key) return reply.status(400).send(/* erro de chave ausente */);

  // Verifica resultado existente
  const existing = await redis.get(`checkout:idempotency:${key}`);
  if (existing) {
    return reply
      .status(200)
      .header('Idempotency-Replayed', 'true')
      .send(JSON.parse(existing));
  }

  // Lock para evitar requisições concorrentes com a mesma chave
  const lock = await redis.set(
    `checkout:lock:${key}`,
    '1',
    'EX', 30,
    'NX' // define apenas se não existe
  );
  if (!lock) {
    return reply.status(409).send({
      type: 'https://api.example.com/errors/concurrent-request',
      title: 'Concurrent Request',
      status: 409,
      detail: 'A request with this idempotency key is already being processed.',
    });
  }

  try {
    const order = await checkoutService.process(req.body);
    const response = { data: order };

    // Armazena resultado por 24 horas
    await redis.setex(`checkout:idempotency:${key}`, 86400, JSON.stringify(response));

    return reply.status(201).send(response);
  } finally {
    await redis.del(`checkout:lock:${key}`);
  }
});
```

**Resultado:** Retentativas retornam o mesmo resultado sem criar cobranças adicionais. O pagamento do cliente está protegido.

---

## 8. Perguntas de Entrevista

**Q1: Quais são as seis restrições REST definidas por Fielding?**

(1) **Client-Server** — UI separada do armazenamento de dados; evolução independente. (2) **Stateless** — cada requisição contém todas as informações necessárias; o servidor não armazena contexto do cliente entre requisições. (3) **Cacheable** — respostas devem se definir como cacheáveis ou não. (4) **Uniform Interface** — identificação de recursos em requisições, manipulação através de representações, mensagens auto-descritivas, HATEOAS. (5) **Layered System** — o cliente não sabe se está conectado diretamente ao servidor ou através de intermediários. (6) **Code on Demand (opcional)** — servidores podem transferir código executável (JavaScript) para clientes.

**Q2: Qual é a diferença entre PUT e PATCH?**

PUT substitui o recurso inteiro — se você omitir um campo, ele é zerado. PATCH aplica uma atualização parcial — apenas os campos fornecidos mudam. PUT é idempotente (mesmo payload = mesmo resultado); PATCH não é inerentemente idempotente (ex.: PATCH para incrementar um contador). Use PUT quando o cliente tem o recurso completo; use PATCH para atualizações em nível de campo, o que é mais comum.

**Q3: Qual é a diferença entre paginação baseada em cursor e por offset?**

Paginação por offset (`?offset=20&limit=10`) pula N linhas no banco — simples mas inconsistente quando linhas são inseridas/deletadas entre páginas, e lenta em offsets grandes (o banco precisa escanear e descartar linhas). Paginação por cursor usa um ponteiro estável para o último item visto (tipicamente seu ID ou timestamp) — consistente independente de inserções concorrentes, e eficiente porque usa uma cláusula WHERE indexada ao invés de OFFSET. Use paginação por cursor para grandes datasets ou dados em tempo real; por offset para listas de admin pequenas onde simplicidade importa.

**Q4: Como você versiona uma REST API? Quais são os trade-offs?**

Path na URL (`/v1/`): explícito, fácil de rotear e testar, amplamente entendido, mas tecnicamente inclui a versão na identidade do recurso. Header (`Accept: application/vnd.api.v2+json`): semanticamente correto, URLs mais limpas, mas mais difícil de testar e requer configuração cuidadosa de cache. Query param (`?version=2`): mais simples de implementar, mas polui URLs e é fácil de omitir acidentalmente. Minha escolha padrão: versionamento no path da URL para APIs públicas porque é o mais previsível para os consumidores.

**Q5: O que é idempotência e por que ela importa em APIs?**

Idempotência significa que aplicar uma operação N vezes produz o mesmo resultado que aplicá-la uma vez. GET, PUT, DELETE são idempotentes por definição. POST não é — chamar POST /payments duas vezes cria dois pagamentos. Idempotência importa porque redes falham e clientes tentam novamente. Para operações POST não idempotentes, implemente chaves de idempotência: o cliente gera uma chave única por operação; o servidor armazena o resultado em cache pela chave, retornando o resultado em cache nas retentativas ao invés de executar novamente.

**Q6: Qual é a diferença entre 401 e 403?**

401 Unauthorized (nome enganoso) significa que a requisição não tem credenciais de autenticação válidas — o servidor não sabe quem você é. 403 Forbidden significa que o servidor sabe quem você é, mas você não tem permissão para acessar o recurso. Teste prático: 401 = "por favor faça login primeiro"; 403 = "você está logado mas não tem permissão aqui."

**Q7: O que é a RFC 7807 e por que você deveria usá-la?**

RFC 7807 "Problem Details for HTTP APIs" define um formato JSON padrão para erros com campos: `type` (URI identificando a categoria do erro), `title` (resumo legível por humanos), `status` (código de status HTTP), `detail` (explicação específica), `instance` (URI da ocorrência específica do erro). Usá-la torna os erros legíveis por máquina e interoperáveis — clientes que entendem RFC 7807 podem tratar erros de qualquer API compatível sem parsing customizado.

**Q8: O que é HATEOAS e é usado na prática?**

HATEOAS (Hypermedia as the Engine of Application State) significa que respostas de API incluem links para ações e recursos relacionados, permitindo que clientes naveguem na API sem URLs hardcodadas. Exemplo: uma resposta GET /orders/99 inclui `_links.cancel` e `_links.pay`. Em teoria, permite evolução da API sem mudanças no cliente. Na prática, poucas APIs de produção implementam HATEOAS completo porque aumenta o tamanho do payload e clientes raramente seguem links dinamicamente. O conceito ainda é valioso: inclua as URLs de ação mais relevantes nas respostas para reduzir o acoplamento do cliente à estrutura de URLs.

---

## 9. Exercícios

**Exercício 1: Projete uma REST API para um blog**

Projete a estrutura completa de URLs para um blog com: usuários, posts, comentários, tags e curtidas. Defina endpoints para: listar posts (com filtro por tag e autor), criar um post, atualizar um post, deletar um post, adicionar um comentário, obter comentários de um post, curtir um post. Especifique o método HTTP e os status codes esperados para cada um.

*Dica:* Considere: `/posts`, `/posts/:id`, `/posts/:id/comments`, `/posts/:id/likes`. Tags devem ser um sub-recurso ou um query param? Por quê?

**Exercício 2: Implemente paginação por cursor**

Dada uma tabela PostgreSQL `posts(id UUID, title TEXT, created_at TIMESTAMPTZ)`, implemente:
- Uma rota Fastify `GET /posts` com query params `?limit` e `?cursor`
- Cursor codifica `{ id, createdAt }` como base64url JSON
- SQL que usa `WHERE (created_at, id) < ($cursor_created_at, $cursor_id)` para ordenação estável
- Envelope de resposta com `data`, `meta.nextCursor`, `meta.hasMore`

*Dica:* A comparação de tuplas `(a, b) < (c, d)` no PostgreSQL é equivalente a `a < c OR (a = c AND b < d)` — útil quando timestamps podem ser iguais.

**Exercício 3: Implemente um handler de erros RFC 7807**

Construa um plugin de handler de erros para Fastify que:
- Captura erros de validação Zod e retorna 422 com detalhes de erro por campo
- Captura erros de domínio conhecidos (NotFoundError, ConflictError, ForbiddenError) e mapeia para status codes corretos
- Captura erros desconhecidos e retorna 500 sem vazar stack traces
- Sempre retorna formato RFC 7807

*Dica:* Use `fastify.setErrorHandler()`. Verifique `error instanceof ZodError` e use `error.errors` para detalhes de campo.

---

## 10. Leitura Complementar

- [Dissertação original de Fielding sobre REST (Capítulo 5)](https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)
- [RFC 7807 — Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc7807)
- [RFC 9457 — Problem Details Atualizado (2023)](https://www.rfc-editor.org/rfc/rfc9457)
- [HTTP semantics RFC 9110](https://www.rfc-editor.org/rfc/rfc9110)
- [Stripe API — excelente design REST real](https://stripe.com/docs/api)
- [GitHub REST API — versionamento + paginação](https://docs.github.com/en/rest)
- [Documentação do Fastify](https://fastify.dev/docs/latest/)
- [Documentação do Zod](https://zod.dev)
- Livro: *REST API Design Rulebook* por Mark Masse
- Livro: *Building Microservices* por Sam Newman (Capítulo sobre APIs)
