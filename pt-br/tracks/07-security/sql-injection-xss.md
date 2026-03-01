# SQL Injection & XSS

## Visão Geral

SQL Injection e Cross-Site Scripting (XSS) são duas das vulnerabilidades mais antigas e bem compreendidas em aplicações web — e ainda assim continuam aparecendo em sistemas em produção todo ano. SQL Injection permite que atacantes manipulem queries de banco de dados. XSS permite que atacantes injetem e executem JavaScript no navegador de vítimas. Ambas são evitáveis com a mesma técnica fundamental: nunca confie em input do usuário, e sempre codifique-o ou parametrize-o antes do uso.

---

## Pré-requisitos

- Conhecimento básico de SQL (SELECT, INSERT, cláusulas WHERE)
- Fundamentos de HTML e JavaScript
- Node.js, TypeScript e Fastify
- Compreensão básica do DOM

---

## Conceitos Fundamentais

### SQL Injection

SQL injection ocorre quando dados fornecidos pelo usuário são concatenados diretamente em uma query SQL. O banco de dados interpreta o input do usuário como código SQL em vez de dados.

**Exemplo clássico:** Se `userId` for `1 OR 1=1`, a query `SELECT * FROM users WHERE id = ${userId}` se torna `SELECT * FROM users WHERE id = 1 OR 1=1`, que retorna todos os usuários.

### XSS (Cross-Site Scripting)

XSS ocorre quando dados fornecidos pelo usuário são renderizados em uma página HTML sem codificação. O navegador interpreta o input do usuário como código HTML ou JavaScript.

Três tipos:
- **Stored XSS**: script malicioso salvo no banco de dados, servido a todos os usuários que visualizam o conteúdo
- **Reflected XSS**: script malicioso na URL, ecoado de volta na resposta
- **DOM-based XSS**: script malicioso injetado no DOM diretamente por JavaScript no lado do cliente

---

## Exemplos Práticos

### SQL Injection — queries parametrizadas

**Vulnerável:**

```typescript
// NUNCA faça isso
const userId = request.params.id;
const result = await db.query(
  `SELECT * FROM users WHERE id = '${userId}'`
);
// userId = "1' OR '1'='1" → retorna todos os usuários
// userId = "1'; DROP TABLE users; --" → destrói dados
```

**Corrigido com Prisma (parametrização automática):**

```typescript
// Prisma sempre usa queries parametrizadas por baixo dos panos
const user = await db.user.findUnique({
  where: { id: request.params.id },
});
// id é passado como parâmetro, nunca interpolado em SQL
```

**Corrigido com SQL raw usando tagged template do Prisma:**

```typescript
// $queryRaw usa tagged template literals — valores são parametrizados
const users = await db.$queryRaw`
  SELECT * FROM users
  WHERE org_id = ${orgId}
  AND created_at > ${since}
`;
```

**Corrigido com `pg` (node-postgres):**

```typescript
import { Pool } from 'pg';
const pool = new Pool();

// Sempre use queries parametrizadas com $1, $2, etc.
const result = await pool.query(
  'SELECT * FROM users WHERE id = $1 AND org_id = $2',
  [userId, orgId]
);
```

### Validação de input com Zod

Queries parametrizadas previnem injection; Zod previne input absurdo de chegar à camada do banco de dados:

```typescript
import { z } from 'zod';

const GetUserSchema = z.object({
  id: z.string().uuid(),
});

const SearchSchema = z.object({
  q: z.string().max(200).trim(),
  page: z.coerce.number().int().positive().max(1000).default(1),
  limit: z.coerce.number().int().positive().max(100).default(20),
});

fastify.get('/users/:id', async (request, reply) => {
  const { id } = GetUserSchema.parse(request.params);
  // id é garantidamente um UUID — nenhuma injection possível
  return db.user.findUnique({ where: { id } });
});

fastify.get('/users/search', async (request, reply) => {
  const { q, page, limit } = SearchSchema.parse(request.query);
  const offset = (page - 1) * limit;

  return db.$queryRaw`
    SELECT * FROM users
    WHERE name ILIKE ${'%' + q + '%'}
    LIMIT ${limit} OFFSET ${offset}
  `;
});
```

