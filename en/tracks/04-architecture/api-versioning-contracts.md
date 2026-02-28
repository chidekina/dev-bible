# API Versioning & Contracts

## 1. What & Why

An API is a contract between a producer (the service) and its consumers (clients, mobile apps, third parties, other microservices). Once an API is published, changes to it can break consumers that depend on it — unless the contract is managed carefully.

API versioning and contract design answer three questions:
1. How do you signal that a version exists (URL? header? query param)?
2. What changes are breaking vs. non-breaking?
3. How do you verify that the contract is honored without deploying both sides simultaneously?

> 💡 A well-managed API lets you ship changes at any time without coordinating with every consumer. A poorly managed API means every server-side change requires synchronized deployment of all clients — the distributed system equivalent of a big-bang release.

---

## 2. Core Concepts

### Breaking vs. Non-Breaking Changes

Understanding the boundary between breaking and non-breaking changes is the foundation of API management.

**Non-breaking changes (backward compatible):**
- Adding a new optional request field
- Adding a new response field
- Adding a new endpoint
- Adding a new enum value (if clients use a "default/unknown" case)
- Relaxing validation (accepting more input)
- Adding a new optional header

**Breaking changes (requires new version):**
- Removing a field from a request or response
- Renaming a field
- Changing a field's type (`string` → `number`)
- Changing a field's semantics (same name, different meaning)
- Making an optional field required
- Removing or renaming an endpoint
- Changing the HTTP method of an endpoint
- Tightening validation (rejecting previously accepted input)
- Removing an enum value

```typescript
// V1 response — existing contract
interface UserResponseV1 {
  id: string;
  name: string;
  email: string;
}

// ✅ Non-breaking: added optional field, kept all existing fields
interface UserResponseV1Compatible {
  id: string;
  name: string;
  email: string;
  avatarUrl?: string;   // new optional field — safe
  createdAt?: string;   // new optional field — safe
}

// ❌ Breaking: renamed field, removed field
interface UserResponseV2Breaking {
  userId: string;       // BREAKING: renamed from 'id'
  fullName: string;     // BREAKING: renamed from 'name'
  // email removed — BREAKING
}
```

> ⚠️ Adding a new required field to a request is always breaking — even if the field has a default value on the server. Existing clients that don't send the field will trigger validation errors.

---

## 3. Versioning Strategies

### Comparison Table

| Strategy | Example | Pros | Cons |
|---------|---------|------|------|
| URL path | `/api/v1/users` | Explicit, easy to test in browser | Proliferates routes, URL changes are ugly |
| Accept header | `Accept: application/vnd.api+json; version=1` | Clean URLs, REST-standard | Not cacheable by default, harder to test |
| Query param | `/api/users?version=1` | Easy to add, no route changes | Pollutes query strings, easy to forget |
| Custom header | `X-API-Version: 1` | Clean URLs | Non-standard, CDN caching issues |

### URL Path Versioning (Most Common)

```typescript
// Fastify — URL path versioning
import Fastify from 'fastify';

const app = Fastify();

// V1 routes
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

// V2 routes — different response shape
app.register(async (v2) => {
  v2.get('/users/:id', async (req) => {
    const user = await userService.findById(req.params.id);
    return {
      id: user.id,
      displayName: user.name,          // renamed
      contact: { email: user.email },  // restructured
      metadata: { createdAt: user.createdAt.toISOString() },
    };
  });
}, { prefix: '/api/v2' });
```

### Accept Header Versioning

```typescript
// Fastify — content negotiation versioning
app.get('/api/users/:id', {
  constraints: {
    version: '1.0.0',
  },
}, async (req) => {
  // V1 handler
  return { id: req.params.id, name: 'Alice' };
});

app.get('/api/users/:id', {
  constraints: {
    version: '2.0.0',
  },
}, async (req) => {
  // V2 handler
  return { id: req.params.id, displayName: 'Alice' };
});

// Client sends: Accept-Version: 2.0.0
```

### Query Parameter Versioning

