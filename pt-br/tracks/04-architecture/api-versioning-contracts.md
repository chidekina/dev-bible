# Versionamento de API e Contratos

## 1. O Que e Por Que

Uma API é um contrato entre um produtor (o serviço) e seus consumidores (clientes, apps mobile, terceiros, outros microsserviços). Uma vez publicada, alterações na API podem quebrar os consumidores que dependem dela — a menos que o contrato seja gerenciado com cuidado.

Versionamento de API e design de contratos respondem a três perguntas:
1. Como você sinaliza que uma versão existe (URL? header? query param)?
2. Quais mudanças são breaking vs. non-breaking?
3. Como você verifica que o contrato está sendo respeitado sem precisar fazer deploy dos dois lados simultaneamente?

> Uma API bem gerenciada permite que você publique mudanças a qualquer momento sem precisar coordenar com cada consumidor. Uma API mal gerenciada significa que toda mudança no servidor exige deploy sincronizado de todos os clientes — o equivalente distribuído de um big-bang release.

---

## 2. Conceitos Fundamentais

### Breaking vs. Non-Breaking Changes

Entender a fronteira entre mudanças breaking e non-breaking é a base do gerenciamento de API.

**Mudanças non-breaking (retrocompatíveis):**
- Adicionar um novo campo opcional na requisição
- Adicionar um novo campo na resposta
- Adicionar um novo endpoint
- Adicionar um novo valor de enum (se os clientes usam um caso "default/unknown")
- Relaxar validação (aceitar mais entradas)
- Adicionar um novo header opcional

**Breaking changes (exige nova versão):**
- Remover um campo da requisição ou resposta
- Renomear um campo
- Alterar o tipo de um campo (`string` → `number`)
- Alterar a semântica de um campo (mesmo nome, significado diferente)
- Tornar um campo opcional em obrigatório
- Remover ou renomear um endpoint
- Alterar o método HTTP de um endpoint
- Apertar validação (rejeitar entradas antes aceitas)
- Remover um valor de enum

```typescript
// Resposta V1 — contrato existente
interface UserResponseV1 {
  id: string;
  name: string;
  email: string;
}

// Compatível com V1: campo opcional adicionado, todos os campos existentes mantidos
interface UserResponseV1Compatible {
  id: string;
  name: string;
  email: string;
  avatarUrl?: string;   // novo campo opcional — seguro
  createdAt?: string;   // novo campo opcional — seguro
}

// Breaking: campo renomeado, campo removido
interface UserResponseV2Breaking {
  userId: string;       // BREAKING: renomeado de 'id'
  fullName: string;     // BREAKING: renomeado de 'name'
  // email removido — BREAKING
}
```

> Adicionar um novo campo obrigatório em uma requisição é sempre breaking — mesmo que o campo tenha um valor padrão no servidor. Clientes existentes que não enviam o campo terão erros de validação.

---

## 3. Estratégias de Versionamento

### Tabela Comparativa

| Estratégia | Exemplo | Prós | Contras |
|------------|---------|------|---------|
| URL path | `/api/v1/users` | Explícito, fácil de testar no browser | Prolifera rotas, mudança de URL é feia |
| Accept header | `Accept: application/vnd.api+json; version=1` | URLs limpas, padrão REST | Não cacheável por padrão, mais difícil de testar |
| Query param | `/api/users?version=1` | Fácil de adicionar, sem mudança de rota | Polui query strings, fácil de esquecer |
| Header customizado | `X-API-Version: 1` | URLs limpas | Não padrão, problemas de cache em CDN |

### Versionamento por URL Path (Mais Comum)

```typescript
// Fastify — versionamento por URL path
import Fastify from 'fastify';

const app = Fastify();

// Rotas V1
app.register(async (v1) => {
  v1.get('/users/:id', async (req) => {
    const user = await userService.findById(req.params.id);
    return {
      id: user.id,
      name: user.name,
      email: user.email,
    };
  });
}, { prefix: '/api/v1' });

// Rotas V2 — shape de resposta diferente
app.register(async (v2) => {
  v2.get('/users/:id', async (req) => {
    const user = await userService.findById(req.params.id);
    return {
      id: user.id,
      displayName: user.name,          // renomeado
      contact: { email: user.email },  // reestruturado
      metadata: { createdAt: user.createdAt.toISOString() },
    };
  });
}, { prefix: '/api/v2' });
```

