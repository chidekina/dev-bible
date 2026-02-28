# GraphQL

## 1. What & Why

GraphQL is a query language for APIs and a server-side runtime for executing those queries, developed internally at Facebook in 2012 and open-sourced in 2015. It provides a complete and understandable description of the data in your API, gives clients the power to ask for exactly what they need, and makes it easier to evolve APIs over time.

The core insight that motivated GraphQL was that Facebook's mobile app was making dozens of REST API calls per screen, receiving massive responses where only a fraction of fields were used. This over-fetching wasted bandwidth and battery. At the same time, other screens needed data from multiple endpoints, requiring waterfall requests. GraphQL solves both: one request, exactly the data needed.

GraphQL is not a replacement for REST in all cases. It is a better fit for certain problems — primarily when multiple heterogeneous clients need different views of the same data.

---

## 2. Core Concepts

### Schema Definition Language (SDL)

The schema is the contract between client and server. It is strongly typed and self-documenting.

```graphql
# Scalar types: String, Int, Float, Boolean, ID (plus custom scalars)

# Object type
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

# Interface — shared fields across types
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

# Union — a field that can be one of several types
union SearchResult = User | Post | Comment

# Input type — for mutations (cannot use regular types as inputs)
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

# Root types
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

# Connection pattern for pagination
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

> 💡 The `!` suffix means non-nullable. `[Post!]!` means a non-nullable list of non-nullable Posts. Lean towards nullable fields unless you have a strong guarantee — unexpected nulls cause client errors, but clients can handle null.

---

### Resolvers

Resolvers are functions that return the data for each field in the schema.

```typescript
// Resolver signature: (parent, args, context, info) => value | Promise<value>
//
// parent (or root): the resolved value of the parent type
// args:    field arguments from the query
// context: shared per-request object (db connection, current user, DataLoader instances)
// info:    query AST metadata (used in advanced cases)

