# Padrões ORM

## 1. O Que É e Por Que Usar

Um ORM (Object-Relational Mapper) faz a ponte entre o mundo orientado a objetos do código da sua aplicação e o mundo relacional do banco de dados. Em vez de escrever strings SQL e mapear manualmente linhas de resultado para objetos, você trabalha com modelos tipados e chamadas de método.

Mas ORMs não são apenas ferramentas de conveniência — eles incorporam padrões de design com implicações reais para testabilidade, manutenibilidade e performance. O padrão Active Record acopla seus modelos de domínio às preocupações do banco. O padrão Data Mapper os separa. O padrão Repository adiciona mais uma camada de abstração. Entender esses padrões ajuda a fazer escolhas deliberadas em vez de aceitar os defaults.

Este arquivo cobre os dois padrões principais (Active Record e Data Mapper), os dois ORMs TypeScript líderes (Prisma e Drizzle), armadilhas comuns de performance (N+1) e o padrão Repository para construir camadas de acesso a dados testáveis.

---

## 2. Conceitos Fundamentais

### Padrão Active Record

No padrão Active Record, um objeto de modelo sabe como salvar a si mesmo no banco. O modelo contém tanto os dados de domínio quanto a lógica de persistência.

```typescript
// Active Record: o modelo É a linha do banco
class User extends ActiveRecord {
  id: string;
  name: string;
  email: string;

  // Comportamento de domínio + persistência misturados
  async save(): Promise<void> {
    await db.query('UPDATE users SET name=$1, email=$2 WHERE id=$3', [this.name, this.email, this.id]);
  }

  static async findByEmail(email: string): Promise<User | null> {
    const row = await db.query('SELECT * FROM users WHERE email=$1', [email]);
    return row ? Object.assign(new User(), row) : null;
  }

  async posts(): Promise<Post[]> {
    return Post.findByUserId(this.id); // também faz query no banco
  }
}

// Uso
const user = await User.findByEmail('alice@example.com');
user.name = 'Alice Atualizada';
await user.save();
```

**Prós:**
- Simples e rápido de escrever para apps CRUD-heavy.
- Modelo e persistência ficam juntos — fácil de encontrar.
- Bom para protótipos rápidos.

**Contras:**
- Difícil de testar unitariamente — cada método do modelo requer uma conexão real com o banco.
- Lógica de negócio acaba na camada errada (modelos, não serviços).
- Modelos ficam "gordos" — misturando responsabilidades.
- Difícil de trocar o mecanismo de persistência.

---

### Padrão Data Mapper

No padrão Data Mapper, objetos de domínio são estruturas de dados puras (POJOs) sem conhecimento do banco. Uma camada "mapper" separada cuida de toda a persistência.

```typescript
// Objeto de domínio puro — não sabe nada sobre banco de dados
class User {
  constructor(
    public readonly id: string,
    public name: string,
    public email: string,
    public readonly createdAt: Date
  ) {}

  // Comportamento de domínio puro — sem queries no banco
  rename(newName: string): void {
    if (!newName.trim()) throw new Error('Nome não pode ser vazio');
    this.name = newName;
  }

  isAdmin(): boolean {
    return this.email.endsWith('@internal.example.com');
  }
}

// Mapper: conhece tanto o objeto de domínio quanto o banco de dados
class UserMapper {
  constructor(private db: Pool) {}

  async findById(id: string): Promise<User | null> {
    const row = await this.db.query('SELECT * FROM users WHERE id=$1', [id]);
    if (!row.rows[0]) return null;
    return this.toDomain(row.rows[0]);
  }

  async save(user: User): Promise<void> {
    await this.db.query(
      'INSERT INTO users (id, name, email, created_at) VALUES ($1,$2,$3,$4) ON CONFLICT (id) DO UPDATE SET name=$2, email=$3',
      [user.id, user.name, user.email, user.createdAt]
    );
  }

  private toDomain(row: UserRow): User {
    return new User(row.id, row.name, row.email, row.created_at);
  }
}
```

**Prós:**
- Objetos de domínio são puros — totalmente testáveis sem banco.
- Separação clara de responsabilidades.
- Lógica de domínio isolada de mudanças de infraestrutura.
- Mais fácil de trocar ORMs ou banco de dados.