### Versionamento por Accept Header

```typescript
// Fastify — versionamento por content negotiation
app.get('/api/users/:id', {
  constraints: {
    version: '1.0.0',
  },
}, async (req) => {
  // Handler V1
  return { id: req.params.id, name: 'Alice' };
});

app.get('/api/users/:id', {
  constraints: {
    version: '2.0.0',
  },
}, async (req) => {
  // Handler V2
  return { id: req.params.id, displayName: 'Alice' };
});

// Cliente envia: Accept-Version: 2.0.0
```

### Versionamento por Query Parameter

```typescript
// Abordagem simples por query param
app.get('/api/users/:id', async (req, reply) => {
  const version = req.query.version ?? '1';

  const user = await userService.findById(req.params.id);

  if (version === '2') {
    return { id: user.id, displayName: user.name };
  }

  // Padrão: V1
  return { id: user.id, name: user.name };
});
```

> Versionamento por URL path é o padrão da indústria para APIs públicas (Stripe, GitHub e Twilio usam esse modelo). Versionamento por Accept header é teoricamente mais RESTful, mas mais difícil para os consumidores descobrirem. Escolha URL path, a menos que tenha um motivo específico para não fazer isso.

---

## 4. Deprecação: Sunset e Deprecation Headers

Ao retirar uma versão, dê aos consumidores tempo para migrar. Os headers HTTP `Deprecation` e `Sunset` (RFC 8594 e draft RFC) comunicam isso de forma legível por máquina.

```typescript
// Hook Fastify para adicionar headers de deprecação nas rotas V1
app.addHook('onSend', async (req, reply) => {
  if (req.url.startsWith('/api/v1/')) {
    // Deprecation: data em que a rota foi depreciada
    reply.header('Deprecation', 'Sat, 01 Jan 2026 00:00:00 GMT');
    // Sunset: data em que a rota será removida
    reply.header('Sunset', 'Mon, 01 Jun 2026 00:00:00 GMT');
    // Link para o guia de migração
    reply.header('Link', '</docs/migration/v1-to-v2>; rel="deprecation"');
  }
});
```

**Diretrizes para o período de Sunset:**
- APIs internas: mínimo de 2 ciclos de sprint (4 semanas)
- APIs externas com poucos consumidores: 3 a 6 meses
- APIs públicas com muitos consumidores: 12 a 24 meses (Stripe dá 3+ anos)

**Processo de deprecação:**
1. Comunicar via e-mail, changelog e portal do desenvolvedor
2. Adicionar headers `Deprecation` e `Sunset` a todas as respostas depreciadas
3. Logar o uso de endpoints depreciados para identificar quem ainda os utiliza
4. Enviar e-mails de lembrete conforme a data de Sunset se aproxima
5. Remover o endpoint na data de Sunset

```typescript
// Registrar uso de endpoint depreciado para monitoramento
app.addHook('onSend', async (req, reply) => {
  if (req.url.startsWith('/api/v1/')) {
    logger.warn({
      event: 'deprecated_endpoint_accessed',
      endpoint: `${req.method} ${req.url}`,
      consumer: req.headers['x-consumer-id'] ?? 'unknown',
      sunsetDate: '2026-06-01',
    });
  }
});
```

---

## 5. OpenAPI 3.x: Definindo e Documentando o Contrato

OpenAPI (anteriormente Swagger) é o padrão para descrever APIs REST em formato legível por máquina (YAML/JSON). Ele alimenta documentação, geração de código de cliente e testes de contrato.

### Fastify + @fastify/swagger

```typescript
// src/main.ts
import Fastify from 'fastify';
import swagger from '@fastify/swagger';
import swaggerUi from '@fastify/swagger-ui';

const app = Fastify();

await app.register(swagger, {
  openapi: {
    info: {
      title: 'Order API',
      description: 'API para gerenciamento de pedidos',
      version: '2.0.0',
      contact: {
        name: 'API Support',
        email: 'api-support@example.com',
      },
    },
    servers: [
      { url: 'https://api.example.com/v2', description: 'Production' },
      { url: 'http://localhost:3000/api/v2', description: 'Development' },
    ],
    components: {
      securitySchemes: {
        bearerAuth: {
          type: 'http',
          scheme: 'bearer',
          bearerFormat: 'JWT',
        },
      },
    },
  },
});

await app.register(swaggerUi, {
  routePrefix: '/docs',
  uiConfig: { docExpansion: 'list' },
});
```

