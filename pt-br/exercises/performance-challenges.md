# Desafios de Performance

> Exercícios práticos de engenharia de performance cobrindo profiling, otimização de banco de dados, cache, renderização frontend e tuning de infraestrutura. Cada exercício reflete um problema real de performance em produção. Stack assumida: Node.js 20+, TypeScript, Fastify, Prisma, React, Redis. Nível alvo: intermediário a avançado.

---

## Exercício 1 — Perfil de um Endpoint Lento com clinic.js (Médio)

**Cenário:** Um endpoint `GET /reports/summary` leva 3–8 segundos. Você não sabe por quê. Use clinic.js para fazer o perfil e identificar o gargalo.

**Requisitos:**
- Instale `clinic` globalmente e faça o perfil do endpoint sob carga usando `clinic doctor`.
- Identifique se o gargalo é: computação intensiva de CPU, espera de I/O, bloqueio do event loop ou pressão de memória.
- Produza um relatório de flamegraph ou bubbleprof do `clinic.js` e anote os 3 principais hotspots.
- Corrija o gargalo identificado e meça o tempo de resposta antes e depois.
- Documente a causa raiz e a correção em um breve write-up (3–5 frases).

**Critérios de Aceite:**
- [ ] `clinic doctor -- node dist/server.js` roda sem erros e produz um relatório HTML.
- [ ] O relatório HTML identifica claramente o tipo de gargalo (I/O, CPU, etc.).
- [ ] A correção reduz o tempo de resposta p99 em pelo menos 50%.
- [ ] O write-up nomeia a função/módulo específico responsável.
- [ ] Carga durante o profiling usa `autocannon -c 10 -d 30 http://localhost:3000/reports/summary`.

**Dicas:**
1. Culpados comuns: `JSON.parse` síncrono em payloads grandes, `await` faltando em chamadas de banco de dados (famintos do event loop), `fs.readFileSync` bloqueante em caminhos quentes, queries N+1.
2. `clinic flame` fornece um flamegraph de CPU — melhor para problemas com muita CPU.
3. `clinic bubbleprof` visualiza operações assíncronas — melhor para esperas de I/O.
4. Após encontrar o hotspot, verifique: a computação pode ser movida offline (pré-computada)? O I/O pode ser paralelizado com `Promise.all`?

---

## Exercício 2 — Corrigir Queries N+1 em um Schema Prisma (Médio)

**Cenário:** Um endpoint `GET /posts` lista posts de blog com seus autores. Cada carregamento de página dispara 21 queries SQL (1 para posts + 1 por post para o autor). Corrija isso.

**Código fornecido (problema N+1):**
```typescript
async function getPosts(page: number) {
  const posts = await prisma.post.findMany({
    skip: (page - 1) * 20,
    take: 20,
    orderBy: { createdAt: 'desc' },
  });

  return Promise.all(
    posts.map(async post => ({
      ...post,
      author: await prisma.user.findUnique({ where: { id: post.authorId } }),
    }))
  );
}
```

**Requisitos:**
- Reescreva `getPosts` para emitir no máximo 2 queries SQL para qualquer tamanho de página.
- Não altere o tipo de retorno da função.
- Adicione logging de queries do Prisma para contar queries antes e depois da correção.
- Escreva um teste que verifica que a versão corrigida chama `prisma.user.findUnique` zero vezes.

**Critérios de Aceite:**
- [ ] Log de queries do Prisma mostra 1–2 queries por chamada (não 21).
- [ ] O shape da resposta é idêntico ao original (array de posts com objetos `author` aninhados).
- [ ] A correção usa `include` ou `select` do Prisma — sem SQL bruto.
- [ ] Um teste mocka `prisma` e verifica que `findUnique` nunca é chamado.
- [ ] `EXPLAIN ANALYZE` mostra nenhum sequential scan na tabela `users` por post.

**Dicas:**
1. Correção: use `include: { author: true }` em `findMany` — Prisma emite uma única query com JOIN.
2. Alternativamente, use `select` para buscar apenas os campos necessários: `select: { id: true, title: true, author: { select: { id: true, name: true } } }`.
3. Logging de queries do Prisma: `new PrismaClient({ log: ['query'] })`. Conte eventos `query` nos testes.
4. Para conjuntos de resultados muito grandes, `include` ainda pode ser lento — considere uma query SQL bruta com JOIN manual para endpoints críticos de performance.

---