**Contras:**
- Mais boilerplate.
- Classes de mapper podem crescer complexas para modelos de domínio ricos.
- Excessivo para aplicações CRUD simples.

---

## 3. Prisma

Prisma é um ORM TypeScript-first que adota uma abordagem schema-first. Você define seu modelo de dados em `schema.prisma` e o Prisma gera um client totalmente tipado.

### Definição de Schema

```prisma
// schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(uuid())
  name      String
  email     String   @unique
  role      Role     @default(USER)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  posts     Post[]
  profile   Profile?
}

enum Role {
  USER
  ADMIN
  MODERATOR
}

model Post {
  id          String    @id @default(uuid())
  title       String
  body        String
  published   Boolean   @default(false)
  author      User      @relation(fields: [authorId], references: [id])
  authorId    String
  tags        Tag[]
  createdAt   DateTime  @default(now())

  @@index([authorId, createdAt(sort: Desc)])
}

model Tag {
  id    String @id @default(uuid())
  name  String @unique
  posts Post[]
}

model Profile {
  id     String @id @default(uuid())
  bio    String?
  user   User   @relation(fields: [userId], references: [id])
  userId String @unique
}
```

### Queries Prisma

```typescript
import { PrismaClient, Prisma } from '@prisma/client';
const prisma = new PrismaClient();

// findUnique: correspondência exata (type-safe — retorna T | null)
const user = await prisma.user.findUnique({
  where: { email: 'alice@example.com' },
});

// findFirst: primeira linha correspondente
const recentPost = await prisma.post.findFirst({
  where: { published: true },
  orderBy: { createdAt: 'desc' },
});

// findMany: lista com filtragem + ordenação + paginação
const posts = await prisma.post.findMany({
  where: {
    published: true,
    author: {
      role: { in: ['USER', 'MODERATOR'] },
    },
  },
  orderBy: [{ createdAt: 'desc' }],
  take: 20,
  skip: 0,
  // Cursor pagination:
  // cursor: { id: lastSeenId },
  // skip: 1, // pula o item do cursor
});

// create
const newUser = await prisma.user.create({
  data: {
    name: 'Alice',
    email: 'alice@example.com',
    posts: {
      create: [{ title: 'Primeiro Post', body: 'Olá mundo' }], // create aninhado
    },
  },
  include: { posts: true }, // retorna com posts
});

// update
const updated = await prisma.user.update({
  where: { id: userId },
  data: { name: 'Alice Atualizada' },
});

// upsert
const upserted = await prisma.user.upsert({
  where: { email: 'alice@example.com' },
  create: { name: 'Alice', email: 'alice@example.com' },
  update: { name: 'Alice' },
});

// delete
await prisma.user.delete({ where: { id: userId } });

// deleteMany
await prisma.post.deleteMany({
  where: { authorId: userId, published: false },
});

// count
const count = await prisma.post.count({
  where: { authorId: userId },
});

// aggregate
const stats = await prisma.order.aggregate({
  where: { status: 'COMPLETED' },
  _sum: { total: true },
  _avg: { total: true },
  _count: true,
});
```

### select vs include — A Armadilha do N+1

```typescript
// include: eager loading de registros relacionados — dispara queries adicionais (JOINs)
const postsWithAuthors = await prisma.post.findMany({
  include: {
    author: true,           // JOIN users
    tags: true,             // JOIN tags
    _count: {               // subquery para contagens
      select: { comments: true },
    },
  },
  take: 50,
});
// Prisma executa: 1 query para posts + 1 para autores + 1 para tags
// Ainda muito melhor que o N+1 que você teria com o padrão de resolver sem DataLoader

// select: escolhe campos específicos — NÃO dispara queries extras para campos escalares
const postSummaries = await prisma.post.findMany({
  select: {
    id: true,
    title: true,
    createdAt: true,
    author: {
      select: { name: true, email: true }, // select aninhado para relação
    },
  },
});

// PERIGO: a armadilha do N+1 em um loop
const posts = await prisma.post.findMany({ take: 100 });
for (const post of posts) {
  // Isso é N+1! 100 queries extras
  const author = await prisma.user.findUnique({ where: { id: post.authorId } });
}

// CORREÇÃO: use include ou select
const posts = await prisma.post.findMany({
  take: 100,
  include: { author: { select: { name: true } } },
});
```

