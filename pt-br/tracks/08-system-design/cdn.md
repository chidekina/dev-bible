# CDN

## Visão Geral

Uma Content Delivery Network (CDN) é uma rede geograficamente distribuída de servidores (edge nodes) que fazem cache e servem conteúdo de locais próximos aos usuários. Em vez de todo usuário bater no seu servidor de origem em São Paulo, um usuário em Tóquio recebe o conteúdo de um edge node em Tóquio. Isso reduz a latência, alivia a carga no servidor de origem e melhora a resiliência. CDNs servem assets estáticos, mas CDNs modernas também lidam com conteúdo dinâmico, respostas de API, otimização de imagens e edge computing. Este capítulo cobre os fundamentos de CDN, invalidação de cache e configuração prática.

---

## Pré-requisitos

- Headers HTTP de caching (Cache-Control, ETags)
- Entendimento básico de DNS
- Track 07 — Security: conceitos de HTTPS/TLS

---

## Conceitos Fundamentais

### Como o caching da CDN funciona

```
Primeira requisição (cache miss):
Usuário (Tóquio) → CDN Edge (Tóquio) → Origem (São Paulo) → Resposta
                ↑ CDN armazena a resposta

Requisições subsequentes (cache hit):
Usuário (Tóquio) → CDN Edge (Tóquio) → Resposta em cache (rápido — sem bater na origem)
```

**Cache hit ratio:** o percentual de requisições servidas do cache. Meta: > 90% para assets estáticos.

### Tipos de conteúdo

| Tipo de conteúdo | Estratégia CDN | TTL típico |
|-----------------|----------------|-----------|
| Assets imutáveis (JS, CSS com hash no nome) | Cache para sempre | 1 ano |
| Imagens, fontes | Cache longo | 30 dias |
| Páginas HTML | Cache curto ou sem cache | 0–5 minutos |
| Respostas de API | Depende do frescor dos dados | 0 segundos a minutos |
| Dados específicos de usuário | Geralmente não cacheados | N/A |

### CDN Pull vs Push

**CDN Pull (mais comum):** A CDN busca da origem na primeira requisição, depois faz cache. Zero configuração — basta configurar o DNS.

**CDN Push:** Você faz upload explícito de assets para a CDN. Útil para arquivos grandes (vídeos, backups) onde você quer pré-posicionar o conteúdo antes de alguém requisitar.

---

## Exemplos Práticos

### Configurando headers Cache-Control no Fastify

```typescript
// src/plugins/cache-headers.ts
import { FastifyPluginAsync } from 'fastify';

const cacheHeaders: FastifyPluginAsync = async (fastify) => {
  // Assets estáticos servidos pelo Fastify (geralmente tratados pelo Nginx/CDN em vez disso)
  fastify.register(import('@fastify/static'), {
    root: new URL('../public', import.meta.url).pathname,
  });

  // Define headers de caching por tipo de rota
  fastify.addHook('onSend', async (request, reply) => {
    const url = request.url;

    // Assets estáticos imutáveis (nomes com hash: app.abc123.js)
    if (/\.(js|css|woff2|woff|ttf)$/.test(url) && /[a-f0-9]{8,}/.test(url)) {
      reply.header('Cache-Control', 'public, max-age=31536000, immutable');
      return;
    }

    // Imagens — cache longo
    if (/\.(png|jpg|jpeg|webp|avif|svg|ico|gif)$/.test(url)) {
      reply.header('Cache-Control', 'public, max-age=2592000'); // 30 dias
      return;
    }

    // Páginas HTML — cache curto com revalidação
    if (reply.getHeader('Content-Type')?.toString().includes('text/html')) {
      reply.header('Cache-Control', 'public, max-age=60, stale-while-revalidate=300');
      return;
    }

    // Respostas de API — não cacheadas por padrão
    if (url.startsWith('/api/')) {
      reply.header('Cache-Control', 'no-store');
      return;
    }
  });
};
```

### Cache-Control para respostas de API

