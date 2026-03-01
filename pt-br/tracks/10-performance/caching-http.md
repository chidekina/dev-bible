# HTTP Caching

## Visão Geral

HTTP caching é o mecanismo pelo qual navegadores, CDNs e proxies armazenam respostas e as reutilizam em requisições subsequentes. Quando feito corretamente, elimina categorias inteiras de requisições de rede — o usuário obtém o recurso do cache local em milissegundos e o servidor nunca vê a requisição. Este capítulo cobre o header Cache-Control em detalhes, ETags, a estratégia `stale-while-revalidate` e como projetar uma estratégia de caching para diferentes tipos de recursos.

---

## Pré-requisitos

- Fundamentos de HTTP (headers, status codes)
- Entendimento de CDNs (Track 08)
- Noções básicas de Fastify ou Express

---

## Conceitos Fundamentais

### Onde o caching acontece

```
Cache do navegador → Cache de borda da CDN → Cache de reverse proxy → Servidor de origem
```

Os headers HTTP controlam o comportamento em cada camada. Uma resposta com `Cache-Control: public, max-age=3600` é armazenada em cache por 1 hora pelo navegador, pela CDN e por qualquer proxy intermediário.

### Diretivas do Cache-Control

| Diretiva | Significado |
|----------|-------------|
| `public` | A resposta pode ser armazenada por qualquer cache (navegador + CDN) |
| `private` | A resposta só pode ser armazenada pelo navegador (não pela CDN) |
| `no-store` | Nunca armazenar — sempre buscar do servidor |
| `no-cache` | Pode armazenar, mas deve revalidar antes de usar (nome confuso) |
| `max-age=N` | Cache válido por N segundos a partir do momento da resposta |
| `s-maxage=N` | Sobrescreve `max-age` apenas para caches compartilhados (CDNs) |
| `immutable` | Indica que o recurso nunca mudará (pula revalidação) |
| `stale-while-revalidate=N` | Serve conteúdo stale por até N segundos enquanto busca conteúdo fresco em segundo plano |
| `must-revalidate` | Deve revalidar ao expirar (sem servir stale) |

---

## Exemplos Práticos

### Configurando Cache-Control no Fastify

```typescript
// src/plugins/cache-control.ts
import { FastifyPluginAsync } from 'fastify';

const cacheControl: FastifyPluginAsync = async (fastify) => {
  fastify.addHook('onSend', async (request, reply, payload) => {
    const url = request.url;

    // Assets com hash imutável — cache eterno, navegador pode pular revalidação
    if (/\.(js|css|woff2|ttf)$/.test(url) && /[a-f0-9]{8}/.test(url)) {
      reply.header('Cache-Control', 'public, max-age=31536000, immutable');
      return payload;
    }

    // Respostas de API com dados que podem estar levemente desatualizados
    if (url.startsWith('/api/products') && request.method === 'GET') {
      reply.header('Cache-Control', 'public, max-age=60, stale-while-revalidate=600');
      return payload;
    }

    // Dados privados do usuário — navegador pode armazenar, CDN não
    if (url.startsWith('/api/me') || url.startsWith('/api/account')) {
      reply.header('Cache-Control', 'private, max-age=300');
      return payload;
    }

    // Padrão: sem caching para endpoints de API
    if (url.startsWith('/api/')) {
      reply.header('Cache-Control', 'no-store');
      return payload;
    }

    return payload;
  });
};
```

### Caching condicional com ETag

ETags permitem que o navegador pergunte "isso mudou?". O servidor retorna 304 Not Modified se o conteúdo for o mesmo — nenhum body é enviado, economizando banda:

```typescript
import crypto from 'crypto';

function generateETag(content: unknown): string {
  const hash = crypto
    .createHash('sha256')
    .update(JSON.stringify(content))
    .digest('hex')
    .slice(0, 16);
  return `"${hash}"`;
}

fastify.get('/api/products/:id', async (request, reply) => {
  const { id } = request.params as { id: string };
  const product = await db.product.findUnique({ where: { id } });

  if (!product) return reply.status(404).send({ error: 'Not found' });

  const etag = generateETag(product);
  reply.header('ETag', etag);
  reply.header('Cache-Control', 'public, max-age=60, stale-while-revalidate=300');

  // Se o cliente enviar header If-None-Match com o mesmo ETag, retorna 304
  if (request.headers['if-none-match'] === etag) {
    return reply.status(304).send();
  }

  return product;
});
```

A segunda requisição do navegador inclui `If-None-Match: "abc123"`. Se o produto não mudou, o servidor retorna `304` sem body — o navegador usa sua cópia em cache.

### Caching condicional com Last-Modified

Alternativa às ETags usando timestamps:

```typescript
fastify.get('/api/posts/:id', async (request, reply) => {
  const post = await db.post.findUnique({ where: { id: request.params.id } });
  if (!post) return reply.status(404).send();

  const lastModified = post.updatedAt.toUTCString();
  reply.header('Last-Modified', lastModified);
  reply.header('Cache-Control', 'public, max-age=300');

  // Verifica If-Modified-Since
  const ifModifiedSince = request.headers['if-modified-since'];
  if (ifModifiedSince && new Date(ifModifiedSince) >= post.updatedAt) {
    return reply.status(304).send();
  }

  return post;
});
```

### stale-while-revalidate — servindo conteúdo stale enquanto atualiza

```
Cliente requisita /api/products
  max-age=60 expirou (cache stale)
  stale-while-revalidate=600 ainda NÃO expirou

  → Serve resposta stale imediatamente (rápido para o usuário)
  → Em segundo plano: busca conteúdo fresco da origem e atualiza o cache
```