> Nunca chame Prisma dentro de um loop sem ter carregado a relação antes. Sempre carregue dados relacionados via `include` ou `select` na query inicial.

### Transações Prisma

```typescript
// $transaction: forma array (execução paralela, sem dependência sequencial)
const [newPost, updatedUser] = await prisma.$transaction([
  prisma.post.create({ data: { title: 'Novo Post', body: '...', authorId } }),
  prisma.user.update({ where: { id: authorId }, data: { postCount: { increment: 1 } } }),
]);

// $transaction: interativa (sequencial, acessa resultados de operações anteriores)
const result = await prisma.$transaction(async (tx) => {
  const user = await tx.user.findUnique({ where: { id: userId } });
  if (!user) throw new Error('Usuário não encontrado');

  if (user.credits < requiredCredits) throw new Error('Créditos insuficientes');

  await tx.user.update({
    where: { id: userId },
    data: { credits: { decrement: requiredCredits } },
  });

  const order = await tx.order.create({
    data: { userId, amount: requiredCredits, status: 'PENDING' },
  });

  return order;
});
// Se qualquer passo lança, a transação inteira é revertida (rollback)

// Com nível de isolamento
await prisma.$transaction(async (tx) => { /* ... */ }, {
  isolationLevel: Prisma.TransactionIsolationLevel.Serializable,
  timeout: 5000, // ms
});
```

### Queries Raw

```typescript
// $queryRaw: use para queries complexas que o Prisma não consegue expressar
const result = await prisma.$queryRaw<Array<{ id: string; score: number }>>`
  SELECT u.id, COUNT(p.id)::int as score
  FROM users u
  LEFT JOIN posts p ON p.author_id = u.id
  WHERE u.created_at > ${thirtyDaysAgo}
  GROUP BY u.id
  ORDER BY score DESC
  LIMIT 10
`;

// $executeRaw: para INSERT/UPDATE/DELETE onde você não precisa dos resultados
const affected = await prisma.$executeRaw`
  UPDATE posts SET views = views + 1 WHERE id = ${postId}
`;
```

---

## 4. Drizzle

Drizzle é um ORM TypeScript leve que define schema como código TypeScript e gera queries type-safe.

```typescript
// schema.ts
import { pgTable, uuid, text, boolean, timestamp, index } from 'drizzle-orm/pg-core';
import { relations } from 'drizzle-orm';

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: text('name').notNull(),
  email: text('email').notNull().unique(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});

export const posts = pgTable('posts', {
  id: uuid('id').primaryKey().defaultRandom(),
  title: text('title').notNull(),
  body: text('body').notNull(),
  published: boolean('published').default(false).notNull(),
  authorId: uuid('author_id').notNull().references(() => users.id),
  createdAt: timestamp('created_at').defaultNow().notNull(),
}, (table) => ({
  authorIdx: index('posts_author_idx').on(table.authorId, table.createdAt),
}));

export const usersRelations = relations(users, ({ many }) => ({
  posts: many(posts),
}));

export const postsRelations = relations(posts, ({ one }) => ({
  author: one(users, { fields: [posts.authorId], references: [users.id] }),
}));
```

```typescript
// queries.ts
import { drizzle } from 'drizzle-orm/node-postgres';
import { eq, desc, and, lt, sql } from 'drizzle-orm';
import { Pool } from 'pg';
import * as schema from './schema';

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const db = drizzle(pool, { schema });

// Select com where e ordenação
const publishedPosts = await db
  .select({
    id: schema.posts.id,
    title: schema.posts.title,
    authorName: schema.users.name,
  })
  .from(schema.posts)
  .leftJoin(schema.users, eq(schema.posts.authorId, schema.users.id))
  .where(and(
    eq(schema.posts.published, true),
    lt(schema.posts.createdAt, new Date())
  ))
  .orderBy(desc(schema.posts.createdAt))
  .limit(20);

// Insert
const [newUser] = await db
  .insert(schema.users)
  .values({ name: 'Alice', email: 'alice@example.com' })
  .returning();

// Update
await db
  .update(schema.posts)
  .set({ published: true })
  .where(eq(schema.posts.id, postId));

// Delete
await db.delete(schema.posts).where(eq(schema.posts.authorId, userId));

// Transações
await db.transaction(async (tx) => {
  await tx.update(schema.users)
    .set({ credits: sql`credits - ${amount}` })
    .where(eq(schema.users.id, userId));

  await tx.insert(schema.orders)
    .values({ userId, amount });
});
```