```typescript
// Definindo schemas inline com as rotas
const CreateOrderSchema = {
  $id: 'CreateOrderRequest',
  type: 'object',
  required: ['customerId', 'items'],
  properties: {
    customerId: { type: 'string', format: 'uuid', description: 'UUID do cliente' },
    currency: { type: 'string', enum: ['USD', 'EUR', 'BRL'], default: 'USD' },
    items: {
      type: 'array',
      minItems: 1,
      items: {
        type: 'object',
        required: ['productId', 'quantity', 'unitPrice'],
        properties: {
          productId: { type: 'string', format: 'uuid' },
          quantity: { type: 'integer', minimum: 1, maximum: 100 },
          unitPrice: { type: 'number', exclusiveMinimum: 0 },
        },
      },
    },
  },
};

const OrderResponseSchema = {
  $id: 'OrderResponse',
  type: 'object',
  properties: {
    orderId: { type: 'string', format: 'uuid' },
    status: { type: 'string', enum: ['pending', 'confirmed', 'shipped', 'delivered', 'cancelled'] },
    total: { type: 'number' },
    currency: { type: 'string' },
    createdAt: { type: 'string', format: 'date-time' },
  },
};

app.post('/api/v2/orders', {
  schema: {
    tags: ['Orders'],
    summary: 'Criar um novo pedido',
    description: 'Cria um novo pedido para o cliente especificado.',
    security: [{ bearerAuth: [] }],
    body: CreateOrderSchema,
    response: {
      201: OrderResponseSchema,
      400: {
        type: 'object',
        properties: {
          error: { type: 'string' },
          details: { type: 'array', items: { type: 'string' } },
        },
      },
      401: { type: 'object', properties: { error: { type: 'string' } } },
    },
  },
}, async (req, reply) => {
  const order = await orderService.create(req.body);
  reply.status(201).send(order);
});
```

---

## 6. Contract Testing com Pact

Contract testing verifica se um consumidor e um provedor concordam com um contrato de API — sem exigir que os dois estejam em execução simultaneamente. Pact é o padrão da indústria para testes de contrato orientados ao consumidor (consumer-driven contract testing).

**Como funciona:**
1. O consumidor escreve um contrato (o que ele espera do provedor)
2. O contrato é publicado no Pact Broker
3. O provedor executa a verificação: minha implementação atual satisfaz o contrato?
4. Ambos os lados podem fazer deploy independentemente, desde que o contrato seja satisfeito

```typescript
// Lado do consumidor — Order Frontend define o que espera da Order API
// tests/pact/order-api.consumer.test.ts
import { PactV3, MatchersV3 } from '@pact-foundation/pact';
import { OrderApiClient } from '../../src/clients/OrderApiClient';

const { like, eachLike, string, integer } = MatchersV3;

const provider = new PactV3({
  consumer: 'order-frontend',
  provider: 'order-api',
  dir: './pacts',
});

describe('Order API Consumer Contract', () => {
  describe('GET /api/v2/orders/:id', () => {
    it('retorna um pedido quando ele existe', async () => {
      await provider
        .given('order ORD-001 exists')
        .uponReceiving('a request to get order ORD-001')
        .withRequest({
          method: 'GET',
          path: '/api/v2/orders/ORD-001',
          headers: { Authorization: like('Bearer valid_token') },
        })
        .willRespondWith({
          status: 200,
          headers: { 'Content-Type': 'application/json' },
          body: {
            orderId: string('ORD-001'),
            status: string('confirmed'),
            total: like(149.99),
            currency: string('USD'),
            createdAt: string('2026-01-01T10:00:00.000Z'),
          },
        })
        .executeTest(async (mockServer) => {
          const client = new OrderApiClient(mockServer.url);
          const order = await client.getOrder('ORD-001');

          expect(order.orderId).toBe('ORD-001');
          expect(order.status).toBe('confirmed');
        });
    });

    it('retorna 404 quando o pedido não existe', async () => {
      await provider
        .given('order MISSING does not exist')
        .uponReceiving('a request for a non-existent order')
        .withRequest({
          method: 'GET',
          path: '/api/v2/orders/MISSING',
          headers: { Authorization: like('Bearer valid_token') },
        })
        .willRespondWith({
          status: 404,
          body: { error: string('Order not found') },
        })
        .executeTest(async (mockServer) => {
          const client = new OrderApiClient(mockServer.url);
          await expect(client.getOrder('MISSING')).rejects.toThrow('Order not found');
        });
    });
  });
});
```

