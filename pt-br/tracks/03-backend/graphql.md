# GraphQL

## 1. O Que É e Por Que Usar

GraphQL é uma linguagem de consulta para APIs e um runtime no servidor para executar essas consultas. Desenvolvido internamente no Facebook em 2012 e open-sourced em 2015, fornece uma descrição completa e compreensível dos dados da API, dá ao cliente o poder de pedir exatamente o que precisa e facilita a evolução da API ao longo do tempo.

O insight central que motivou o GraphQL foi que o app mobile do Facebook fazia dezenas de chamadas REST por tela, recebendo respostas enormes onde apenas uma fração dos campos era usada — desperdício de banda e bateria. Ao mesmo tempo, outras telas precisavam de dados de múltiplos endpoints, gerando requisições em cascata. GraphQL resolve ambos: uma requisição, exatamente os dados necessários.

GraphQL não substitui REST em todos os casos. É mais adequado quando múltiplos clientes heterogêneos precisam de visões diferentes dos mesmos dados.

---

## 2. Conceitos Fundamentais

### Schema Definition Language (SDL)

O schema é o contrato entre cliente e servidor. É fortemente tipado e auto-documentável.

```graphql
# Tipos escalares: String, Int, Float, Boolean, ID (mais escalares customizados)

# Tipo objeto
type User {
  id: ID!
  name: String!
  email: String!
  role: UserRole!
  posts: [Post!]!
  createdAt: String!
}

# Enum
enum UserRole {
  USER
  ADMIN
  MODERATOR
}

# Interface — campos compartilhados entre tipos
interface Node {
  id: ID!
}

type Post implements Node {
  id: ID!
  title: String!
  body: String!
  author: User!
  tags: [String!]!
  publishedAt: String
  commentCount: Int!
}

# Union — campo que pode ser um de vários tipos
union SearchResult = User | Post | Comment

# Input type — para mutations (não se usa tipos regulares como inputs)
input CreatePostInput {
  title: String!
  body: String!
  tags: [String!]
}

input UpdatePostInput {
  title: String
  body: String
  tags: [String!]
}

# Tipos raiz
type Query {
  user(id: ID!): User
  users(limit: Int, cursor: String, role: UserRole): UserConnection!
  post(id: ID!): Post
  search(query: String!): [SearchResult!]!
}

type Mutation {
  createPost(input: CreatePostInput!): Post!
  updatePost(id: ID!, input: UpdatePostInput!): Post
  deletePost(id: ID!): Boolean!
  login(email: String!, password: String!): AuthPayload!
}

type Subscription {
  postCreated: Post!
  commentAdded(postId: ID!): Comment!
}

# Padrão Connection para paginação
type UserConnection {
  edges: [UserEdge!]!
  pageInfo: PageInfo!
}
type UserEdge {
  node: User!
  cursor: String!
}
type PageInfo {
  hasNextPage: Boolean!
  endCursor: String
}

type AuthPayload {
  token: String!
  user: User!
}
```

> O sufixo `!` indica não-nulável. `[Post!]!` significa lista não-nulável de Posts não-nuláveis. Prefira campos anuláveis a menos que haja uma garantia forte — nulls inesperados causam erros no cliente, mas o cliente consegue lidar com null.

---

### Resolvers

Resolvers são funções que retornam os dados de cada campo no schema.