> Drizzle é "SQL-like" — se você conhece SQL, a API de query do Drizzle é imediatamente legível. É significativamente mais leve que o Prisma e evita o binário do query engine do Prisma. Escolha Drizzle quando quiser controle fino sobre o SQL com segurança TypeScript; escolha Prisma quando quiser uma API de nível mais alto e migrações automáticas.

---

## 5. O Padrão Repository

O padrão repository define uma interface para acesso a dados e fornece uma implementação concreta. Isso torna a camada de acesso a dados substituível e, crucialmente, testável com implementações em memória.

```typescript
// 1. Tipo de domínio (puro)
interface User {
  id: string;
  name: string;
  email: string;
  role: 'USER' | 'ADMIN';
  createdAt: Date;
}

// 2. Interface do repository — o contrato
interface UserRepository {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  list(options: { limit: number; cursor?: string; role?: User['role'] }): Promise<User[]>;
  save(user: User): Promise<User>;
  delete(id: string): Promise<boolean>;
}

// 3. Implementação Prisma
class PrismaUserRepository implements UserRepository {
  constructor(private prisma: PrismaClient) {}

  async findById(id: string): Promise<User | null> {
    return this.prisma.user.findUnique({ where: { id } });
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.prisma.user.findUnique({ where: { email } });
  }

  async list({ limit, cursor, role }: { limit: number; cursor?: string; role?: User['role'] }): Promise<User[]> {
    return this.prisma.user.findMany({
      where: { role: role ?? undefined },
      take: limit,
      ...(cursor && { cursor: { id: cursor }, skip: 1 }),
      orderBy: { createdAt: 'desc' },
    });
  }

  async save(user: User): Promise<User> {
    return this.prisma.user.upsert({
      where: { id: user.id },
      create: user,
      update: { name: user.name, email: user.email, role: user.role },
    });
  }

  async delete(id: string): Promise<boolean> {
    try {
      await this.prisma.user.delete({ where: { id } });
      return true;
    } catch {
      return false;
    }
  }
}

// 4. Implementação em memória para testes — rápida, sem banco necessário
class InMemoryUserRepository implements UserRepository {
  private users = new Map<string, User>();

  async findById(id: string): Promise<User | null> {
    return this.users.get(id) ?? null;
  }

  async findByEmail(email: string): Promise<User | null> {
    return Array.from(this.users.values()).find((u) => u.email === email) ?? null;
  }

  async list({ limit, cursor, role }: { limit: number; cursor?: string; role?: User['role'] }): Promise<User[]> {
    let results = Array.from(this.users.values());
    if (role) results = results.filter((u) => u.role === role);
    if (cursor) {
      const idx = results.findIndex((u) => u.id === cursor);
      results = results.slice(idx + 1);
    }
    return results.slice(0, limit);
  }

  async save(user: User): Promise<User> {
    this.users.set(user.id, { ...user });
    return user;
  }

  async delete(id: string): Promise<boolean> {
    return this.users.delete(id);
  }
}

// 5. Serviço usa a interface — não a implementação concreta
class UserService {
  constructor(private userRepo: UserRepository) {}

  async getUser(id: string): Promise<User> {
    const user = await this.userRepo.findById(id);
    if (!user) throw new NotFoundError(`Usuário ${id} não encontrado`);
    return user;
  }

  async promoteToAdmin(id: string): Promise<User> {
    const user = await this.getUser(id);
    return this.userRepo.save({ ...user, role: 'ADMIN' });
  }
}
```

**Testando sem banco de dados:**