```typescript
// Lado do provedor — Order API verifica se satisfaz o contrato
// tests/pact/order-api.provider.test.ts
import { PactV3 } from '@pact-foundation/pact';
import { app } from '../../src/app';

const verifier = new PactV3({
  provider: 'order-api',
  providerBaseUrl: 'http://localhost:3000',
  pactBrokerUrl: process.env.PACT_BROKER_URL,
  consumerVersionSelectors: [{ mainBranch: true }, { deployed: true }],
});

describe('Order API Provider Verification', () => {
  beforeAll(async () => {
    await app.listen({ port: 3000 });
  });

  afterAll(async () => {
    await app.close();
  });

  it('satisfaz todos os contratos dos consumidores', async () => {
    await verifier.verifyProvider({
      stateHandlers: {
        'order ORD-001 exists': async () => {
          await testDb.seed({ orders: [{ id: 'ORD-001', status: 'confirmed', total: 149.99 }] });
        },
        'order MISSING does not exist': async () => {
          await testDb.clean();
        },
      },
    });
  });
});
```

> Contract tests dão a confiança necessária para fazer deploy do consumidor e do provedor de forma independente. A ferramenta "can-i-deploy" do Pact Broker verifica se uma versão específica do consumidor/provedor satisfaz todos os contratos atuais antes do deploy.

---

## 7. Versionamento em GraphQL

GraphQL adota uma abordagem diferente para versionamento: o schema é projetado para evoluir sem números de versão. Em vez de criar uma `v2` da API, você estende o schema e deprecia campos antigos.

```graphql
type User {
  id: ID!
  name: String! @deprecated(reason: "Use 'displayName' instead, deprecated since 2026-01")
  displayName: String!
  email: String!
  # Novo campo adicionado — queries existentes não são afetadas
  avatarUrl: String
}

type Query {
  user(id: ID!): User
  # Query antiga depreciada
  getUser(id: ID!): User @deprecated(reason: "Use 'user(id)' instead")
}
```

```typescript
// Definição de schema TypeScript com deprecação
import { buildSchema } from 'graphql';

const schema = buildSchema(`
  type User {
    id: ID!
    name: String @deprecated(reason: "Use displayName")
    displayName: String!
    email: String!
  }

  type Query {
    user(id: ID!): User
  }
`);
```

**Princípios de versionamento em GraphQL:**
- Nunca quebre queries existentes — os clientes devem continuar funcionando sem mudanças
- Adicione novos campos livremente; deprecie os antigos com `@deprecated`
- Remova campos somente após um longo período de deprecação e quando o uso cair a zero
- Use persisted queries para rastrear quais clientes usam quais campos
- Schema stitching e federation permitem que múltiplos serviços componham um único grafo

**Monitorando uso de campos:**

```typescript
// Apollo Server — rastreando uso de campos depreciados
const server = new ApolloServer({
  schema,
  plugins: [
    {
      requestDidStart: async () => ({
        willSendResponse: async ({ document }) => {
          // Inspecionar a query em busca de campos depreciados
          // Registrar no sistema de monitoramento
        },
      }),
    },
  ],
});
```

> Sem monitorar quais clientes usam quais campos depreciados, você não pode removê-los com segurança. Sempre instrumente o uso dos campos antes de deprecar.

---

## 8. Erros Comuns e Armadilhas