```typescript
// Assinatura: (parent, args, context, info) => valor | Promise<valor>
//
// parent (ou root): valor resolvido do tipo pai
// args:    argumentos do campo na query
// context: objeto compartilhado por requisição (conexão ao banco, usuário atual, instâncias de DataLoader)
// info:    metadados da AST da query (casos avançados)

const resolvers = {
  Query: {
    user: async (_parent, { id }, context) => {
      return context.db.users.findUnique({ where: { id } });
    },
    users: async (_parent, { limit = 20, cursor, role }, context) => {
      // ... implementação de paginação
    },
  },

  Mutation: {
    createPost: async (_parent, { input }, context) => {
      if (!context.currentUser) throw new GraphQLError('Não autenticado', {
        extensions: { code: 'UNAUTHENTICATED' },
      });
      return context.db.posts.create({
        data: { ...input, authorId: context.currentUser.id },
      });
    },
  },

  // Resolvers de tipo — resolvem campos em um tipo
  User: {
    // Resolve o campo 'posts' de um User
    // parent aqui é o objeto User retornado de Query.user
    posts: async (parent, _args, context) => {
      return context.db.posts.findMany({
        where: { authorId: parent.id },
      });
    },
  },

  Post: {
    author: async (parent, _args, context) => {
      return context.db.users.findUnique({ where: { id: parent.authorId } });
    },
  },

  // Resolver de Union/Interface
  SearchResult: {
    __resolveType(obj) {
      if (obj.title !== undefined) return 'Post';
      if (obj.email !== undefined) return 'User';
      return 'Comment';
    },
  },
};
```

---

## 3. Como Funciona

### O Problema N+1

Este é o problema de performance mais crítico em GraphQL. Exemplo concreto:

```graphql
# Query do cliente
query {
  posts(limit: 100) {
    id
    title
    author {
      name
    }
  }
}
```

**Execução ingênua sem DataLoader:**

```
1. Resolver Query.posts → SELECT * FROM posts LIMIT 100   (1 query)
2. Resolver Post.author para post[0]  → SELECT * FROM users WHERE id = 1
3. Resolver Post.author para post[1]  → SELECT * FROM users WHERE id = 2
4. Resolver Post.author para post[2]  → SELECT * FROM users WHERE id = 1 ← mesmo usuário de novo!
...
100. Resolver Post.author para post[99] → SELECT * FROM users WHERE id = 42

Total: 1 + 100 = 101 queries
```

Pior ainda: muitas dessas queries são para os mesmos IDs (autores escrevem múltiplos posts).

### DataLoader: Batch + Cache por Requisição

DataLoader resolve o N+1 agrupando carregamentos individuais em uma única query em batch e cacheando dentro do tempo de vida da requisição.

```typescript
import DataLoader from 'dataloader';
import { PrismaClient } from '@prisma/client';

const db = new PrismaClient();

// Função batch do DataLoader: recebe array de keys, retorna array de valores na mesma ordem
function createUserLoader() {
  return new DataLoader<string, User | null>(
    async (userIds: readonly string[]) => {
      // Uma query para todos os IDs solicitados
      const users = await db.user.findMany({
        where: { id: { in: [...userIds] } },
      });

      // DataLoader exige resultados na MESMA ORDEM das keys
      // Monta um mapa para lookup O(1)
      const userMap = new Map(users.map((u) => [u.id, u]));
      return userIds.map((id) => userMap.get(id) ?? null);
    },
    {
      // Cache: dentro da requisição, o mesmo ID nunca é buscado duas vezes
      cache: true,
      // Opcional: agrupa até 100 keys por query SQL
      maxBatchSize: 100,
    }
  );
}

// Cria uma nova instância de DataLoader POR REQUISIÇÃO (não global — isso compartilharia cache entre usuários!)
function createContext({ req }: { req: Request }) {
  return {
    currentUser: getUserFromToken(req.headers.authorization),
    db,
    loaders: {
      user: createUserLoader(),
      // Adicione mais loaders conforme necessário: post, comment, tag, etc.
    },
  };
}

// Resolver agora usa o loader
const resolvers = {
  Post: {
    author: async (parent: Post, _args: unknown, context: Context) => {
      return context.loaders.user.load(parent.authorId);
      // DataLoader agrupa todos os .load() chamados durante este tick
      // em um único findMany() → as 100 queries do N+1 viram 2 queries
    },
  },
};
```

**Execução com DataLoader:**

```
1. Resolver Query.posts → SELECT * FROM posts LIMIT 100   (1 query)
2. DataLoader coleta: [userId1, userId2, userId1, ..., userId42]
   Deduplica: [userId1, userId2, ..., uniqueUserIds] (digamos 12 únicos)
3. Função batch dispara: SELECT * FROM users WHERE id IN (1, 2, ..., 12)  (1 query)

Total: 2 queries independente do número de posts
```