```typescript
// tests/user-service.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { UserService } from '../src/services/user.service';
import { InMemoryUserRepository } from '../src/repositories/in-memory/user.repository';

describe('UserService', () => {
  let repo: InMemoryUserRepository;
  let service: UserService;

  beforeEach(() => {
    repo = new InMemoryUserRepository();
    service = new UserService(repo);
  });

  it('deve promover um usuário para admin', async () => {
    const user = await repo.save({
      id: '1', name: 'Alice', email: 'alice@test.com',
      role: 'USER', createdAt: new Date(),
    });

    const promoted = await service.promoteToAdmin(user.id);

    expect(promoted.role).toBe('ADMIN');
  });

  it('deve lançar NotFoundError para usuário inexistente', async () => {
    await expect(service.getUser('inexistente'))
      .rejects.toThrow('Usuário inexistente não encontrado');
  });
});
// Esses testes rodam em < 10ms sem conexão com banco
```

---

## 6. Drizzle vs Prisma: Quando Escolher Cada Um

| Fator | Prisma | Drizzle |
|-------|--------|---------|
| Type safety | Excelente (tipos gerados) | Excelente (tipos inferidos) |
| API de query | Alto nível, estilo ORM | Estilo SQL, mais baixo nível |
| Tamanho do bundle | Grande (binário do query engine) | Mínimo |
| Serverless/Edge | Funciona (Prisma Accelerate) | Suporte nativo |
| Migrações | Automático (prisma migrate) | SQL manual + drizzle-kit |
| SQL raw | `$queryRaw` (tagged template) | Fácil (escape hatch) |
| Curva de aprendizado | Menor para devs JS | Menor para devs SQL |
| Queries complexas | Às vezes desajeitado | Natural (familiaridade SQL) |
| Maturidade do ecossistema | Mais maduro, comunidade maior | Crescendo rápido |

---

## 7. Erros Comuns e Armadilhas

**Chamar Prisma em um loop (N+1):**

```typescript
// ERRADO: 100 queries no banco para 100 posts
const posts = await prisma.post.findMany({ take: 100 });
const enriched = await Promise.all(
  posts.map(async (p) => ({
    ...p,
    author: await prisma.user.findUnique({ where: { id: p.authorId } }),
  }))
);

// CORRETO: 1 query com include
const posts = await prisma.post.findMany({
  take: 100,
  include: { author: { select: { id: true, name: true } } },
});
```

**Não liberar conexões do pool em transações:**

```typescript
// ERRADO com pg raw: vazamento de conexão se ocorrer erro antes do release
const client = await pool.connect();
await client.query('BEGIN');
const result = await client.query('SELECT ...');
// Erro aqui → client nunca é liberado → pool esgotado
client.release();

// CORRETO: sempre use try/finally
const client = await pool.connect();
try {
  await client.query('BEGIN');
  // ...
  await client.query('COMMIT');
} catch (err) {
  await client.query('ROLLBACK');
  throw err;
} finally {
  client.release(); // sempre executa
}
```

**Prisma client não sendo um singleton:**

```typescript
// ERRADO: cria um novo pool de conexão a cada import em ambientes com hot-reload
// (ex: servidor de desenvolvimento do Next.js)
export const prisma = new PrismaClient();

// CORRETO: padrão singleton
import { PrismaClient } from '@prisma/client';

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient };

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: process.env.NODE_ENV === 'development' ? ['query'] : [],
  });

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma;
}
```

---

## 8. Quando Usar Cada Abordagem

**Active Record:** Scripts pequenos, protótipos rápidos, apps CRUD simples sem lógica de domínio complexa. Quando simplicidade > testabilidade.

**Data Mapper (Prisma/Drizzle como mapper):** A maioria das aplicações em produção. O ORM gera o mapper para você. Use o padrão Repository por cima para testabilidade.

**Padrão Repository:** Qualquer aplicação onde testes unitários importam. Qualquer aplicação que possa mudar de banco. Qualquer aplicação usando Domain-Driven Design. Obrigatório quando você quer escrever testes que rodam em < 1 segundo sem banco de dados.

**SQL raw (via `$queryRaw` ou `pg` diretamente):** Agregações complexas com window functions, CTEs recursivas, recursos específicos do banco, ou queries críticas de performance onde a abstração ORM gera SQL subótimo.