const resolvers = {
  Query: {
    user: async (_parent, { id }, context) => {
      return context.db.users.findUnique({ where: { id } });
    },
    users: async (_parent, { limit = 20, cursor, role }, context) => {
      // ... pagination implementation
    },
  },

  Mutation: {
    createPost: async (_parent, { input }, context) => {
      if (!context.currentUser) throw new GraphQLError('Not authenticated', {
        extensions: { code: 'UNAUTHENTICATED' },
      });
      return context.db.posts.create({
        data: { ...input, authorId: context.currentUser.id },
      });
    },
  },

  // Type resolvers — resolve fields on a type
  User: {
    // Resolve the 'posts' field of a User
    // parent here is the User object returned from Query.user
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

  // Union/Interface resolver
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

## 3. How It Works

### The N+1 Problem

This is the most critical performance issue in GraphQL. Here is a concrete example:

```graphql
# Client query
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

**Naive resolver execution without DataLoader:**

```
1. Query.posts resolver → SELECT * FROM posts LIMIT 100   (1 query)
2. Post.author resolver for post[0]  → SELECT * FROM users WHERE id = 1
3. Post.author resolver for post[1]  → SELECT * FROM users WHERE id = 2
4. Post.author resolver for post[2]  → SELECT * FROM users WHERE id = 1 ← same user again!
...
100. Post.author resolver for post[99] → SELECT * FROM users WHERE id = 42

Total: 1 + 100 = 101 queries
```

Even worse: many of those user queries are for the same user IDs (authors write multiple posts).

### DataLoader: Batch + Cache per Request

DataLoader solves N+1 by coalescing individual loads into a single batched query, and caching within the request lifetime.

```typescript
import DataLoader from 'dataloader';
import { PrismaClient } from '@prisma/client';

const db = new PrismaClient();

// DataLoader batch function: receives an array of keys, returns array of values in same order
function createUserLoader() {
  return new DataLoader<string, User | null>(
    async (userIds: readonly string[]) => {
      // One query for all requested user IDs
      const users = await db.user.findMany({
        where: { id: { in: [...userIds] } },
      });

      // DataLoader requires results in the SAME ORDER as keys
      // Build a map for O(1) lookup
      const userMap = new Map(users.map((u) => [u.id, u]));
      return userIds.map((id) => userMap.get(id) ?? null);
    },
    {
      // Cache: within this request, same ID is never fetched twice
      cache: true,
      // Optional: batch up to 100 keys per SQL query
      maxBatchSize: 100,
    }
  );
}

// Create a new DataLoader instance PER REQUEST (not global — that would share cache across users!)
function createContext({ req }: { req: Request }) {
  return {
    currentUser: getUserFromToken(req.headers.authorization),
    db,
    loaders: {
      user: createUserLoader(),
      // Add more loaders as needed: post, comment, tag, etc.
    },
  };
}

// Resolver now uses the loader
const resolvers = {
  Post: {
    author: async (parent: Post, _args: unknown, context: Context) => {
      return context.loaders.user.load(parent.authorId);
      // DataLoader batches all .load() calls made during this tick
      // into a single findMany() → the 100-query N+1 becomes 2 queries
    },
  },
};
```

**Execution with DataLoader:**

```
1. Query.posts resolver → SELECT * FROM posts LIMIT 100   (1 query)
2. DataLoader collects: [userId1, userId2, userId1, ..., userId42]
   Deduplicates: [userId1, userId2, ..., uniqueUserIds] (say 12 unique)
3. Batch function fires: SELECT * FROM users WHERE id IN (1, 2, ..., 12)  (1 query)

Total: 2 queries regardless of number of posts
```

> ⚠️ Always create DataLoader instances per request, not as global singletons. A global DataLoader would cache user data across requests — a user could receive another user's stale data.

---

## 4. Code Examples (TypeScript)

### Full GraphQL Server with Fastify + Mercurius

```typescript
import Fastify from 'fastify';
import mercurius from 'mercurius';
import { z } from 'zod';
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
        throw new mercurius.ErrorWithProps('Not authenticated', {}, 401);
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

### Error Handling: Partial Success

GraphQL's error model is unique: a response can have both `data` and `errors`. Some fields succeeded; others failed. This is called **partial success**.

```typescript
import { GraphQLError } from 'graphql';

// Custom error types
class NotFoundError extends GraphQLError {
  constructor(resource: string, id: string) {
    super(`${resource} with id ${id} not found`, {
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
    super(`Not authorized to ${action}`, {
      extensions: { code: 'FORBIDDEN' },
    });
  }
}

// Resolver that throws — GraphQL catches and adds to errors array
const resolvers = {
  Query: {
    user: async (_: unknown, { id }: { id: string }, ctx: Context) => {
      const user = await db.user.findUnique({ where: { id } });
      if (!user) throw new NotFoundError('User', id);
      if (!ctx.user?.isAdmin && ctx.user?.id !== id) {
        throw new ForbiddenError('view other users');
      }
      return user;
    },
  },
};

// Client response example:
// {
//   "data": {
//     "posts": [...],       ← succeeded
//     "user": null          ← failed, set to null
//   },
//   "errors": [{
//     "message": "User with id 999 not found",
//     "locations": [{ "line": 3, "column": 5 }],
//     "path": ["user"],
//     "extensions": { "code": "NOT_FOUND", "resource": "User", "id": "999" }
//   }]
// }
```

### Subscriptions with WebSockets

```typescript
// Schema
const schema = `
  type Subscription {
    commentAdded(postId: ID!): Comment!
  }
`;

// Server (Mercurius with subscription support)
await app.register(mercurius, {
  schema,
  resolvers: {
    Subscription: {
      commentAdded: {
        subscribe: async function* (_: unknown, { postId }: { postId: string }, ctx: Context) {
          // Yield values as they arrive
          const pubsub = ctx.pubsub;
          for await (const event of pubsub.subscribe(`comment:${postId}`)) {
            yield { commentAdded: event };
          }
        },
      },
    },
  },
  subscription: true, // enables WebSocket support
});

// Client query
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
// Persisted queries: client sends a hash instead of the full query string
// Reduces payload size (especially on mobile), prevents arbitrary queries in production

// 1. At build time, generate query manifest:
// { "sha256:abc123": "query GetUser($id: ID!) { user(id: $id) { name email } }" }

// 2. Server: look up query by hash
app.register(mercurius, {
  schema,
  resolvers,
  persistedQueryProvider: {
    getQueryFromHash: async (hash: string) => {
      return queryManifest[hash]; // lookup from file/Redis
    },
    // Optionally allow unknown hashes (for development):
    // getHashForQuery: sha256,
    // notFoundError: new Error('Unknown query'),
  },
});

// 3. Client sends:
// POST /graphql
// { "extensions": { "persistedQuery": { "version": 1, "sha256Hash": "abc123" } } }
// No "query" field needed — the server resolves it from the hash
```

---

## 5. Common Mistakes & Pitfalls

**Creating DataLoader instances globally:**

```typescript
// WRONG — single global loader caches data across all users/requests
const userLoader = new DataLoader(batchFn);

// CORRECT — new instance per request (created in context factory)
function createContext(req) {
  return {
    userLoader: new DataLoader(batchFn), // fresh cache per request
  };
}
```

**Returning null for errors instead of throwing:**

```typescript
// Wrong — client receives null with no explanation
const resolver = async (_: unknown, { id }: { id: string }) => {
  const user = await db.user.findUnique({ where: { id } });
  return user ?? null; // silently returns null — client has no idea why
};

// Correct — throw to populate the errors array
const resolver = async (_: unknown, { id }: { id: string }) => {
  const user = await db.user.findUnique({ where: { id } });
  if (!user) throw new GraphQLError(`User ${id} not found`, {
    extensions: { code: 'NOT_FOUND' },
  });
  return user;
};
```

**Forgetting that GraphQL queries can be deeply nested:**

A malicious query can request deeply nested data causing exponential DB load:

```graphql
# Malicious — N^depth queries
query {
  users {
    friends {
      friends {
        friends {
          # ... 10 levels deep
        }
      }
    }
  }
}
```

Protection strategies:
- Query depth limiting (`graphql-depth-limit`)
- Query complexity analysis (`graphql-query-complexity`)
- Persisted queries in production (only allow known queries)
- Rate limiting on the HTTP layer

**Exposing internal errors to clients:**

```typescript
// Wrong — stack traces and DB errors exposed
const server = new ApolloServer({
  // default in development exposes all errors
});

// Correct — mask unexpected errors in production
const server = new ApolloServer({
  formatError: (formattedError, error) => {
    // Only expose expected errors (with extensions.code)
    if (formattedError.extensions?.code) {
      return formattedError; // known error, safe to expose
    }
    // Log unexpected errors but return generic message
    console.error('Unexpected GraphQL error:', error);
    return {
      message: 'Internal server error',
      extensions: { code: 'INTERNAL_SERVER_ERROR' },
    };
  },
});
```

---

## 6. When to Use / Not Use

**GraphQL is the right choice when:**
- Multiple clients (iOS, Android, web, third-party) need different data shapes from the same API
- Frontend teams iterate rapidly and frequently change data requirements
- Data is deeply nested with complex relationships (social graphs, content trees)
- You want to replace a proliferation of specialized REST endpoints
- You need real-time subscriptions alongside queries

**Prefer REST when:**
- Simple CRUD with a single client
- HTTP caching is essential (GraphQL POST requests are not cached by default)
- Public API with external consumers who need stability and documentation
- The team is unfamiliar with GraphQL and the project deadline is tight
- File upload/download is the primary use case (REST handles binary better)

> ⚠️ GraphQL is not inherently faster than REST. Without DataLoader and careful resolver design, it can be significantly slower due to the N+1 problem. Performance requires deliberate effort.

---

## 7. Real-World Scenario

**Problem:** A React Native app makes 7 separate REST API calls on the home screen — user profile, recent orders, active subscriptions, notifications, product recommendations, loyalty points, and delivery address. This causes slow initial load (requests are sequential due to dependencies) and wastes data on mobile.

**Solution: Consolidate with a GraphQL API**

```graphql
# Single query replacing 7 REST calls
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

**Backend implementation with DataLoader:**

```typescript
// All resolvers execute in parallel — GraphQL resolves independent fields concurrently
const resolvers = {
  Query: {
    // These four resolve in parallel (no dependency on each other)
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

**Result:** 7 round trips reduced to 1. Page load time drops from 3.2s to 800ms. Mobile data usage reduced by 60% because each client only requests fields it needs.

---

## 8. Interview Questions

**Q1: What is the N+1 problem in GraphQL and how do you solve it?**

The N+1 problem occurs when a resolver fetches related data for each item individually: fetching 100 posts, then making 100 separate DB queries to fetch each post's author. Solution: DataLoader. It batches all individual `.load(id)` calls made during a single event loop tick into a single batch function call — typically one SQL `WHERE id IN (...)` query. It also caches results within the request, so the same ID is never fetched twice. Key rule: create a new DataLoader per request to prevent cross-request cache pollution.

**Q2: How does DataLoader work internally?**

DataLoader uses microtask scheduling: when you call `loader.load(id)`, the ID is added to a pending queue and a Promise is returned. DataLoader schedules the batch function to run in the next microtask (after all synchronous code and current async operations). This means all `.load()` calls from resolvers running in the same tick are collected before the batch executes. The batch function receives all queued IDs, fetches them in one query, and DataLoader resolves each individual Promise with the corresponding value.

**Q3: How do GraphQL subscriptions work?**

Subscriptions are long-lived operations over WebSocket (or SSE). The client sends a subscription query; the server establishes a WebSocket connection and pushes new data whenever the subscribed event occurs. The server typically uses a pub/sub system (Redis, EventEmitter, Kafka) to broadcast events to interested subscribers. Each subscription resolver is an async generator that yields values as events arrive.

**Q4: When would you choose REST over GraphQL?**

REST is better when: HTTP caching is critical (CDN caching doesn't work well with GraphQL POST queries), you're building a public API that needs to be stable and versioned, the team lacks GraphQL experience and time is limited, the API is simple CRUD, or you're dealing with file uploads/downloads. GraphQL wins when: multiple clients need different data shapes, reducing over/under-fetching matters, the domain has complex nested relationships, or you need to rapidly iterate on the frontend without backend changes.

**Q5: What is GraphQL Federation?**

Federation is a way to split a GraphQL schema across multiple independent services (subgraphs) while presenting a unified schema to clients through a gateway. Each service owns its part of the schema and is independently deployable. The `@key` directive marks entity primary keys so the gateway can stitch types together across services. For example, a User service owns `User` type, and an Orders service can extend it with `orders: [Order]` — the gateway resolves the cross-service reference automatically.

**Q6: How do you handle authorization in GraphQL?**

Three main approaches: (1) **In resolvers** — check `context.user.role` at each resolver (flexible but repetitive), (2) **Schema directives** — custom `@auth` or `@hasRole` directives applied declaratively in SDL, (3) **Middleware layer** — `graphql-shield` or similar to define permission rules separate from business logic. Important: avoid relying on schema introspection to hide unauthorized fields — clients with schema access can still request them. Always enforce permissions in resolvers or a rule layer, not just in the schema.

**Q7: What is a persisted query?**

A persisted query replaces the full query string with a hash. At build time, all queries are extracted and stored on the server (in a file or Redis) keyed by their SHA-256 hash. At runtime, the client sends only `{ extensions: { persistedQuery: { sha256Hash: "abc..." } } }`. Benefits: smaller HTTP payloads, enables GET requests (cacheable), and in production you can disable the ability to run arbitrary queries entirely — only pre-registered queries are allowed, which prevents introspection attacks and reduces the attack surface.

---

## 9. Exercises

**Exercise 1: Build a typed GraphQL API**

Create a GraphQL API for a task management app with:
- Types: `Task`, `Project`, `User`
- Queries: `task(id)`, `tasks(projectId, status)`, `project(id)` with tasks
- Mutations: `createTask`, `updateTask(id, input)`, `completeTask(id)`
- Subscriptions: `taskUpdated(projectId)`
- Implement DataLoader for user and project lookups
- Use TypeScript + graphql-codegen to generate typed resolvers

*Hint:* Use `graphql-codegen` with `@graphql-codegen/typescript-resolvers` to generate resolver types from SDL.

**Exercise 2: Fix an N+1 problem**

Given this resolver with an N+1 problem:

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

Refactor it to use DataLoader. Verify with a query logger that fetching 50 posts with authors and comments results in 3 queries total instead of 101+.

*Hint:* You need two DataLoaders: one for users (by ID), one for comments (batched by postId — note the 1:many relationship requires returning arrays).

**Exercise 3: Query complexity analysis**

Implement query complexity analysis that:
- Assigns complexity scores to each field (default 1, list fields = 10)
- Rejects queries with total complexity > 1000
- Returns a `429` status with remaining complexity budget in the response

*Hint:* Use `graphql-query-complexity` library with `fieldExtensionsEstimator` and `simpleEstimator`.

---

## 10. Further Reading

- [GraphQL official specification](https://spec.graphql.org/)
- [GraphQL best practices (graphql.org)](https://graphql.org/learn/best-practices/)
- [DataLoader library (GitHub)](https://github.com/graphql/dataloader)
- [Mercurius — Fastify GraphQL adapter](https://mercurius.dev/)
- [Apollo Server documentation](https://www.apollographql.com/docs/apollo-server/)
- [graphql-shield — permission layer](https://the-guild.dev/graphql/shield)
- [graphql-query-complexity](https://github.com/slicknode/graphql-query-complexity)
- [GraphQL Federation (Apollo)](https://www.apollographql.com/docs/federation/)
- [The Guild — GraphQL tools ecosystem](https://the-guild.dev/)
- Book: *Production Ready GraphQL* by Marc-André Giroux