> Sempre crie instâncias de DataLoader por requisição, nunca como singletons globais. Um DataLoader global cachearia dados entre requisições — um usuário poderia receber dados obsoletos de outro usuário.

---

## 4. Exemplos de Código (TypeScript)

### Servidor GraphQL Completo com Fastify + Mercurius

```typescript
import Fastify from 'fastify';
import mercurius from 'mercurius';
import DataLoader from 'dataloader';
import { PrismaClient } from '@prisma/client';

const db = new PrismaClient();
const app = Fastify({ logger: true });

const schema = `
  type Query {
    post(id: ID!): Post
    posts(limit: Int, cursor: String): PostConnection!
  }
  type Mutation {
    createPost(input: CreatePostInput!): Post!
  }
  type Post {
    id: ID!
    title: String!
    author: User!
  }
  type User {
    id: ID!
    name: String!
  }
  type PostConnection {
    edges: [PostEdge!]!
    pageInfo: PageInfo!
  }
  type PostEdge { node: Post! cursor: String! }
  type PageInfo { hasNextPage: Boolean! endCursor: String }
  input CreatePostInput { title: String! body: String! }
`;

const resolvers = {
  Query: {
    post: (_: unknown, { id }: { id: string }, ctx: Context) =>
      ctx.loaders.post.load(id),

    posts: async (_: unknown, { limit = 20, cursor }: { limit?: number; cursor?: string }, ctx: Context) => {
      const cursorId = cursor
        ? Buffer.from(cursor, 'base64url').toString()
        : undefined;

      const posts = await db.post.findMany({
        take: limit + 1,
        ...(cursorId && { cursor: { id: cursorId }, skip: 1 }),
        orderBy: { createdAt: 'desc' },
      });

      const hasNextPage = posts.length > limit;
      const edges = posts.slice(0, limit).map((post) => ({
        node: post,
        cursor: Buffer.from(post.id).toString('base64url'),
      }));

      return {
        edges,
        pageInfo: {
          hasNextPage,
          endCursor: edges[edges.length - 1]?.cursor ?? null,
        },
      };
    },
  },

  Mutation: {
    createPost: async (
      _: unknown,
      { input }: { input: { title: string; body: string } },
      ctx: Context
    ) => {
      if (!ctx.user) {
        throw new mercurius.ErrorWithProps('Não autenticado', {}, 401);
      }
      return db.post.create({
        data: { ...input, authorId: ctx.user.id },
      });
    },
  },

  Post: {
    author: (post: { authorId: string }, _: unknown, ctx: Context) =>
      ctx.loaders.user.load(post.authorId),
  },
};

type Context = {
  user: { id: string } | null;
  loaders: {
    post: DataLoader<string, unknown>;
    user: DataLoader<string, unknown>;
  };
};

await app.register(mercurius, {
  schema,
  resolvers,
  context: (req) => ({
    user: getUserFromToken(req.headers.authorization),
    loaders: {
      user: new DataLoader(async (ids: readonly string[]) => {
        const users = await db.user.findMany({ where: { id: { in: [...ids] } } });
        const map = new Map(users.map((u) => [u.id, u]));
        return ids.map((id) => map.get(id) ?? null);
      }),
      post: new DataLoader(async (ids: readonly string[]) => {
        const posts = await db.post.findMany({ where: { id: { in: [...ids] } } });
        const map = new Map(posts.map((p) => [p.id, p]));
        return ids.map((id) => map.get(id) ?? null);
      }),
    },
  }),
  graphiql: process.env.NODE_ENV === 'development',
});
```

### Tratamento de Erros: Sucesso Parcial

O modelo de erros do GraphQL é único: uma resposta pode ter tanto `data` quanto `errors`. Alguns campos tiveram sucesso; outros falharam. Isso se chama **sucesso parcial**.