### NoSQL Injection (estilo MongoDB)

ORMs que usam queries baseadas em objetos também podem ser injetados se você passar objetos brutos do usuário:

```typescript
// VULNERÁVEL — se body contiver { username: { $gt: "" } }, injection de operador MongoDB
const user = await UserModel.findOne({ username: request.body.username });

// CORRIGIDO — valide e extraia valores escalares
const { username } = z.object({ username: z.string().min(1).max(100) }).parse(request.body);
const user = await UserModel.findOne({ username });
```

---

### XSS — prevenção de stored XSS

Ao aceitar conteúdo do usuário que será exibido como HTML, sanitize com DOMPurify (navegador) ou `sanitize-html` (servidor):

```typescript
import sanitizeHtml from 'sanitize-html';

// Usuário envia um post de blog com conteúdo HTML
fastify.post('/posts', async (request, reply) => {
  const { title, content } = z.object({
    title: z.string().max(200),
    content: z.string().max(50000),
  }).parse(request.body);

  // Sanitiza o HTML para permitir apenas tags e atributos seguros
  const sanitizedContent = sanitizeHtml(content, {
    allowedTags: [
      'h1', 'h2', 'h3', 'p', 'br', 'ul', 'ol', 'li',
      'strong', 'em', 'a', 'code', 'pre', 'blockquote',
    ],
    allowedAttributes: {
      a: ['href', 'title'],
      code: ['class'],
    },
    allowedSchemes: ['http', 'https'], // sem URLs javascript:
  });

  return db.post.create({
    data: { title, content: sanitizedContent, authorId: request.user.sub },
  });
});
```

### XSS — React (lado do cliente)

React escapa HTML em JSX por padrão — `{userContent}` é seguro. O perigo está em contornar isso:

```tsx
// SEGURO — React escapa isso
function Comment({ text }: { text: string }) {
  return <p>{text}</p>;
}

// PERIGOSO — contorna o escape do React
function Comment({ html }: { html: string }) {
  return <p dangerouslySetInnerHTML={{ __html: html }} />;
}

// Uso seguro de dangerouslySetInnerHTML — apenas com conteúdo sanitizado
import DOMPurify from 'dompurify';

function RichContent({ html }: { html: string }) {
  const clean = DOMPurify.sanitize(html);
  return <div dangerouslySetInnerHTML={{ __html: clean }} />;
}
```

### XSS — Content Security Policy

CSP é um mecanismo do navegador que restringe quais scripts, estilos e recursos podem ser carregados. Oferece defesa em profundidade contra XSS — mesmo que um script seja injetado, o CSP pode impedir sua execução.

```typescript
import helmet from '@fastify/helmet';

await fastify.register(helmet, {
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'nonce-{nonce}'"], // nonces para scripts inline
      styleSrc: ["'self'", 'https://fonts.googleapis.com'],
      fontSrc: ["'self'", 'https://fonts.gstatic.com'],
      imgSrc: ["'self'", 'data:', 'https://cdn.example.com'],
      connectSrc: ["'self'", 'https://api.example.com'],
      objectSrc: ["'none'"],
      upgradeInsecureRequests: [],
    },
  },
});
```

Para scripts inline (por exemplo, Next.js), use nonces:

```typescript
// Gera um nonce único por requisição
fastify.addHook('onRequest', async (request, reply) => {
  const nonce = crypto.randomBytes(16).toString('base64');
  request.cspNonce = nonce;
});
```

### XSS — reflected XSS via parâmetros de URL

```typescript
// VULNERÁVEL — ecoando params de URL de volta no HTML sem codificação
fastify.get('/search', async (request, reply) => {
  const { q } = request.query as { q: string };
  return reply.type('text/html').send(`
    <h1>Resultados para: ${q}</h1>
  `);
  // q = "<script>document.cookie</script>" → script executa
});

// CORRIGIDO — nunca interpole input do usuário em HTML; use um template engine com auto-escape
import Handlebars from 'handlebars';

const template = Handlebars.compile(`<h1>Resultados para: {{q}}</h1>`);
// Handlebars codifica {{q}} em HTML automaticamente
fastify.get('/search', async (request, reply) => {
  const { q } = request.query as { q: string };
  return reply.type('text/html').send(template({ q }));
});
```

