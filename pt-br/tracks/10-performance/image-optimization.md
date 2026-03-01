# Otimização de Imagens

## Visão Geral

Imagens respondem por 50–70% do peso de página na maioria dos sites. Uma biblioteca de imagens não otimizadas pode facilmente adicionar 5–10 segundos ao tempo de carregamento em uma conexão móvel. A otimização de imagens abrange a escolha do formato correto, o dimensionamento adequado, a compressão eficiente, o serving responsivo e o lazy loading de imagens abaixo da dobra. Este capítulo cobre o pipeline completo: desde o processamento de uploads com Sharp até o serving responsivo e a negociação de formato.

---

## Pré-requisitos

- Elementos HTML `<img>` e `<picture>`
- Entendimento básico de CDNs
- Node.js para processamento com Sharp

---

## Conceitos Fundamentais

### Formatos de imagem modernos

| Formato | Suporte do navegador | Caso de uso | Tamanho típico vs JPEG |
|---------|---------------------|-------------|------------------------|
| JPEG | Universal | Fotos | Linha de base |
| PNG | Universal | Transparência, screenshots | 2–3× maior que JPEG |
| WebP | 97%+ | Fotos + transparência | 25–35% menor que JPEG |
| AVIF | 90%+ | Fotos | 50% menor que JPEG |
| SVG | Universal | Ícones, logos | Vetorial — escala perfeitamente |

**Estratégia:** sirva AVIF para navegadores que suportam, WebP como fallback e JPEG/PNG para navegadores mais antigos.

### O checklist de otimização de imagens

- Dimensões corretas (não sirva uma imagem de 2000px para um slot de 400px)
- Formato adequado (AVIF/WebP para fotos, SVG para ícones)
- Compressão adequada (qualidade 75–85 para fotos)
- Tamanhos responsivos (`srcset` para diferentes tamanhos de viewport)
- Lazy loading para imagens abaixo da dobra
- Entrega via CDN (geograficamente próxima dos usuários)

---

## Exemplos Práticos

### Processamento no servidor com Sharp

```bash
npm install sharp
```

```typescript
// src/lib/image-processor.ts
import sharp from 'sharp';
import path from 'path';

interface ProcessOptions {
  width?: number;
  height?: number;
  quality?: number;
}

interface ProcessedImage {
  webp: Buffer;
  avif: Buffer;
  jpeg: Buffer;
  width: number;
  height: number;
}

export async function processUploadedImage(
  inputBuffer: Buffer,
  options: ProcessOptions = {}
): Promise<ProcessedImage> {
  const { width = 1200, quality = 80 } = options;

  const base = sharp(inputBuffer)
    .resize(width, undefined, {
      withoutEnlargement: true,  // não amplia imagens pequenas
      fit: 'inside',
    });

  const [webp, avif, jpeg, metadata] = await Promise.all([
    base.clone().webp({ quality }).toBuffer(),
    base.clone().avif({ quality: quality - 5 }).toBuffer(), // AVIF usa qualidade um pouco menor para a mesma qualidade perceptual
    base.clone().jpeg({ quality, progressive: true }).toBuffer(),
    base.metadata(),
  ]);

  return {
    webp,
    avif,
    jpeg,
    width: metadata.width ?? width,
    height: metadata.height ?? 0,
  };
}

// Gera múltiplos tamanhos para imagens responsivas
export async function generateResponsiveSizes(
  inputBuffer: Buffer,
  sizes = [400, 800, 1200, 1920]
): Promise<Map<number, ProcessedImage>> {
  const results = new Map<number, ProcessedImage>();

  await Promise.all(
    sizes.map(async (size) => {
      const processed = await processUploadedImage(inputBuffer, { width: size });
      results.set(size, processed);
    })
  );

  return results;
}
```

### Endpoint de upload com processamento de imagem