## Exercício 3 — Implementar Paginação Baseada em Cursor (Médio)

**Cenário:** Sua API usa paginação com offset (`LIMIT x OFFSET y`). Na página 500 com 20 itens/página, o banco de dados escaneia 10.000 linhas para pular. Substitua por paginação baseada em cursor.

**Requisitos:**
- Substitua `GET /posts?page=N&limit=20` por `GET /posts?cursor=<id>&limit=20`.
- O cursor é o `id` do último item retornado na página anterior (opaco para o cliente — codifique em base64).
- Formato da resposta: `{ data: Post[], nextCursor: string | null }`. `nextCursor` é `null` na última página.
- A query deve usar `WHERE id > cursor` — não `OFFSET`.
- Paginação para trás não é necessária.

**Critérios de Aceite:**
- [ ] `GET /posts` (sem cursor) retorna os primeiros 20 posts e um `nextCursor`.
- [ ] `GET /posts?cursor=<nextCursor>` retorna os próximos 20 posts com um `nextCursor` diferente.
- [ ] A última página retorna `nextCursor: null`.
- [ ] `EXPLAIN ANALYZE` mostra um index scan usando a chave primária — sem sequential scans.
- [ ] Um teste verifica que paginar por todos os 100 itens (5 páginas × 20) retorna cada item exatamente uma vez.

**Dicas:**
1. Query: `WHERE id > decodeCursor(cursor) ORDER BY id ASC LIMIT limit + 1`. Busque `limit + 1` linhas — se você receber `limit + 1`, há uma próxima página; defina `nextCursor` como o `id` do item `limit + 1`, então retorne apenas os primeiros `limit` itens.
2. Codificação do cursor: `Buffer.from(String(id)).toString('base64')`. Decodificação: `parseInt(Buffer.from(cursor, 'base64').toString('utf8'))`.
3. O `ORDER BY id` deve corresponder ao índice — adicione `CREATE INDEX ON posts(id)` se não for a chave primária.
4. Cursor ausente (primeira página): `WHERE id > 0` ou omita a cláusula `WHERE`.

---

## Exercício 4 — Adicionar Cache Redis com Padrão Cache-Aside (Médio)

**Cenário:** Um endpoint `GET /users/:id` acessa o PostgreSQL em cada requisição. Os dados do usuário raramente mudam (no máximo uma vez por dia). Adicione um cache Redis usando o padrão cache-aside.

**Requisitos:**
- Em cache miss: busque do PostgreSQL, armazene no Redis com TTL de 5 minutos, retorne ao cliente.
- Em cache hit: retorne do Redis sem tocar no PostgreSQL.
- Na atualização do usuário (`PUT /users/:id`): invalide a chave de cache imediatamente.
- Formato da chave de cache: `user:v1:<id>`. Versione a chave para invalidar todas as entradas com formato antigo incrementando `v1`.
- Adicione header de resposta `X-Cache: HIT` ou `X-Cache: MISS`.

**Critérios de Aceite:**
- [ ] Segunda requisição para o mesmo usuário retorna `X-Cache: HIT`.
- [ ] Após `PUT /users/:id`, o próximo `GET /users/:id` retorna `X-Cache: MISS` (cache foi invalidado).
- [ ] O PostgreSQL nunca é consultado quando o cache está quente (verificado por contador de queries de banco de dados).
- [ ] A chave Redis expira após 5 minutos (verificado checando o TTL após o set).
- [ ] Serialização/desserialização JSON é tratada transparentemente — quem chama recebe um objeto `User` tipado, não uma string bruta.

**Dicas:**
1. Cache-aside: `const cached = await redis.get(key); if (cached) return JSON.parse(cached); const user = await db.findUser(id); await redis.setex(key, 300, JSON.stringify(user)); return user;`.
2. Desserialização tipada: crie uma função `deserializeUser(raw: string): User` que faz parse e valida com Zod.
3. Na atualização: `await redis.del(`user:v1:${id}`)` imediatamente após a atualização no banco de dados.
4. Teste a invalidação: `await userService.updateUser(id, data); const res = await app.inject({ url: `/users/${id}` }); expect(res.headers['x-cache']).toBe('MISS');`.

---

## Exercício 5 — Otimizar um Componente React com memo/useMemo (Médio)

**Cenário:** Um componente `ProductList` re-renderiza em toda mudança de estado do pai, mesmo quando seus próprios dados não mudaram. Faça o perfil com React DevTools e otimize-o.