| Erro | Consequência | Correção |
|------|-------------|----------|
| Sem versionamento | Breaking changes quebram silenciosamente todos os clientes | Adote versionamento por URL path desde o início |
| Versionar cada mudança menor | Proliferação de versões; consumidores precisam atualizar sempre | Versione apenas em breaking changes |
| Manter versões antigas para sempre | Carga de manutenção dobra a cada versão | Defina e aplique datas de Sunset |
| Sem headers Deprecation/Sunset | Consumidores não detectam deprecação automaticamente | Sempre adicione headers ao deprecar |
| Compartilhar o tipo de request/response entre versões | Mudanças de V2 vazam para V1 | Defina tipos separados por versão |
| Sem contract tests | Quebrar o consumidor só é descoberto no deploy | Adicione Pact ou testes de contrato por schema |
| Breaking changes em patch releases | "É só um pequeno fix" — mas quebrou o app mobile | Classifique breaking changes obrigatoriamente no PR review |

---

## 9. Quando Usar Cada Estratégia

| Situação | Abordagem recomendada |
|----------|----------------------|
| API pública com muitos consumidores externos | Versionamento por URL path + período de sunset longo (12+ meses) |
| API interna entre seus próprios serviços | Contract testing (Pact) + período de sunset curto |
| API GraphQL | Evolução de schema com `@deprecated`, sem números de versão |
| API REST onde os clientes são também seu próprio código | Contract tests; versione somente quando necessário |
| API gateway roteando múltiplos backends | Versionamento por URL path — fácil de rotear por versão |
| Backend para app mobile | Versionamento por URL path — você não pode forçar usuários a atualizar o app |

---

## 10. Perguntas de Entrevista

**Q1: Qual é a diferença entre uma breaking change e uma non-breaking change? Dê dois exemplos de cada.**

R: Uma non-breaking change não exige que os consumidores existentes mudem nada. Exemplos: adicionar um novo campo opcional na resposta, adicionar um novo endpoint. Uma breaking change força os consumidores existentes a atualizar ou eles vão falhar. Exemplos: renomear um campo, alterar o tipo de um campo de `string` para `number`, remover um endpoint, tornar um campo opcional em obrigatório.

---

**Q2: O que é consumer-driven contract testing e por que é valioso em uma arquitetura de microsserviços?**

R: Consumer-driven contract testing permite que o consumidor defina o que espera do provedor, e o provedor verifica isso de forma independente. É valioso porque substitui testes de integração end-to-end (que exigem ambos os serviços em execução) por testes isolados. Cada serviço pode ser testado e colocado em produção de forma independente, desde que a versão atual satisfaça todos os contratos dos consumidores registrados.

---

**Q3: Quais são os trade-offs entre versionamento por URL path e por Accept header?**

R: Versionamento por URL path (`/api/v2/`) é explícito, testável no browser, fácil de cachear e fácil de rotear no API gateway. A desvantagem é que as URLs mudam e pode parecer menos RESTful. Versionamento por Accept header (`Accept: application/vnd.api+json; version=2`) mantém URLs estáveis e é mais RESTful, mas é mais difícil de testar manualmente, não suportado por todos os clientes por padrão e mais difícil de cachear. Na prática, URL path é a escolha dominante para APIs públicas.

---

**Q4: Como o GraphQL trata a evolução de API de forma diferente do REST?**

R: GraphQL foi projetado para evoluir sem números de versão. Como os clientes especificam exatamente quais campos precisam em suas queries, adicionar novos campos nunca quebra os clientes existentes. Campos são depreciados com diretivas `@deprecated`, e a remoção acontece somente após zero uso confirmado. APIs REST com respostas JSON também podem evoluir sem quebrar, adicionando campos opcionais, mas o REST não possui a aplicação de schema e a seleção de campos pelo cliente que tornam a evolução do GraphQL mais natural.

---

**Q5: Qual deve ser o período de sunset para uma API pública?**

R: Depende da sua base de consumidores. Para desenvolvedores terceiros: mínimo de 12 meses, idealmente 18 a 24 meses (Stripe dá 3+ anos para versões maiores). Para APIs internas entre suas próprias equipes: 2 a 4 ciclos de sprint (4 a 8 semanas). Para APIs que suportam apps mobile: alinhe com o tempo que leva para seus usuários mais lentos atualizarem o app — isso pode ser 12+ meses se você não puder forçar atualizações.

---