```typescript
// Simple query param approach
app.get('/api/users/:id', async (req, reply) => {
  const version = req.query.version ?? '1';

  const user = await userService.findById(req.params.id);

  if (version === '2') {
    return { id: user.id, displayName: user.name };
  }

  // Default: V1
  return { id: user.id, name: user.name };
});
```

> 💡 URL path versioning is the industry standard for public APIs (Stripe, GitHub, Twilio all use it). Accept header versioning is theoretically more RESTful but harder for consumers to discover. Choose URL path unless you have a specific reason not to.

---

## 4. Deprecation: Sunset and Deprecation Headers

When retiring a version, give consumers time to migrate. The `Deprecation` and `Sunset` HTTP headers (RFC 8594 and draft RFC) communicate this in a machine-readable way.

```typescript
// Fastify hook to add deprecation headers to V1 routes
app.addHook('onSend', async (req, reply) => {
  if (req.url.startsWith('/api/v1/')) {
    // Deprecation: the date when the route became deprecated
    reply.header('Deprecation', 'Sat, 01 Jan 2026 00:00:00 GMT');
    // Sunset: the date when the route will be removed
    reply.header('Sunset', 'Mon, 01 Jun 2026 00:00:00 GMT');
    // Link to migration guide
    reply.header('Link', '</docs/migration/v1-to-v2>; rel="deprecation"');
  }
});
```

**Sunset period guidelines:**
- Internal APIs: minimum 2 sprint cycles (4 weeks)
- External APIs with few consumers: 3-6 months
- Public APIs with many consumers: 12-24 months (Stripe gives 3+ years)

**Deprecation process:**
1. Announce via email, changelog, developer portal
2. Add `Deprecation` and `Sunset` headers to all deprecated responses
3. Log usage of deprecated endpoints to identify stragglers
4. Send reminder emails as Sunset date approaches
5. Remove endpoint on Sunset date