```typescript
import { GraphQLError } from 'graphql';

// Tipos de erro customizados
class NotFoundError extends GraphQLError {
  constructor(resource: string, id: string) {
    super(`${resource} com id ${id} não encontrado`, {
      extensions: {
        code: 'NOT_FOUND',
        resource,
        id,
      },
    });
  }
}

class ForbiddenError extends GraphQLError {
  constructor(action: string) {
    super(`Sem autorização para ${action}`, {
      extensions: { code: 'FORBIDDEN' },
    });
  }
}

// Resolver que lança erro — GraphQL captura e adiciona ao array errors
const resolvers = {
  Query: {
    user: async (_: unknown, { id }: { id: string }, ctx: Context) => {
      const user = await db.user.findUnique({ where: { id } });
      if (!user) throw new NotFoundError('User', id);
      if (!ctx.user?.isAdmin && ctx.user?.id !== id) {
        throw new ForbiddenError('ver outros usuários');
      }
      return user;
    },
  },
};

// Exemplo de resposta do cliente:
// {
//   "data": {
//     "posts": [...],       ← sucesso
//     "user": null          ← falhou, definido como null
//   },
//   "errors": [{
//     "message": "User com id 999 não encontrado",
//     "locations": [{ "line": 3, "column": 5 }],
//     "path": ["user"],
//     "extensions": { "code": "NOT_FOUND", "resource": "User", "id": "999" }
//   }]
// }
```

### Subscriptions com WebSockets

```typescript
// Schema
const schema = `
  type Subscription {
    commentAdded(postId: ID!): Comment!
  }
`;

// Servidor (Mercurius com suporte a subscription)
await app.register(mercurius, {
  schema,
  resolvers: {
    Subscription: {
      commentAdded: {
        subscribe: async function* (_: unknown, { postId }: { postId: string }, ctx: Context) {
          // Emite valores conforme chegam
          const pubsub = ctx.pubsub;
          for await (const event of pubsub.subscribe(`comment:${postId}`)) {
            yield { commentAdded: event };
          }
        },
      },
    },
  },
  subscription: true, // habilita suporte a WebSocket
});

// Query do cliente
const COMMENT_SUBSCRIPTION = `
  subscription OnCommentAdded($postId: ID!) {
    commentAdded(postId: $postId) {
      id
      body
      author { name }
    }
  }
`;
```

### Persisted Queries

```typescript
// Persisted queries: o cliente envia um hash em vez da query completa
// Reduz o tamanho do payload (especialmente em mobile), impede queries arbitrárias em produção

// 1. No build, gera o manifesto de queries:
// { "sha256:abc123": "query GetUser($id: ID!) { user(id: $id) { name email } }" }

// 2. Servidor: busca a query pelo hash
app.register(mercurius, {
  schema,
  resolvers,
  persistedQueryProvider: {
    getQueryFromHash: async (hash: string) => {
      return queryManifest[hash]; // busca em arquivo/Redis
    },
    // Opcionalmente permite hashes desconhecidos (em desenvolvimento):
    // getHashForQuery: sha256,
    // notFoundError: new Error('Query desconhecida'),
  },
});

// 3. Cliente envia:
// POST /graphql
// { "extensions": { "persistedQuery": { "version": 1, "sha256Hash": "abc123" } } }
// Sem campo "query" — o servidor resolve pelo hash
```

---

## 5. Erros Comuns e Armadilhas

**Criar instâncias de DataLoader globalmente:**

```typescript
// ERRADO — loader global único cacheia dados entre todos os usuários/requisições
const userLoader = new DataLoader(batchFn);

// CORRETO — nova instância por requisição (criada na factory de contexto)
function createContext(req) {
  return {
    userLoader: new DataLoader(batchFn), // cache fresco por requisição
  };
}
```

**Retornar null em vez de lançar erro:**