**Q6: Como você lidaria com uma situação em que precisa fazer uma breaking change em uma API da qual centenas de clientes dependem?**

R: (1) Publique o novo comportamento em uma nova versão da API (`v2`). (2) Adicione headers `Deprecation` e `Sunset` a todas as respostas `v1`, dando mais de 12 meses de aviso. (3) Publique um guia de migração. (4) Monitore o uso de `v1` por consumidor para identificar quem ainda não migrou. (5) Contate diretamente os consumidores de alto tráfego. (6) Com 90 dias antes do sunset, envie lembretes por e-mail. (7) Remova `v1` na data de sunset. Nunca remova uma versão antes do prazo.

---

**Q7: O que é o padrão "Tolerant Reader" e como ele se relaciona com versionamento de API?**

R: O padrão Tolerant Reader (Martin Fowler) diz que os consumidores devem ser tolerantes com o que aceitam: ignorar campos desconhecidos, usar padrões para campos opcionais ausentes e não falhar em valores de enum não reconhecidos. Consumidores que implementam Tolerant Reader são mais resilientes a non-breaking changes e podem lidar com adições do lado do provedor sem exigir atualizações do consumidor. Isso é particularmente importante em sistemas event-driven, onde você não consegue versionar eventos com a mesma facilidade que respostas HTTP.

---

## 11. Exercícios

**Exercício 1: Classificar mudanças como breaking ou non-breaking**

Para cada item abaixo, classifique como breaking ou non-breaking e explique o motivo:

a) Adicionar `middleName?: string` a uma resposta de usuário
b) Alterar `price: number` para `price: { amount: number; currency: string }`
c) Remover o endpoint `GET /api/v1/users/:id/preferences`
d) Adicionar `GET /api/v1/users/:id/settings` (novo endpoint)
e) Alterar `status` de `'active' | 'inactive'` para `'active' | 'inactive' | 'suspended'`
f) Tornar `email` obrigatório em uma requisição que antes o tinha como opcional

---

**Exercício 2: Adicionar versionamento a um app Fastify existente**

Pegue um app Fastify com rotas em `/users`, `/orders` e `/products`. Adicione versionamento por URL path para que as rotas existentes se tornem `/api/v1/...`. Crie uma rota `/api/v2/users/:id` que retorna um shape de resposta diferente (renomeie `name` para `displayName`). Adicione headers de deprecação em todas as rotas v1 com um sunset de 6 meses.

---

**Exercício 3: Escrever um consumer test com Pact**

Escreva um consumer test com Pact para um `ProductApiClient` que chama `GET /api/v1/products/:id`. O consumidor espera: `{ id, name, price, inStock }`. Escreva o contrato do consumidor e depois escreva uma verificação de provedor stub que popula dados de teste.

---

**Exercício 4: Schema OpenAPI para uma Orders API**

Escreva um schema OpenAPI 3.x completo (YAML ou JSON) para os seguintes endpoints:
- `POST /api/v2/orders` — criar pedido
- `GET /api/v2/orders/:id` — buscar pedido
- `PATCH /api/v2/orders/:id/cancel` — cancelar pedido

Inclua schemas, campos obrigatórios, respostas de erro (400, 401, 404) e segurança (Bearer JWT).

---

## 12. Leitura Complementar

- **[Semantic Versioning](https://semver.org/)** — o padrão para numeração de versões
- **[RFC 8594 — Sunset Header](https://tools.ietf.org/html/rfc8594)** — RFC oficial para o header Sunset
- **[Pact Documentation](https://docs.pact.io/)** — framework de consumer-driven contract testing
- **[OpenAPI Specification](https://spec.openapis.org/oas/v3.1.0)** — spec oficial OpenAPI 3.1.0
- **[@fastify/swagger](https://github.com/fastify/fastify-swagger)** — integração OpenAPI para Fastify
- **[GraphQL Best Practices — Versioning](https://graphql.org/learn/best-practices/#versioning)** — orientação oficial do GraphQL
- **[API Design Patterns](https://www.manning.com/books/api-design-patterns)** — JJ Geewax (Manning, 2021)
- **[Stripe API Versioning](https://stripe.com/docs/api/versioning)** — como o Stripe gerencia 3+ anos de versões de API