```typescript
// CDN e navegador tratam stale-while-revalidate automaticamente
// Você apenas define o header:
reply.header('Cache-Control', 'public, max-age=60, stale-while-revalidate=600');

// Stale por 1 minuto → servido do cache
// Após 1 minuto → cache busca conteúdo fresco em segundo plano, serve stale enquanto aguarda
// Após 11 minutos (1 + 10) → cache expirado, deve bloquear para busca fresca
```

Essa é a melhor estratégia para dados que podem estar 1–2 minutos desatualizados sem impacto ao usuário.

### Header Vary — caching de respostas com negociação de conteúdo

Quando a mesma URL retorna conteúdo diferente com base em headers da requisição (ex: Accept, Accept-Language), use `Vary`:

```typescript
fastify.get('/api/products', async (request, reply) => {
  const acceptLanguage = request.headers['accept-language'] ?? 'en';
  const lang = acceptLanguage.startsWith('pt') ? 'pt' : 'en';

  const products = await getLocalizedProducts(lang);

  // Armazena em cache separadamente por idioma
  reply.header('Vary', 'Accept-Language');
  reply.header('Cache-Control', 'public, max-age=300');

  return products;
});
```

Sem `Vary: Accept-Language`, uma CDN poderia servir uma resposta em inglês para um usuário brasileiro que requisitou a mesma URL.

### Cache-busting para páginas HTML

Páginas HTML referenciam assets por URL. Quando você faz deploy de novo código:
- Assets com nomes hasheados: novo hash = nova URL = cache miss = novo download
- Páginas HTML: precisam ser atualizadas para que os usuários obtenham as novas URLs de assets

```typescript
// Páginas HTML — TTL curto, sempre revalidar
reply.header('Cache-Control', 'public, max-age=0, must-revalidate');
// Ou use ETag — navegador revalida, servidor retorna 304 se não mudou
```

### Service Worker caching (offline-first)

```typescript
// public/sw.js
const CACHE_VERSION = 'v2';
const STATIC_CACHE = `static-${CACHE_VERSION}`;
const API_CACHE = `api-${CACHE_VERSION}`;

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(STATIC_CACHE).then((cache) =>
      cache.addAll(['/offline.html', '/styles/main.css', '/scripts/app.js'])
    )
  );
});

self.addEventListener('fetch', (event) => {
  const url = new URL(event.request.url);

  // Respostas de API: network first, cache como fallback
  if (url.pathname.startsWith('/api/')) {
    event.respondWith(
      fetch(event.request)
        .then((response) => {
          const clone = response.clone();
          caches.open(API_CACHE).then((cache) => cache.put(event.request, clone));
          return response;
        })
        .catch(() => caches.match(event.request)) // fallback offline
    );
    return;
  }

  // Assets estáticos: cache first
  event.respondWith(
    caches.match(event.request).then((cached) => cached ?? fetch(event.request))
  );
});
```

---

## Padrões e Boas Práticas

- **Assets imutáveis com hashes** — `max-age=31536000, immutable`; invalidação de cache sem custo
- **Páginas HTML** — TTL curto (0–60s) com ETag para revalidação condicional
- **Dados públicos de API** — `stale-while-revalidate` para frescor + performance
- **Dados do usuário** — `private` para que a CDN não armazene; navegador pode armazenar brevemente
- **Endpoints autenticados** — `no-store` ou `private` dependendo da sensibilidade
- **Purge da CDN no deploy** — limpe as páginas HTML após cada deploy

---

## Anti-Padrões a Evitar

- `max-age=0` sem `must-revalidate` ou ETags — o cache é inútil
- `Cache-Control: public` em respostas específicas de usuário — a CDN serve dados de um usuário para outro
- TTL longo em arquivos estáticos sem hash — usuários veem JS/CSS desatualizados após deploy
- Não definir `Vary` quando as respostas diferem por header de requisição
- `no-cache` quando você quer dizer `no-store` — `no-cache` significa "armazene mas revalide", não "nunca armazene"

---

## Debugging e Resolução de Problemas

**"O navegador não está armazenando minha resposta em cache"**
Verifique os headers de resposta na aba Network do DevTools. Se `Cache-Control: no-store`, o navegador descarta a resposta imediatamente. Se há um `Set-Cookie` na resposta, alguns navegadores não armazenam mesmo com `public`.

**"A CDN está servindo conteúdo stale após o deploy"**
O TTL do cache não expirou e você não fez purge. Use a API de purge da CDN ou mude a URL do asset (adicione um parâmetro de versão ou use content hashes).

**"Usuários estão recebendo conteúdo diferente na mesma URL"**
Está faltando o header `Vary`. Adicione `Vary: Accept-Language` ou `Vary: Accept-Encoding` para que os caches armazenem cópias separadas para variantes diferentes.

---

## Leituras Complementares

- [MDN: Cache-Control](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control)
- [web.dev: HTTP caching guide](https://web.dev/http-cache/)
- [RFC 9111: HTTP Caching](https://httpwg.org/specs/rfc9111.html)
- Track 08: [CDN](../08-system-design/cdn.md)
- Track 08: [Estratégias de Caching](../08-system-design/caching-strategies.md)

---

## Resumo

HTTP caching é a otimização de performance de maior impacto que não exige nenhuma mudança no código da aplicação para assets estáticos. A estratégia é simples: assets imutáveis com content hashes recebem `max-age=31536000, immutable` (cache eterno, sem necessidade de invalidação); páginas HTML recebem TTLs curtos com ETags para revalidação condicional; dados públicos de API usam `stale-while-revalidate` para frescor sem bloqueio; dados específicos de usuário usam `private` ou `no-store`. O erro mais comum é não fazer nenhum caching (banda desperdiçada, carregamentos lentos) ou fazer caching incorretamente (`public` em dados privados, TTLs longos em arquivos sem hash).