```typescript
// Errado — cliente recebe null sem explicação
const resolver = async (_: unknown, { id }: { id: string }) => {
  const user = await db.user.findUnique({ where: { id } });
  return user ?? null; // retorna null silenciosamente — cliente não sabe por quê
};

// Correto — lance para popular o array errors
const resolver = async (_: unknown, { id }: { id: string }) => {
  const user = await db.user.findUnique({ where: { id } });
  if (!user) throw new GraphQLError(`User ${id} não encontrado`, {
    extensions: { code: 'NOT_FOUND' },
  });
  return user;
};
```

**Esquecer que queries GraphQL podem ter aninhamento profundo:**

Uma query maliciosa pode solicitar dados profundamente aninhados causando carga exponencial no banco:

```graphql
# Malicioso — N^profundidade queries
query {
  users {
    friends {
      friends {
        friends {
          # ... 10 níveis de profundidade
        }
      }
    }
  }
}
```

Estratégias de proteção:
- Limitação de profundidade (`graphql-depth-limit`)
- Análise de complexidade da query (`graphql-query-complexity`)
- Persisted queries em produção (só permite queries conhecidas)
- Rate limiting na camada HTTP

**Expor erros internos para clientes:**

```typescript
// Errado — stack traces e erros do banco expostos
const server = new ApolloServer({
  // padrão em desenvolvimento expõe todos os erros
});

// Correto — mascara erros inesperados em produção
const server = new ApolloServer({
  formatError: (formattedError, error) => {
    // Só expõe erros esperados (com extensions.code)
    if (formattedError.extensions?.code) {
      return formattedError; // erro conhecido, seguro de expor
    }
    // Loga erros inesperados mas retorna mensagem genérica
    console.error('Erro GraphQL inesperado:', error);
    return {
      message: 'Erro interno do servidor',
      extensions: { code: 'INTERNAL_SERVER_ERROR' },
    };
  },
});
```

---

## 6. Quando Usar / Não Usar

**GraphQL é a escolha certa quando:**
- Múltiplos clientes (iOS, Android, web, terceiros) precisam de formatos diferentes dos mesmos dados
- Times de frontend iteram rapidamente e mudam frequentemente os requisitos de dados
- Os dados são profundamente aninhados com relacionamentos complexos (grafos sociais, árvores de conteúdo)
- Você quer substituir uma proliferação de endpoints REST especializados
- Você precisa de subscriptions em tempo real ao lado de queries

**Prefira REST quando:**
- CRUD simples com um único cliente
- HTTP caching é essencial (requisições POST do GraphQL não são cacheadas por padrão)
- API pública com consumidores externos que precisam de estabilidade e documentação
- O time desconhece GraphQL e o prazo é curto
- Upload/download de arquivos é o caso de uso principal (REST lida melhor com binário)

> GraphQL não é inerentemente mais rápido que REST. Sem DataLoader e design cuidadoso de resolvers, pode ser significativamente mais lento por causa do problema N+1. Performance requer esforço deliberado.

---

## 7. Cenário Real

**Problema:** Um app React Native faz 7 chamadas REST separadas na tela inicial — perfil do usuário, pedidos recentes, assinaturas ativas, notificações, recomendações de produtos, pontos de fidelidade e endereço de entrega. Isso causa carregamento inicial lento (requisições sequenciais por dependências) e desperdiça dados em mobile.

**Solução: Consolidar com uma API GraphQL**

```graphql
# Query única substituindo 7 chamadas REST
query HomeScreen($userId: ID!) {
  user(id: $userId) {
    name
    loyaltyPoints
    defaultAddress {
      street
      city
    }
  }
  recentOrders(userId: $userId, limit: 3) {
    id
    status
    total
    items { name }
  }
  activeSubscriptions(userId: $userId) {
    id
    product { name }
    renewalDate
  }
  notifications(userId: $userId, unreadOnly: true, limit: 5) {
    id
    message
    createdAt
  }
  recommendations(userId: $userId, limit: 4) {
    id
    name
    price
    imageUrl
  }
}
```

**Implementação no backend com DataLoader:**