```typescript
// Log deprecated endpoint usage for monitoring
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

## 5. OpenAPI 3.x: Defining and Documenting the Contract

OpenAPI (formerly Swagger) is the standard for describing REST APIs in a machine-readable format (YAML/JSON). It drives documentation, client code generation, and contract testing.

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
      description: 'API for managing orders',
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
// Define schemas inline with routes
const CreateOrderSchema = {
  $id: 'CreateOrderRequest',
  type: 'object',
  required: ['customerId', 'items'],
  properties: {
    customerId: { type: 'string', format: 'uuid', description: 'Customer UUID' },
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
    summary: 'Create a new order',
    description: 'Creates a new order for the specified customer.',
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

## 6. Contract Testing with Pact

Contract testing verifies that a consumer and provider agree on an API contract — without requiring both to be running simultaneously. Pact is the industry standard for consumer-driven contract testing.

**How it works:**
1. The consumer writes a contract (what it expects from the provider)
2. The contract is published to a Pact Broker
3. The provider runs verification: does my current implementation satisfy the contract?
4. Both sides can deploy independently as long as the contract is satisfied

```typescript
// Consumer side — Order Frontend tests what it expects from Order API
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
    it('returns an order when it exists', async () => {
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

    it('returns 404 when order does not exist', async () => {
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
// Provider side — Order API verifies it satisfies the contract
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

  it('satisfies all consumer contracts', async () => {
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

> 💡 Contract tests give you the confidence to deploy consumer and provider independently. The Pact Broker's "can-i-deploy" tool checks whether a specific version of consumer/provider satisfies all current contracts before deployment.

---

## 7. GraphQL Versioning

GraphQL takes a different approach to versioning: the schema is designed to evolve without version numbers. Instead of creating `v2` of an API, you extend the schema and deprecate old fields.

```graphql
type User {
  id: ID!
  name: String! @deprecated(reason: "Use 'displayName' instead, deprecated since 2026-01")
  displayName: String!
  email: String!
  # New field added — existing queries are unaffected
  avatarUrl: String
}

type Query {
  user(id: ID!): User
  # Old query deprecated
  getUser(id: ID!): User @deprecated(reason: "Use 'user(id)' instead")
}
```

```typescript
// TypeScript schema definition with deprecation
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

**GraphQL versioning principles:**
- Never break existing queries — clients should continue to work without changes
- Add new fields freely; deprecate old ones with `@deprecated`
- Remove fields only after a long deprecation period and when usage drops to zero
- Use persisted queries to track which clients use which fields
- Schema stitching and federation allow multiple services to compose one graph

**Monitoring field usage:**

```typescript
// Apollo Server — track deprecated field usage
const server = new ApolloServer({
  schema,
  plugins: [
    {
      requestDidStart: async () => ({
        willSendResponse: async ({ document }) => {
          // Introspect the query for deprecated field usage
          // Log to your monitoring system
        },
      }),
    },
  ],
});
```

> ⚠️ Without monitoring which clients use which deprecated fields, you cannot safely remove them. Always instrument field usage before deprecating.

---

## 8. Common Mistakes & Pitfalls

| Mistake | Consequence | Fix |
|---------|-------------|-----|
| No versioning at all | Breaking changes silently break all clients | Adopt URL path versioning from day 1 |
| Versioning every minor change | Version proliferation; consumers must keep upgrading | Version only on breaking changes |
| Keeping old versions forever | Maintenance burden doubles with each version | Set and enforce Sunset dates |
| No Deprecation/Sunset headers | Consumers cannot detect deprecation automatically | Always add headers when deprecating |
| Sharing the request/response type across versions | V2 changes leak into V1 | Define separate types per version |
| No contract tests | Breaking the consumer is discovered at deployment | Add Pact or schema-level contract tests |
| Breaking changes in patch releases | "It's just a small fix" — but it broke the mobile app | Enforce breaking change classification in PR review |

---

## 9. When to Use / Not Use Different Strategies

| Situation | Recommended approach |
|-----------|---------------------|
| Public API with many external consumers | URL path versioning + long sunset period (12+ months) |
| Internal API between your own services | Contract testing (Pact) + short sunset period |
| GraphQL API | Schema evolution with `@deprecated`, no version numbers |
| REST API where clients are also your own code | Contract tests; version only when necessary |
| API gateway routing multiple backends | URL path versioning — easy to route per version |
| Mobile app backend | URL path versioning — you cannot force users to update apps |

---

## 10. Interview Questions

**Q1: What is the difference between a breaking and a non-breaking change? Give two examples of each.**

A: A non-breaking change does not require existing consumers to change anything. Examples: adding a new optional response field, adding a new endpoint. A breaking change forces existing consumers to update or they will fail. Examples: renaming a field, changing a field's type from `string` to `number`, removing an endpoint, making an optional field required.

---

**Q2: What is consumer-driven contract testing and why is it valuable in a microservices architecture?**

A: Consumer-driven contract testing lets the consumer define what it expects from the provider, and the provider verifies it independently. It is valuable because it replaces end-to-end integration tests (which require both services running) with isolated tests. Each service can be tested and deployed independently, as long as the current version satisfies all registered consumer contracts.

---

**Q3: What are the trade-offs between URL path versioning and Accept header versioning?**

A: URL path versioning (`/api/v2/`) is explicit, browser-testable, easy to cache, and easy to route at the API gateway. The downside is that URLs change and it can feel un-RESTful. Accept header versioning (`Accept: application/vnd.api+json; version=2`) keeps URLs stable and is more RESTful, but is harder to test manually, not supported by all clients by default, and harder to cache. In practice, URL path is the dominant choice for public APIs.

---

**Q4: How does GraphQL handle API evolution differently from REST?**

A: GraphQL is designed to evolve without version numbers. Because clients specify exactly which fields they need in their query, adding new fields never breaks existing clients. Fields are deprecated with `@deprecated` directives, and removal happens only after confirmed zero usage. REST APIs with JSON responses can also evolve without breaking by adding optional fields, but REST lacks the schema enforcement and client-specified field selection that makes GraphQL evolution seamless.

---

**Q5: What should the sunset period be for a public API?**

A: It depends on your consumer base. For third-party developers: a minimum of 12 months, ideally 18-24 months (Stripe gives 3+ years for major versions). For internal APIs between your own teams: 2-4 sprint cycles (4-8 weeks). For APIs backing mobile apps: align with the time it takes your slowest users to update their app — this can be 12+ months if you cannot force updates.

---

**Q6: How would you handle a situation where you need to make a breaking change to an API that hundreds of clients depend on?**

A: (1) Release the new behavior in a new API version (`v2`). (2) Add `Deprecation` and `Sunset` headers to all `v1` responses, giving 12+ months notice. (3) Publish a migration guide. (4) Monitor `v1` usage by consumer to identify stragglers. (5) Reach out directly to high-traffic consumers. (6) At 90 days before sunset, send email reminders. (7) Remove `v1` on the sunset date. Never remove a version early.

---

**Q7: What is the "Tolerant Reader" pattern and how does it relate to API versioning?**

A: The Tolerant Reader pattern (Martin Fowler) says consumers should be lenient in what they accept: ignore unknown fields, use defaults for missing optional fields, and not fail on unrecognized enum values. Consumers that implement Tolerant Reader are more resilient to non-breaking changes and can handle provider-side additions without requiring consumer updates. This is particularly important in event-driven systems where you cannot version events as easily as HTTP responses.

---

## 11. Exercises

**Exercise 1: Classify changes as breaking or non-breaking**

For each of the following, classify as breaking or non-breaking and explain why:

a) Adding `middleName?: string` to a user response
b) Changing `price: number` to `price: { amount: number; currency: string }`
c) Removing the `GET /api/v1/users/:id/preferences` endpoint
d) Adding `GET /api/v1/users/:id/settings` (new endpoint)
e) Changing `status` from `'active' | 'inactive'` to `'active' | 'inactive' | 'suspended'`
f) Making `email` required in a request that previously had it as optional

---

**Exercise 2: Add versioning to an existing Fastify app**

Take a Fastify app with routes at `/users`, `/orders`, and `/products`. Add URL path versioning so existing routes become `/api/v1/...`. Create a `/api/v2/users/:id` route that returns a different response shape (rename `name` to `displayName`). Add deprecation headers to all v1 routes with a 6-month sunset.

---

**Exercise 3: Write a Pact consumer test**

Write a Pact consumer test for a `ProductApiClient` that calls `GET /api/v1/products/:id`. The consumer expects: `{ id, name, price, inStock }`. Write the consumer contract, then write a stub provider verification that seeds test data.

---

**Exercise 4: OpenAPI schema for an Orders API**

Write a complete OpenAPI 3.x schema (YAML or JSON) for the following endpoints:
- `POST /api/v2/orders` — create order
- `GET /api/v2/orders/:id` — get order
- `PATCH /api/v2/orders/:id/cancel` — cancel order

Include schemas, required fields, error responses (400, 401, 404), and security (Bearer JWT).

---

## 12. Further Reading

- **[Semantic Versioning](https://semver.org/)** — the standard for version numbering
- **[RFC 8594 — Sunset Header](https://tools.ietf.org/html/rfc8594)** — official RFC for the Sunset header
- **[Pact Documentation](https://docs.pact.io/)** — consumer-driven contract testing framework
- **[OpenAPI Specification](https://spec.openapis.org/oas/v3.1.0)** — official OpenAPI 3.1.0 spec
- **[@fastify/swagger](https://github.com/fastify/fastify-swagger)** — Fastify OpenAPI integration
- **[GraphQL Best Practices — Versioning](https://graphql.org/learn/best-practices/#versioning)** — official GraphQL guidance
- **[API Design Patterns](https://www.manning.com/books/api-design-patterns)** — JJ Geewax (Manning, 2021)
- **[Stripe API Versioning](https://stripe.com/docs/api/versioning)** — how Stripe manages 3+ years of API versions