---

## 9. Cenário Real

**Problema:** O serviço de usuários de uma startup cresceu para 30 testes, mas todos requerem uma conexão PostgreSQL ativa. A suite de testes leva 4 minutos para rodar. O CI falha aleatoriamente por limites de conexão ao banco sendo excedidos.

**Solução: Padrão Repository + implementação InMemory**

```typescript
// Antes: serviço importa Prisma diretamente
class UserService {
  async findActive(): Promise<User[]> {
    return prisma.user.findMany({ where: { status: 'ACTIVE' } }); // dependência rígida!
  }
}

// Depois: serviço depende da interface
class UserService {
  constructor(private users: UserRepository) {}

  async findActive(): Promise<User[]> {
    return this.users.list({ status: 'ACTIVE' });
  }
}

// Testes: usa repo em memória
const service = new UserService(new InMemoryUserRepository());
// Produção: usa repo Prisma (conectado na composition root / container de DI)
const service = new UserService(new PrismaUserRepository(prisma));
```

**Resultado:** 30 testes unitários rodam em 180ms em vez de 4 minutos. Confiabilidade do CI vai para 100%. Testes de integração (com banco real) ainda existem mas são uma suite separada rodada com menos frequência.

---

## 10. Perguntas de Entrevista

**P1: Qual a diferença entre os padrões Active Record e Data Mapper?**

Active Record: o objeto de modelo cuida tanto dos dados/comportamento de domínio quanto da persistência. O modelo sabe como salvar/carregar a si mesmo (`user.save()`, `User.find()`). Simples mas acopla lógica de domínio à infraestrutura do banco — difícil de testar unitariamente. Data Mapper: o objeto de domínio é um POJO puro sem conhecimento do banco. Uma classe mapper separada cuida da persistência. O objeto de domínio pode ser instanciado e testado sem banco de dados. O Prisma segue a abordagem Data Mapper — `PrismaClient` é o mapper, seus tipos de domínio são separados.

**P2: O que é o problema N+1 em ORMs e como corrigi-lo no Prisma?**

N+1 ocorre quando você carrega N registros e então faz 1 query adicional por registro para carregar dados relacionados. Exemplo: carrega 100 posts (1 query), depois em um loop carrega o autor de cada post (100 queries) = 101 queries. No Prisma, corrija com `include` na query inicial: `prisma.post.findMany({ include: { author: true } })` — o Prisma busca autores em uma segunda query em batch (não 100 queries individuais). Alternativamente use `select` para carregar apenas os campos necessários. Nunca faça query dentro de um loop.

**P3: Por que usar o padrão repository por cima de um ORM como o Prisma?**

O padrão repository adiciona uma interface de abstração entre sua camada de serviço e o ORM. Benefícios: (1) Testabilidade — escreva um `InMemoryUserRepository` que o serviço usa nos testes, sem banco necessário, testes rodam em milissegundos. (2) Substituibilidade — troca de Prisma para Drizzle trocando a implementação concreta do repository, sem mudanças nos serviços. (3) Ocultamento de queries — código do serviço fica limpo de detalhes de query. (4) Separação de responsabilidades — lógica de acesso a dados fica em um único lugar.

**P4: Qual a diferença entre `select` e `include` no Prisma?**

`include` carrega uma relação (ex: autor de um post) — o Prisma emite uma query adicional (ou JOIN) para buscar os registros relacionados. `select` especifica quais campos escalares retornar e pode incluir selects aninhados para relações. Ambos podem ser usados para evitar over-fetching. A diferença crítica: `include: { author: true }` retorna todos os campos do autor relacionado; `select: { author: { select: { name: true } } }` retorna apenas o nome do autor. Para performance: sempre selecione apenas o que precisa; nunca `include` uma relação inteira se precisar de apenas um campo dela.

**P5: Como funcionam as transações do Prisma e quais são os dois modos?**