```typescript
// src/routes/uploads.ts
import { pipeline } from 'stream/promises';
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { processUploadedImage } from '../lib/image-processor.js';

const s3 = new S3Client({ region: process.env.AWS_REGION });
const CDN_BASE = process.env.CDN_BASE_URL!;

fastify.post('/api/uploads/image', { preHandler: [requireAuth] }, async (request, reply) => {
  const data = await request.file();
  if (!data) return reply.status(400).send({ error: 'Nenhum arquivo enviado' });

  // Valida o tipo do arquivo
  if (!data.mimetype.startsWith('image/')) {
    return reply.status(400).send({ error: 'O arquivo deve ser uma imagem' });
  }

  const inputBuffer = await data.toBuffer();

  // Limita o tamanho (5MB)
  if (inputBuffer.length > 5 * 1024 * 1024) {
    return reply.status(400).send({ error: 'Imagem muito grande (máx 5MB)' });
  }

  const id = crypto.randomUUID();
  const processed = await processUploadedImage(inputBuffer);

  // Faz upload de todos os formatos para o S3 em paralelo
  await Promise.all([
    s3.send(new PutObjectCommand({
      Bucket: process.env.S3_BUCKET!,
      Key: `images/${id}.webp`,
      Body: processed.webp,
      ContentType: 'image/webp',
      CacheControl: 'public, max-age=31536000, immutable',
    })),
    s3.send(new PutObjectCommand({
      Bucket: process.env.S3_BUCKET!,
      Key: `images/${id}.avif`,
      Body: processed.avif,
      ContentType: 'image/avif',
      CacheControl: 'public, max-age=31536000, immutable',
    })),
    s3.send(new PutObjectCommand({
      Bucket: process.env.S3_BUCKET!,
      Key: `images/${id}.jpg`,
      Body: processed.jpeg,
      ContentType: 'image/jpeg',
      CacheControl: 'public, max-age=31536000, immutable',
    })),
  ]);

  return {
    id,
    urls: {
      avif: `${CDN_BASE}/images/${id}.avif`,
      webp: `${CDN_BASE}/images/${id}.webp`,
      jpeg: `${CDN_BASE}/images/${id}.jpg`,
    },
    width: processed.width,
    height: processed.height,
  };
});
```

### Imagens responsivas em HTML

```html
<!-- elemento picture — o navegador seleciona o melhor formato + tamanho -->
<picture>
  <!-- AVIF para navegadores compatíveis (menor tamanho) -->
  <source
    srcset="
      /images/hero-400.avif  400w,
      /images/hero-800.avif  800w,
      /images/hero-1200.avif 1200w
    "
    sizes="(max-width: 600px) 400px, (max-width: 1200px) 800px, 1200px"
    type="image/avif"
  />

  <!-- Fallback WebP -->
  <source
    srcset="
      /images/hero-400.webp  400w,
      /images/hero-800.webp  800w,
      /images/hero-1200.webp 1200w
    "
    sizes="(max-width: 600px) 400px, (max-width: 1200px) 800px, 1200px"
    type="image/webp"
  />

  <!-- JPEG como fallback final -->
  <img
    src="/images/hero-800.jpg"
    srcset="
      /images/hero-400.jpg  400w,
      /images/hero-800.jpg  800w,
      /images/hero-1200.jpg 1200w
    "
    sizes="(max-width: 600px) 400px, (max-width: 1200px) 800px, 1200px"
    alt="Imagem hero"
    width="1200"
    height="600"
    loading="eager"
    fetchpriority="high"
  />
</picture>
```

### Imagens responsivas em React

```tsx
// src/components/OptimizedImage.tsx
interface OptimizedImageProps {
  id: string;
  alt: string;
  width: number;
  height: number;
  sizes?: string;
  priority?: boolean;
}

const CDN = process.env.NEXT_PUBLIC_CDN_URL ?? '';
const RESPONSIVE_WIDTHS = [400, 800, 1200, 1920];

function buildSrcSet(id: string, format: string): string {
  return RESPONSIVE_WIDTHS
    .map((w) => `${CDN}/images/${id}-${w}.${format} ${w}w`)
    .join(', ');
}

export function OptimizedImage({
  id,
  alt,
  width,
  height,
  sizes = '(max-width: 768px) 100vw, 50vw',
  priority = false,
}: OptimizedImageProps) {
  return (
    <picture>
      <source srcSet={buildSrcSet(id, 'avif')} sizes={sizes} type="image/avif" />
      <source srcSet={buildSrcSet(id, 'webp')} sizes={sizes} type="image/webp" />
      <img
        src={`${CDN}/images/${id}-800.jpg`}
        srcSet={buildSrcSet(id, 'jpg')}
        sizes={sizes}
        alt={alt}
        width={width}
        height={height}
        loading={priority ? 'eager' : 'lazy'}
        decoding={priority ? 'sync' : 'async'}
        fetchPriority={priority ? 'high' : 'auto'}
      />
    </picture>
  );
}
```

### Componente Image do Next.js

O Next.js cuida da otimização automaticamente:

```tsx
import Image from 'next/image';

// next.config.js
module.exports = {
  images: {
    domains: ['cdn.example.com'],
    formats: ['image/avif', 'image/webp'],
    deviceSizes: [640, 828, 1080, 1200, 1920],
    imageSizes: [16, 32, 48, 64, 96, 128, 256, 384],
  },
};

// Uso
function ProductCard({ product }: { product: Product }) {
  return (
    <Image
      src={product.imageUrl}
      alt={product.name}
      width={400}
      height={300}
      sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
      // priority para imagens acima da dobra
      placeholder="blur"
      blurDataURL={product.imagePlaceholder} // placeholder de baixa qualidade
    />
  );
}
```

### Lazy loading

```html
<!-- Lazy loading nativo — o navegador gerencia automaticamente -->
<img
  src="/images/below-fold.jpg"
  alt="..."
  loading="lazy"     <!-- adia o carregamento até próximo ao viewport -->
  decoding="async"   <!-- não bloqueia a thread principal durante a decodificação -->
  width="800"
  height="600"
/>
```

```typescript
// Intersection Observer para lazy loading controlado por JavaScript
function lazyLoadImages() {
  const images = document.querySelectorAll('img[data-src]');

  const observer = new IntersectionObserver(
    (entries) => {
      entries.forEach((entry) => {
        if (entry.isIntersecting) {
          const img = entry.target as HTMLImageElement;
          img.src = img.dataset.src!;
          img.removeAttribute('data-src');
          observer.unobserve(img);
        }
      });
    },
    { rootMargin: '200px' } // carrega 200px antes de entrar no viewport
  );

  images.forEach((img) => observer.observe(img));
}
```

---

## Padrões e Boas Práticas

- **Sempre defina `width` e `height`** — evita CLS (layout shift) enquanto as imagens carregam
- **Use `loading="lazy"` para imagens abaixo da dobra** — nativo, sem JavaScript
- **Use `fetchpriority="high"` na imagem LCP** — indica ao navegador que priorize a maior imagem visível
- **Processe imagens no momento do upload** — não no momento do serving; pré-gere todos os tamanhos e formatos necessários
- **Use uma CDN com otimização de imagens** (Cloudflare Images, Imgix) — elas transformam sob demanda
- **Inclua um placeholder de baixa qualidade** (LQIP) — uma versão desfocada de 20px enquanto a imagem real carrega

---

## Anti-Padrões a Evitar

- Servir originais não otimizados da origem (exports de PSD de 100MB)
- URL de imagem única sem srcset — sempre 1200px no mobile
- `width="100%"` sem `aspect-ratio` ou altura explícita — causa layout shift
- Não fazer lazy loading em uma página com 50 imagens de produtos — todas carregam na renderização inicial
- JPEG para logos e ícones — use SVG; eles escalam sem perda de qualidade

---

## Debugging e Resolução de Problemas

**"As imagens estão causando layout shift (CLS)"**
Faltam atributos `width` e `height`. O navegador não sabe as dimensões da imagem até baixá-la. Defina dimensões explícitas ou use `aspect-ratio` no CSS.

**"O LCP está lento"**
Verifique se a imagem LCP tem `loading="lazy"` — remova. Adicione `fetchpriority="high"`. Adicione um `<link rel="preload">` no `<head>`.

**"O WebP não está sendo servido para o Chrome"**
Verifique se o `<source>` dentro do elemento `<picture>` tem `type="image/webp"`. Sem o atributo `type`, o navegador não sabe que é WebP e pode não usá-lo.

---

## Leituras Complementares

- [web.dev: Use modern image formats](https://web.dev/uses-webp-images/)
- [Documentação do Sharp](https://sharp.pixelplumbing.com/)
- [Guia de imagens responsivas — MDN](https://developer.mozilla.org/en-US/docs/Learn/HTML/Multimedia_and_embedding/Responsive_images)
- [Squoosh — ferramenta de compressão de imagens no navegador](https://squoosh.app/)
- Track 10: [Métricas de Performance Web](web-performance-metrics.md)

---

## Resumo

A otimização de imagens é a otimização de maior impacto para a maioria dos sites ricos em conteúdo. O pipeline é: upload → processamento com Sharp (múltiplos tamanhos × múltiplos formatos) → S3/storage → entrega via CDN. No HTML, use `<picture>` com fontes AVIF, WebP e JPEG mais `srcset` para tamanhos responsivos. Sempre defina atributos `width` e `height` para evitar CLS. Use `loading="lazy"` para imagens abaixo da dobra e `fetchpriority="high"` para a imagem LCP. Se você usa Next.js, o componente `<Image>` integrado cuida de todo o pipeline automaticamente.