**Componente fornecido (simplificado):**
```tsx
function ProductList({ products, onAddToCart, discountRate }) {
  const discountedProducts = products.map(p => ({
    ...p,
    finalPrice: p.price * (1 - discountRate),
  }));

  return (
    <ul>
      {discountedProducts.map(product => (
        <ProductCard key={product.id} product={product} onAdd={() => onAddToCart(product.id)} />
      ))}
    </ul>
  );
}
```

**Requisitos:**
- Use o Profiler do React DevTools para medir renderizações antes da otimização.
- Aplique `useMemo` para memoizar a computação de `discountedProducts`.
- Envolva `ProductCard` com `React.memo` para que ele pule re-renderizações quando suas props não mudam.
- Estabilize o callback `onAdd` com `useCallback` para prevenir `ProductCard` de re-renderizar por uma nova referência de função.
- Após a otimização, adicionar um item ao carrinho não deve re-renderizar componentes `ProductCard` de outros produtos.

**Critérios de Aceite:**
- [ ] O Profiler do React DevTools mostra componentes `ProductCard` não re-renderizando em mudanças de estado não relacionadas.
- [ ] O array de dependências do `useMemo` está correto — `discountedProducts` recalcula apenas quando `products` ou `discountRate` muda.
- [ ] `useCallback` é aplicado a `onAddToCart` no componente pai — não dentro de `ProductList`.
- [ ] Um teste Vitest com `@testing-library/react` verifica que o componente renderiza uma vez no carregamento inicial.
- [ ] Sem otimização prematura: `React.memo` não é aplicado a todo componente — apenas aos que realmente re-renderizam desnecessariamente.

**Dicas:**
1. `useMemo`: `const discountedProducts = useMemo(() => products.map(...), [products, discountRate]);`.
2. `React.memo`: `export default React.memo(ProductCard)`. Faz comparação rasa de props por padrão.
3. O problema da prop `onAdd`: `() => onAddToCart(product.id)` cria uma nova referência de função em cada renderização, quebrando o `memo`. Correção: passe `productId` e `onAddToCart` como props estáveis separadas para `ProductCard`.
4. Confirme com o profiler: grave uma sessão, clique em "Adicionar ao Carrinho" em um produto, inspecione quais componentes `ProductCard` aparecem no flame graph.

---

## Exercício 6 — Corrigir Layout Thrashing em um Trecho JS (Médio)

**Cenário:** O JavaScript a seguir causa layout thrashing — alterna entre ler e escrever propriedades de layout, forçando o browser a recalcular o layout repetidamente. Corrija-o.

**Código fornecido (causa layout thrashing):**
```javascript
const items = document.querySelectorAll('.item');

items.forEach(item => {
  const height = item.offsetHeight; // LEITURA — força layout
  item.style.height = `${height * 2}px`; // ESCRITA — invalida layout
});
```

**Requisitos:**
- Refatore para agrupar todas as leituras primeiro, depois todas as escritas (separação leitura-escrita).
- Meça a diferença de performance usando `performance.mark` e `performance.measure`.
- Verifique com o painel de Performance do Chrome DevTools que os eventos "Recalculate Style" foram reduzidos.
- Aplique a mesma correção a um exemplo mais complexo: uma lista animada onde o `top` de cada item é calculado a partir da altura do irmão anterior.

**Critérios de Aceite:**
- [ ] Código corrigido realiza todas as leituras de `offsetHeight` em uma passagem, depois todas as escritas de `style.height` em uma segunda passagem.
- [ ] `performance.measure` mostra pelo menos um speedup de 5x em uma lista de 500 itens.
- [ ] O painel de Performance do Chrome DevTools mostra um único evento "Layout" em vez de N eventos de layout.
- [ ] Um comentário no código corrigido explica por que o original causava thrashing.
- [ ] A API `requestAnimationFrame` é usada se animações estiverem envolvidas.

**Dicas:**
1. Leituras em lote: `const heights = Array.from(items).map(item => item.offsetHeight);`.
2. Escritas em lote: `items.forEach((item, i) => { item.style.height = `${heights[i] * 2}px`; });`.
3. `performance.mark('inicio'); /* código */; performance.mark('fim'); performance.measure('layout', 'inicio', 'fim');` — leia de `performance.getEntriesByName('layout')`.
4. Para animações, use `requestAnimationFrame(() => { /* todas as escritas aqui */ })` para sincronizar com o ciclo de pintura do browser.

---