Prisma oferece duas APIs de transação: (1) **Modo array** — passa um array de operações Prisma; elas rodam em paralelo mas todas têm sucesso ou falham juntas. Limitado a operações independentes (não é possível usar o resultado de uma como entrada para outra). (2) **Transações interativas** — passa um callback async que recebe um client `tx`; operações rodam sequencialmente dentro de uma única transação. Use `tx` em vez de `prisma` para todas as operações dentro do callback. Se o callback lança, toda a transação é revertida. Use a forma interativa sempre que operações dependem umas das outras ou precisam ler-e-escrever.

**P6: Como você testa código que usa um ORM?**

Três estratégias: (1) **Repository em memória** — implemente a interface do repository com um Map em memória. Testes rodam sem banco, milissegundos por teste. Melhor para testes unitários de lógica de negócio. (2) **Banco de testes** — sobe uma instância PostgreSQL real (via Docker Compose ou `@testcontainers/postgresql`) e roda migrações antes dos testes. Melhor para testes de integração. (3) **Mock do client** — usa `vitest.mock()` para mockar `PrismaClient`. Frágil e acopla testes a detalhes de implementação; prefira o repository em memória.

**P7: Quando você escolheria Drizzle em vez de Prisma?**

Escolha Drizzle quando: você precisa de bundle size mínimo (importante para deploys Edge/serverless onde o binário do query engine do Prisma é um problema), você prefere escrever queries estilo SQL em vez de uma API ORM de alto nível, precisa de controle fino sobre o SQL gerado, ou quer o schema definido em código TypeScript (melhor para monorepo/cenários de compartilhamento de código). Escolha Prisma quando: você prefere migrações auto-geradas, quer o ecossistema mais rico e as ferramentas mais maduras, ou seu time está mais confortável com a API de query de mais alto nível.

---

## 11. Exercícios

**Exercício 1: BaseRepository genérico**

Implemente uma classe genérica `BaseRepository<T, ID>` em TypeScript que:
- Tem métodos abstratos: `findById`, `findAll`, `save`, `delete`
- Fornece uma implementação concreta em memória como `InMemoryRepository<T, ID>`
- Usa um `Map<ID, T>` como backing store
- Suporta `findAll` com filtro de predicado opcional

Então implemente `PostRepository extends InMemoryRepository<Post, string>` que adiciona `findByAuthorId(authorId: string): Promise<Post[]>`.

*Dica:* A constraint genérica: `T extends { id: ID }`. Use generics TypeScript para tornar a classe base reutilizável.

**Exercício 2: Detector de N+1**

Instrumente o Prisma com um middleware `$use` que:
- Rastreia todas as chamadas `findUnique` dentro de um único contexto de requisição
- Avisa (loga) se o mesmo model é consultado individualmente mais de 5 vezes em uma única requisição (indicando N+1)
- Inclui o stack trace da primeira chamada no aviso

*Dica:* Use `AsyncLocalStorage` para passar contexto de requisição pelo middleware do Prisma.

**Exercício 3: CRUD completo com Repository + Validação**

Construa um `PostService` que:
- Usa a interface `PostRepository` (não Prisma diretamente)
- `createPost(authorId, data)`: valida título (mínimo 5 chars), verifica se o autor existe, cria o post
- `updatePost(id, userId, data)`: verifica se o post existe, verifica se userId é o autor, atualiza
- `deletePost(id, userId)`: verifica propriedade, soft-delete (define deletedAt)
- Escreve testes unitários usando `InMemoryPostRepository` (sem banco necessário)
- Escreve um teste de integração usando Prisma com banco de testes

---

## 12. Leituras Complementares

- [Documentação Prisma](https://www.prisma.io/docs)
- [Documentação Drizzle ORM](https://orm.drizzle.team/)
- [Prisma — guia de padrão repository](https://www.prisma.io/docs/guides/testing/unit-testing)
- [Martin Fowler — Active Record](https://www.martinfowler.com/eaaCatalog/activeRecord.html)
- [Martin Fowler — Data Mapper](https://www.martinfowler.com/eaaCatalog/dataMapper.html)
- [Martin Fowler — Repository](https://www.martinfowler.com/eaaCatalog/repository.html)
- Livro: *Patterns of Enterprise Application Architecture* por Martin Fowler (a fonte desses padrões)
- Livro: *Domain-Driven Design* por Eric Evans (por que Data Mapper e Repository importam para domínios ricos)