---

## Padrões Comuns e Boas Práticas

- **Sempre use queries parametrizadas** — nunca interpole input do usuário em SQL com string
- **Valide formato e tipo do input com Zod** em cada fronteira de API
- **Sanitize HTML na entrada** (no servidor) antes de armazenar conteúdo rico
- **Deixe React/framework escapar a saída** — nunca use `dangerouslySetInnerHTML` sem sanitizar
- **Implemente Content Security Policy** como segunda linha de defesa contra XSS
- **Codifique a saída** ao gerar HTML fora de um framework
- **Use query builders de ORM** em vez de SQL raw sempre que possível

---

## Anti-Padrões a Evitar

- Concatenação de strings para construir queries SQL
- Confiar na sanitização de HTML apenas no cliente — também sanitize no servidor antes de armazenar
- Usar `eval()` em input do usuário
- Usar `innerHTML` em JavaScript sem sanitizar
- Depender apenas de validação no frontend — sempre valide no backend
- Pensar que "usuários não conseguem ver este caminho de input" — ferramentas automatizadas conseguem

---

## Debugging e Resolução de Problemas

**"Prisma ainda está vulnerável a injection"**
Prisma é seguro para queries padrão. O risco aparece apenas se você usar `db.$queryRawUnsafe()` com concatenação de strings — use `db.$queryRaw` com tagged template.

**"Meu CSP está bloqueando scripts legítimos"**
Comece com CSP em modo report-only: `Content-Security-Policy-Report-Only`. Isso registra violações sem bloquear. Ajuste a política até não haver violações, depois mude para o modo de aplicação.

**"sanitize-html está removendo tags que preciso"**
Configure as opções `allowedTags` e `allowedAttributes` explicitamente. Os padrões são conservadores. Consulte a [documentação do sanitize-html](https://github.com/apostrophecms/sanitize-html) para a referência completa de configuração.

---

## Cenários do Mundo Real

**Cenário: Endpoint de busca seguro contra injection e XSS**

```typescript
const SearchSchema = z.object({
  q: z.string().trim().max(200).default(''),
  category: z.enum(['articles', 'products', 'users']).default('articles'),
});

fastify.get('/search', async (request, reply) => {
  const { q, category } = SearchSchema.parse(request.query);

  const results = await db.$queryRaw`
    SELECT id, title, excerpt
    FROM ${Prisma.raw(category)}  -- enum validado — seguro para usar como identificador
    WHERE to_tsvector('english', title || ' ' || content) @@ plainto_tsquery(${q})
    LIMIT 20
  `;

  return { q, category, results };
});
```

Nota: `Prisma.raw()` deve ser usado apenas para identificadores (nomes de tabelas, colunas) que foram validados contra um enum — nunca para strings fornecidas pelo usuário.

---

## Leitura Complementar

- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [OWASP XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- [Content Security Policy reference — MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP)
- [sanitize-html](https://github.com/apostrophecms/sanitize-html)
- Track 07: [OWASP Top 10](owasp-top-10.md)

---

## Resumo

SQL injection e XSS compartilham a mesma causa raiz: input do usuário não sanitizado é interpretado como código. SQL injection é prevenida por queries parametrizadas — todo ORM e driver de banco de dados moderno as suporta; nunca há motivo para concatenar input do usuário em SQL. XSS é prevenida pela codificação da saída — React e outros frameworks modernos fazem isso por padrão, mas `dangerouslySetInnerHTML`, interpolação de strings em HTML e template engines sem auto-escape são brechas que devem ser usadas com cuidado. A validação de input com Zod reduz ainda mais a superfície de ataque ao rejeitar inputs malformados antes que cheguem à camada do banco de dados ou do template.