## Exercício 7 — Configurar Monitoramento de Core Web Vitals (Médio)

**Cenário:** Seu app React tem pontuações ruins no Lighthouse. Configure monitoramento de usuários reais (RUM) para Core Web Vitals: LCP, CLS, INP (antigo FID) e TTFB.

**Requisitos:**
- Use a biblioteca `web-vitals` para capturar LCP, CLS, INP e TTFB em sessões de usuários reais.
- Reporte métricas para seu endpoint de analytics (`POST /analytics/vitals`).
- Defina orçamentos de performance: LCP < 2,5s, CLS < 0,1, INP < 200ms.
- Adicione um step de CI usando Lighthouse CI (`lhci`) que falha se algum Core Web Vital exceder seu orçamento.
- Exiba um overlay no modo de desenvolvimento que mostra valores de métricas ao vivo.

**Critérios de Aceite:**
- [ ] `web-vitals` é importado e todas as quatro métricas são reportadas em cada carregamento de página.
- [ ] `POST /analytics/vitals` recebe `{ metric: 'LCP', value: 1234, url: '/dashboard', rating: 'good' }`.
- [ ] Config do Lighthouse CI (`lighthouserc.js`) define asserções para LCP, CLS e INP.
- [ ] Job do Lighthouse CI falha se LCP > 2500ms.
- [ ] O overlay de desenvolvimento aparece apenas quando `NODE_ENV=development` ou query param `?debug=vitals` está presente.

**Dicas:**
1. Uso do `web-vitals`:
   ```javascript
   import { onLCP, onCLS, onINP, onTTFB } from 'web-vitals';
   [onLCP, onCLS, onINP, onTTFB].forEach(fn => fn(metric => sendToAnalytics(metric)));
   ```
2. `sendToAnalytics`: use `navigator.sendBeacon('/analytics/vitals', JSON.stringify(metric))` — não bloqueia o descarregamento da página.
3. Lighthouse CI: `lhci autorun` no GitHub Actions. Config: `assertions: { 'largest-contentful-paint': ['warn', { maxNumericValue: 2500 }] }`.
4. Overlay de desenvolvimento: um `<div>` com posição fixa que se inscreve nos mesmos callbacks do `web-vitals` e atualiza seu conteúdo.

---

## Exercício 8 — Otimizar um Dockerfile para Imagem Menor (Fácil)

**Cenário:** A imagem Docker de uma aplicação Node.js tem 1,2 GB. Reduza para menos de 200 MB sem quebrar a funcionalidade.

**Dockerfile fornecido:**
```dockerfile
FROM node:20

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .
RUN npm run build

CMD ["node", "dist/index.js"]
```

**Requisitos:**
- Use uma build multi-estágio com imagem de runtime baseada em Alpine.
- Instale apenas dependências de produção na imagem final (`npm ci --omit=dev`).
- Use `.dockerignore` para excluir arquivos desnecessários do contexto de build.
- Fixe a imagem base em uma versão de patch específica.
- Verifique o tamanho final da imagem com `docker image ls`.

**Critérios de Aceite:**
- [ ] Tamanho da imagem final é menor que 200 MB (saída de `docker image ls`).
- [ ] O app ainda inicia e serve requisições corretamente após a otimização.
- [ ] Build multi-estágio: estágio `AS builder` faz o build; estágio `AS runtime` copia apenas `dist/` e `node_modules` de prod.
- [ ] `.dockerignore` exclui: `node_modules`, `.git`, `*.log`, `.env*`, `test/`, `coverage/`.
- [ ] Imagem base está fixada (ex: `node:20.11.1-alpine3.19`), não uma tag flutuante.

**Dicas:**
1. Multi-estágio:
   ```dockerfile
   FROM node:20-alpine AS builder
   WORKDIR /app
   COPY package*.json ./
   RUN npm ci
   COPY . .
   RUN npm run build

   FROM node:20-alpine AS runtime
   WORKDIR /app
   COPY package*.json ./
   RUN npm ci --omit=dev
   COPY --from=builder /app/dist ./dist
   CMD ["node", "dist/index.js"]
   ```
2. Imagens Alpine têm ~5 MB vs ~900 MB para `node:20` Debian. Combinado com omitir dependências de dev, espere uma redução de tamanho de 5–10x.
3. Verifique o que é grande no estágio builder: `docker run --rm builder du -sh /app/node_modules` para identificar dependências pesadas.
4. Se módulos nativos são necessários: use `node:20-alpine` com `apk add --no-cache python3 make g++` apenas no estágio builder.