```typescript
// Todos os resolvers executam em paralelo — GraphQL resolve campos independentes concorrentemente
const resolvers = {
  Query: {
    // Esses quatro resolvem em paralelo (sem dependência entre si)
    user: (_, { id }, ctx) => ctx.loaders.user.load(id),
    recentOrders: (_, { userId, limit }, ctx) => ctx.db.orders.findMany({
      where: { userId },
      orderBy: { createdAt: 'desc' },
      take: limit,
      include: { items: true },
    }),
    activeSubscriptions: (_, { userId }, ctx) => ctx.db.subscriptions.findMany({
      where: { userId, status: 'ACTIVE' },
    }),
    notifications: (_, { userId, unreadOnly, limit }, ctx) => ctx.db.notifications.findMany({
      where: { userId, ...(unreadOnly && { readAt: null }) },
      take: limit,
    }),
    recommendations: (_, { userId, limit }, ctx) =>
      ctx.recommendationService.forUser(userId, limit),
  },
};
```

**Resultado:** 7 round trips reduzidos para 1. Tempo de carregamento cai de 3,2s para 800ms. Uso de dados mobile reduzido em 60% porque cada cliente solicita apenas os campos que precisa.

---

## 8. Perguntas de Entrevista

**P1: O que é o problema N+1 em GraphQL e como resolvê-lo?**

O problema N+1 ocorre quando um resolver busca dados relacionados para cada item individualmente: buscando 100 posts, depois fazendo 100 queries separadas para buscar o autor de cada post. Solução: DataLoader. Ele agrupa todas as chamadas individuais `.load(id)` feitas durante um único tick do event loop em uma única chamada de função em batch — tipicamente uma query SQL `WHERE id IN (...)`. Também cacheia resultados dentro da requisição, então o mesmo ID nunca é buscado duas vezes. Regra fundamental: crie um novo DataLoader por requisição para evitar contaminação de cache entre requisições.

**P2: Como o DataLoader funciona internamente?**

DataLoader usa agendamento de microtask: quando você chama `loader.load(id)`, o ID é adicionado a uma fila pendente e uma Promise é retornada. O DataLoader agenda a função batch para rodar no próximo microtask (após todo o código síncrono e operações async atuais). Isso significa que todos os `.load()` de resolvers rodando no mesmo tick são coletados antes do batch executar. A função batch recebe todos os IDs enfileirados, busca em uma query e o DataLoader resolve cada Promise individual com o valor correspondente.

**P3: Como funcionam as subscriptions GraphQL?**

Subscriptions são operações de longa duração via WebSocket (ou SSE). O cliente envia uma query de subscription; o servidor estabelece uma conexão WebSocket e empurra novos dados sempre que o evento inscrito ocorre. O servidor tipicamente usa um sistema pub/sub (Redis, EventEmitter, Kafka) para broadcast de eventos para os inscritos. Cada resolver de subscription é um async generator que emite valores conforme os eventos chegam.

**P4: Quando você escolheria REST em vez de GraphQL?**

REST é melhor quando: HTTP caching é crítico (caching por CDN não funciona bem com requisições POST do GraphQL), você está construindo uma API pública que precisa ser estável e versionada, o time não conhece GraphQL e o tempo é limitado, a API é CRUD simples, ou você lida principalmente com upload/download de arquivos. GraphQL ganha quando: múltiplos clientes precisam de formatos diferentes de dados, reduzir over/under-fetching importa, o domínio tem relacionamentos aninhados complexos, ou você precisa iterar rapidamente no frontend sem mudanças no backend.

**P5: O que é GraphQL Federation?**

Federation é uma forma de dividir um schema GraphQL em múltiplos serviços independentes (subgraphs) enquanto apresenta um schema unificado para clientes através de um gateway. Cada serviço possui sua parte do schema e é deployável independentemente. A diretiva `@key` marca as chaves primárias das entidades para que o gateway possa unir tipos entre serviços. Por exemplo, um serviço User possui o tipo `User`, e um serviço Orders pode estendê-lo com `orders: [Order]` — o gateway resolve a referência entre serviços automaticamente.