```typescript
// Respostas de API públicas e cacheáveis (catálogo de produtos, conteúdo público)
fastify.get('/api/products', async (request, reply) => {
  const products = await getProducts();

  // Cache na CDN e no navegador por 5 minutos; serve desatualizado por 1 hora enquanto revalida
  reply.header('Cache-Control', 'public, max-age=300, stale-while-revalidate=3600');

  return products;
});

// Dados específicos do usuário — nunca faça cache
fastify.get('/api/me', { preHandler: [requireAuth] }, async (request) => {
  // implicitamente: reply.header('Cache-Control', 'no-store')
  return getUserProfile(request.user.id);
});

// Caching condicional baseado em ETag
fastify.get('/api/products/:id', async (request, reply) => {
  const product = await getProduct(request.params.id);
  const etag = `"${product.updatedAt.getTime()}"`;

  reply.header('ETag', etag);
  reply.header('Cache-Control', 'public, max-age=60');

  // Se o cliente envia If-None-Match e os ETags batem, retorna 304
  if (request.headers['if-none-match'] === etag) {
    return reply.status(304).send();
  }

  return product;
});
```

### Invalidação de cache no Cloudflare

```typescript
// Purga URLs específicas do cache do Cloudflare após mudanças de conteúdo
async function purgeCloudflareCache(urls: string[]): Promise<void> {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/zones/${process.env.CLOUDFLARE_ZONE_ID}/purge_cache`,
    {
      method: 'POST',
      headers: {
        Authorization: `Bearer ${process.env.CLOUDFLARE_API_TOKEN}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ files: urls }),
    }
  );

  if (!response.ok) {
    throw new Error(`Cloudflare purge failed: ${response.statusText}`);
  }
}

// Uso: após atualizar um produto
await db.product.update({ where: { id }, data });
await purgeCloudflareCache([
  `https://www.example.com/api/products/${id}`,
  `https://www.example.com/products/${id}`,
]);
```

### Estratégia de nomeação de assets (caching imutável)

A chave para cache-busting sem invalidação de cache é nomes de arquivo endereçados por conteúdo:

```typescript
// vite.config.ts
export default {
  build: {
    rollupOptions: {
      output: {
        // Hash no nome do arquivo — quando o conteúdo muda, o nome muda → nova entrada de cache
        entryFileNames: 'assets/[name].[hash].js',
        chunkFileNames: 'assets/[name].[hash].js',
        assetFileNames: 'assets/[name].[hash][extname]',
      },
    },
  },
};

// Após o build:
// assets/app.a1b2c3d4.js  → Cache-Control: immutable, max-age=31536000
// assets/app.e5f6a7b8.js  → novo hash = nova URL = sem conflito de cache
```

### Configuração do Cloudflare (via Terraform)

```hcl
# Page rules para comportamento de caching
resource "cloudflare_page_rule" "static_assets" {
  zone_id = var.zone_id
  target  = "*.example.com/assets/*"
  priority = 1

  actions {
    cache_level = "cache_everything"
    edge_cache_ttl = 2592000  # 30 dias no edge
    browser_cache_ttl = 31536000
  }
}

resource "cloudflare_page_rule" "api_no_cache" {
  zone_id = var.zone_id
  target  = "*.example.com/api/*"
  priority = 2

  actions {
    cache_level = "bypass"
  }
}
```

### CDN para otimização de imagens

CDNs modernas (Cloudflare Images, Imgix, Cloudinary) podem transformar imagens em tempo real:

```typescript
// src/lib/image.ts — gera URLs de imagem otimizadas
const CDN_BASE = 'https://images.example.com';

interface ImageOptions {
  width?: number;
  height?: number;
  format?: 'webp' | 'avif' | 'auto';
  quality?: number;
}