---

## Exercício 9 — Adicionar Headers de Cache HTTP (ETags + Cache-Control) (Médio)

**Cenário:** Sua API retorna dados de produtos que são caros de computar mas raramente mudam. Adicione headers de cache HTTP para que browsers e CDNs possam cachear respostas, e suporte requisições condicionais para evitar transferência desnecessária de dados.

**Requisitos:**
- `GET /products/:id`: adicione `Cache-Control: public, max-age=300` e um header `ETag`.
- Quando o cliente enviar `If-None-Match: <etag>`, responda com `304 Not Modified` se o conteúdo não mudou.
- `Cache-Control` para endpoints autenticados deve ser `private, no-store` — não cacheável publicamente.
- Para `GET /products` (lista): use `Cache-Control: public, max-age=60, stale-while-revalidate=600`.
- Implemente geração de ETag: hash MD5 ou SHA-1 do JSON do corpo da resposta.

**Critérios de Aceite:**
- [ ] Resposta de `GET /products/1` inclui `ETag: "abc123"` e `Cache-Control: public, max-age=300`.
- [ ] Enviar `If-None-Match: "abc123"` em uma requisição subsequente retorna `304` sem corpo.
- [ ] Após o produto ser atualizado, o ETag muda e a próxima requisição condicional retorna `200` com os novos dados.
- [ ] `GET /profile` (autenticado) retorna `Cache-Control: private, no-store`.
- [ ] Um teste verifica o fluxo 304 de ponta a ponta usando `fastify.inject`.

**Dicas:**
1. Geração de ETag: `const etag = '"' + createHash('md5').update(JSON.stringify(body)).digest('hex') + '"'`. Envolva em aspas duplas conforme RFC 7232.
2. Verificação condicional: `if (req.headers['if-none-match'] === etag) { reply.code(304).send(); return; }`.
3. `stale-while-revalidate`: instrui CDNs a servir uma resposta em cache desatualizada enquanto busca uma nova em segundo plano.
4. No Fastify, defina headers antes de `reply.send`: `reply.header('ETag', etag).header('Cache-Control', 'public, max-age=300')`.

---

## Exercício 10 — Implementar Lazy Loading para um Módulo Pesado (Fácil)

**Cenário:** Seu app Node.js importa uma biblioteca de geração de PDF (`pdf-lib`, ~2 MB) na inicialização, adicionando 800ms ao tempo de cold start. Apenas 5% das requisições realmente usam geração de PDF. Implemente lazy loading.

**Requisitos:**
- Mova o import de `pdf-lib` do nível superior do módulo para dentro da função que o usa.
- A primeira chamada a `generatePdf()` dispara o import; chamadas subsequentes reutilizam o módulo em cache.
- Use sintaxe `import()` dinâmico — não `require()`.
- Meça a melhoria no tempo de cold start com `process.hrtime.bigint()`.
- Aplique o mesmo padrão a outras duas dependências pesadas na codebase.

**Critérios de Aceite:**
- [ ] Tempo de inicialização do servidor é reduzido em pelo menos 500ms.
- [ ] `generatePdf()` funciona corretamente na primeira chamada e em todas as subsequentes.
- [ ] O import lazy é em cache — `import()` é chamado apenas uma vez, não em cada invocação de `generatePdf()`.
- [ ] Tipos TypeScript são preservados — o módulo importado é corretamente tipado mesmo com import dinâmico.
- [ ] Um comentário explica o trade-off: inicialização mais rápida vs. latência ligeiramente maior na primeira requisição de PDF.

**Dicas:**
1. Padrão de import lazy:
   ```typescript
   let pdfLib: typeof import('pdf-lib') | null = null;
   async function generatePdf(data: PdfData): Promise<Buffer> {
     if (!pdfLib) pdfLib = await import('pdf-lib');
     const { PDFDocument } = pdfLib;
     // ...
   }
   ```
2. Tipos TypeScript com import dinâmico: `import type { PDFDocument } from 'pdf-lib'` no topo apenas para tipos — não afeta o bundle.
3. Meça a inicialização: `hyperfine --warmup 3 'node -e "require(\"./dist/server\")"'` — execute antes e depois para comparar.
4. Outros candidatos para lazy loading: bibliotecas de processamento de imagem (`sharp`), geradores de planilhas (`exceljs`), geradores de QR code.