**P6: Como você lida com autorização em GraphQL?**

Três abordagens principais: (1) **Em resolvers** — verifica `context.user.role` em cada resolver (flexível mas repetitivo), (2) **Diretivas de schema** — diretivas `@auth` ou `@hasRole` customizadas aplicadas declarativamente no SDL, (3) **Camada de middleware** — `graphql-shield` ou similar para definir regras de permissão separadas da lógica de negócio. Importante: evite depender da introspecção do schema para esconder campos não autorizados — clientes com acesso ao schema ainda podem solicitá-los. Sempre enforque permissões em resolvers ou uma camada de regras, não apenas no schema.

**P7: O que é uma persisted query?**

Uma persisted query substitui a string completa da query por um hash. No build, todas as queries são extraídas e armazenadas no servidor (em arquivo ou Redis) identificadas por seu hash SHA-256. Em runtime, o cliente envia apenas `{ extensions: { persistedQuery: { sha256Hash: "abc..." } } }`. Benefícios: payloads HTTP menores, habilita requisições GET (cacheáveis), e em produção você pode desabilitar completamente a execução de queries arbitrárias — só queries pré-registradas são permitidas, o que previne ataques de introspecção e reduz a superfície de ataque.

---

## 9. Exercícios

**Exercício 1: Construa uma API GraphQL tipada**

Crie uma API GraphQL para um app de gerenciamento de tarefas com:
- Tipos: `Task`, `Project`, `User`
- Queries: `task(id)`, `tasks(projectId, status)`, `project(id)` com tarefas
- Mutations: `createTask`, `updateTask(id, input)`, `completeTask(id)`
- Subscriptions: `taskUpdated(projectId)`
- Implemente DataLoader para lookups de usuário e projeto
- Use TypeScript + graphql-codegen para gerar resolvers tipados

*Dica:* Use `graphql-codegen` com `@graphql-codegen/typescript-resolvers` para gerar tipos de resolver a partir do SDL.

**Exercício 2: Corrija um problema N+1**

Dado este resolver com problema N+1:

```typescript
const resolvers = {
  Query: {
    posts: () => db.post.findMany({ take: 50 }),
  },
  Post: {
    author: (post) => db.user.findUnique({ where: { id: post.authorId } }),
    comments: (post) => db.comment.findMany({ where: { postId: post.id } }),
  },
};
```

Refatore para usar DataLoader. Verifique com um logger de queries que buscar 50 posts com autores e comentários resulta em 3 queries no total em vez de 101+.

*Dica:* Você precisa de dois DataLoaders: um para usuários (por ID), um para comentários (em batch por postId — note que o relacionamento 1:N exige retornar arrays).

**Exercício 3: Análise de complexidade de query**

Implemente análise de complexidade que:
- Atribui scores de complexidade a cada campo (padrão 1, campos de lista = 10)
- Rejeita queries com complexidade total > 1000
- Retorna status `429` com o budget de complexidade restante na resposta

*Dica:* Use a biblioteca `graphql-query-complexity` com `fieldExtensionsEstimator` e `simpleEstimator`.

---

## 10. Leituras Complementares

- [Especificação oficial do GraphQL](https://spec.graphql.org/)
- [Boas práticas GraphQL (graphql.org)](https://graphql.org/learn/best-practices/)
- [Biblioteca DataLoader (GitHub)](https://github.com/graphql/dataloader)
- [Mercurius — adapter GraphQL para Fastify](https://mercurius.dev/)
- [Documentação Apollo Server](https://www.apollographql.com/docs/apollo-server/)
- [graphql-shield — camada de permissões](https://the-guild.dev/graphql/shield)
- [graphql-query-complexity](https://github.com/slicknode/graphql-query-complexity)
- [GraphQL Federation (Apollo)](https://www.apollographql.com/docs/federation/)
- [The Guild — ecossistema de ferramentas GraphQL](https://the-guild.dev/)
- Livro: *Production Ready GraphQL* por Marc-André Giroux