export function optimizedImageUrl(path: string, options: ImageOptions = {}): string {
  const params = new URLSearchParams();
  if (options.width) params.set('w', String(options.width));
  if (options.height) params.set('h', String(options.height));
  if (options.format) params.set('fm', options.format);
  if (options.quality) params.set('q', String(options.quality));

  return `${CDN_BASE}${path}?${params.toString()}`;
}

// Uso em componente React
function ProductImage({ product }: { product: Product }) {
  return (
    <picture>
      <source
        srcSet={optimizedImageUrl(product.imageUrl, { width: 800, format: 'avif' })}
        type="image/avif"
      />
      <source
        srcSet={optimizedImageUrl(product.imageUrl, { width: 800, format: 'webp' })}
        type="image/webp"
      />
      <img
        src={optimizedImageUrl(product.imageUrl, { width: 800 })}
        alt={product.name}
        loading="lazy"
        decoding="async"
      />
    </picture>
  );
}
```

---

## Padrões e Boas Práticas

- **Assets imutáveis com content hashes** — nunca se preocupe com invalidação de cache para JS/CSS
- **TTL curto com `stale-while-revalidate`** — serve conteúdo em cache enquanto busca versão fresca em background
- **Separe origens de estáticos e API** — `cdn.example.com` para assets, `api.example.com` para API (CDN pode ser configurada diferentemente por subdomínio)
- **Faça warm-up do cache após deploy** — faça requisições para URLs críticas antes de direcionar tráfego de usuários
- **Monitore o cache hit rate** — disponível no Cloudflare Analytics, Fastly, AWS CloudFront

---

## Anti-Padrões a Evitar

- Cachear respostas específicas de usuário (dados de perfil, conteúdo do carrinho) no nível da CDN — todo usuário veria os mesmos dados
- TTLs muito longos em nomes de arquivo sem hash — quando você faz deploy de uma atualização, usuários veem conteúdo desatualizado até o TTL expirar
- Cachear respostas 500 — um erro temporário pode ser cacheado e servido por horas
- Não definir o header `Vary` quando o conteúdo varia por `Accept-Encoding` ou `Accept` — a CDN pode servir conteúdo comprimido para clientes que não suportam

---

## Debugging e Troubleshooting

**"Usuários estão vendo conteúdo antigo após um deploy"**
Seus assets não têm content hashes nos nomes de arquivo. A CDN serve a versão em cache até o TTL expirar. Solução: use uma ferramenta de build que gera nomes com hash, ou purgue o cache da CDN após cada deploy.

**"CDN está servindo conteúdo errado para usuários diferentes"**
Verifique os headers `Vary`. Se sua API serve respostas diferentes com base em `Authorization` ou `Accept-Language`, a CDN deve ser configurada para variar o cache por esses headers, ou contornar o caching completamente para essas rotas.

**"Hit rate de cache é 20% para assets estáticos"**
Verifique se query strings estão causando cache misses (`/styles.css?v=1` e `/styles.css?v=2` são tratadas como URLs diferentes por muitas CDNs). Normalize URLs ou use nomeação com hash em vez disso.

---

## Leitura Complementar

- [Documentação de Cache do Cloudflare](https://developers.cloudflare.com/cache/)
- [MDN: Cache-Control](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control)
- [web.dev: HTTP caching](https://web.dev/http-cache/)
- Track 10: [HTTP Caching](../10-performance/caching-http.md)
- Track 08: [Caching Strategies](caching-strategies.md)

---

## Resumo

CDNs são a melhoria de performance mais custo-efetiva para aplicações web que servem usuários globais. A chave para maximizar a eficácia da CDN são headers Cache-Control corretos: conteúdo imutável com nomes de arquivo com hash recebe caching infinito sem complexidade de invalidação; conteúdo dinâmico recebe TTLs curtos ou contorna o caching completamente. Invalidação de cache deve ser orientada por content hashes (em tempo de build) em vez de purgas manuais (sujeitas a erros). CDNs modernas vão além de servir arquivos estáticos para otimização de imagens, edge compute e proteção DDoS — tornando-as uma camada fundamental de qualquer arquitetura de produção séria.
