# Métricas de Performance Web

## Visão Geral

Você não pode melhorar o que não mede. As métricas de performance web traduzem a experiência do usuário em números precisos e objetivos. Os Core Web Vitals do Google são o conjunto de métricas padrão da indústria — eles se correlacionam diretamente com engajamento, taxas de conversão e posicionamento no SEO. Este capítulo explica o que cada métrica mede, como coletá-las e quais metas perseguir.

---

## Pré-requisitos

- Entendimento básico de como os navegadores carregam páginas
- Familiaridade com HTML, CSS e JavaScript
- Track 02 — Fundamentos de Frontend

---

## Conceitos Fundamentais

### Core Web Vitals (métricas atuais do Google)

| Métrica | O que mede | Bom | Precisa melhorar | Ruim |
|---------|-----------|------|------------------|------|
| **LCP** (Largest Contentful Paint) | Velocidade de carregamento — quando o maior elemento visível é renderizado | < 2,5s | 2,5–4,0s | > 4,0s |
| **INP** (Interaction to Next Paint) | Responsividade — atraso entre a ação do usuário e a atualização visual | < 200ms | 200–500ms | > 500ms |
| **CLS** (Cumulative Layout Shift) | Estabilidade visual — o quanto a página se move durante o carregamento | < 0,1 | 0,1–0,25 | > 0,25 |

O INP substituiu o FID (First Input Delay) em março de 2024. É uma medida mais abrangente de interatividade.

### Métricas adicionais importantes

| Métrica | Descrição |
|---------|-----------|
| **TTFB** (Time to First Byte) | Tempo de resposta do servidor — quanto tempo até o primeiro byte chegar |
| **FCP** (First Contentful Paint) | Quando qualquer conteúdo é renderizado pela primeira vez |
| **TTI** (Time to Interactive) | Quando a página se torna confiável para interação |
| **TBT** (Total Blocking Time) | Tempo total em que a thread principal está bloqueada (métrica de laboratório) |
| **Speed Index** | Com que rapidez o conteúdo é visualmente preenchido |

---

## Exemplos Práticos

### Medindo com Lighthouse (CLI)

```bash
# Instalar o Lighthouse CLI
npm install -g lighthouse

# Executar em uma URL
lighthouse https://your-site.com --output html --output-path report.html

# Executar em modo CI (sem navegador, retorna JSON)
lighthouse https://your-site.com --output json --chrome-flags="--headless" | jq '.categories.performance.score'
```

O Lighthouse simula um dispositivo móvel em 4G lento. Ele retorna uma pontuação de performance composta (0–100) e os valores individuais de cada métrica.

### Medindo com a Performance API (no navegador)

```typescript
// src/lib/performance-monitor.ts
export function collectWebVitals() {
  // LCP — quando o maior elemento é renderizado
  const lcpObserver = new PerformanceObserver((list) => {
    const entries = list.getEntries();
    const lastEntry = entries[entries.length - 1] as PerformanceEventTiming;

    reportMetric('LCP', lastEntry.startTime);
  });
  lcpObserver.observe({ type: 'largest-contentful-paint', buffered: true });

  // CLS — pontuação acumulada de layout shift
  let clsScore = 0;
  const clsObserver = new PerformanceObserver((list) => {
    for (const entry of list.getEntries() as PerformanceEntry[]) {
      if (!(entry as any).hadRecentInput) {
        clsScore += (entry as any).value;
      }
    }
    reportMetric('CLS', clsScore);
  });
  clsObserver.observe({ type: 'layout-shift', buffered: true });

  // INP — pior latência de interação
  let maxInp = 0;
  const inpObserver = new PerformanceObserver((list) => {
    for (const entry of list.getEntries() as PerformanceEventTiming[]) {
      if (entry.duration > maxInp) {
        maxInp = entry.duration;
        reportMetric('INP', maxInp);
      }
    }
  });
  inpObserver.observe({ type: 'event', buffered: true, durationThreshold: 16 });

  // TTFB — do início da navegação ao primeiro byte
  const [navEntry] = performance.getEntriesByType('navigation') as PerformanceNavigationTiming[];
  if (navEntry) {
    reportMetric('TTFB', navEntry.responseStart - navEntry.requestStart);
  }
}

function reportMetric(name: string, value: number) {
  // Envia para o sistema de analytics
  if (typeof navigator.sendBeacon === 'function') {
    navigator.sendBeacon('/api/metrics', JSON.stringify({ metric: name, value, url: location.href }));
  }
}
```

### Usando a biblioteca web-vitals (recomendado)

```bash
npm install web-vitals
```

```typescript
// src/lib/vitals.ts
import { onLCP, onINP, onCLS, onFCP, onTTFB, Metric } from 'web-vitals';

function sendToAnalytics(metric: Metric) {
  const body = JSON.stringify({
    name: metric.name,
    value: metric.value,
    rating: metric.rating, // 'good', 'needs-improvement', 'poor'
    delta: metric.delta,
    id: metric.id,
    url: location.href,
  });

  navigator.sendBeacon('/api/vitals', body);
}

export function initVitals() {
  onLCP(sendToAnalytics);
  onINP(sendToAnalytics);
  onCLS(sendToAnalytics);
  onFCP(sendToAnalytics);
  onTTFB(sendToAnalytics);
}
```

```typescript
// Em Next.js: app/layout.tsx
'use client';
import { useEffect } from 'react';
import { initVitals } from '../lib/vitals.js';

export function VitalsReporter() {
  useEffect(() => { initVitals(); }, []);
  return null;
}
```

### Endpoint de coleta de métricas no backend

```typescript
// src/routes/metrics.ts
import { z } from 'zod';

const VitalSchema = z.object({
  name: z.enum(['LCP', 'INP', 'CLS', 'FCP', 'TTFB']),
  value: z.number().nonnegative(),
  rating: z.enum(['good', 'needs-improvement', 'poor']),
  url: z.string().url(),
});

fastify.post('/api/vitals', async (request) => {
  const vital = VitalSchema.parse(JSON.parse(await request.body as string));

  // Armazena em banco de série temporal ou envia para provedor de analytics
  fastify.log.info({ vital }, 'Web vital recebido');

  // Envia para seu backend de analytics (Posthog, Amplitude, InfluxDB customizado, etc.)
  await analyticsClient.track('web_vital', vital);

  return { ok: true };
});
```

### Diagnosticando problemas de LCP

Elementos LCP comuns e suas correções:

```html
<!-- Imagem hero — elemento LCP mais comum -->
<!-- Ruim: com lazy loading (o navegador adia o carregamento) -->
<img src="hero.jpg" loading="lazy" alt="Hero" />

<!-- Bom: pré-carregado, sem lazy loading -->
<link rel="preload" href="hero.jpg" as="image" />
<img src="hero.jpg" loading="eager" fetchpriority="high" alt="Hero" />
```

```typescript
// Componente Image do Next.js com flag de prioridade
import Image from 'next/image';

function Hero() {
  return (
    <Image
      src="/hero.jpg"
      alt="Hero"
      width={1200}
      height={600}
      priority  // ← desativa lazy loading e adiciona preload link
    />
  );
}
```

### Diagnosticando problemas de CLS

Layout shifts acontecem quando elementos carregam após a renderização inicial e empurram outros conteúdos:

```css
/* Ruim — imagem sem dimensões causa layout shift quando carrega */
img { max-width: 100%; }

/* Bom — reserva espaço com aspect-ratio */
img {
  max-width: 100%;
  aspect-ratio: 16 / 9;
}

/* Ou use atributos width/height no HTML */
/* <img src="photo.jpg" width="800" height="450" alt="..."> */
```

```css
/* Slots de anúncios — reserve espaço para evitar shift quando o anúncio carrega */
.ad-slot {
  min-height: 250px; /* altura padrão de unidade de anúncio */
  contain: layout;
}
```

### Diagnosticando problemas de INP

O INP é causado por tarefas JavaScript longas que bloqueiam a thread principal:

```typescript
// Ruim — computação síncrona pesada na thread principal
function filterProducts(products: Product[], query: string) {
  return products
    .filter(p => expensiveSearch(p, query)) // bloqueia a thread principal por 500ms
    .sort(...);
}

// Bom — divide em chunks usando o scheduler
async function filterProductsAsync(products: Product[], query: string): Promise<Product[]> {
  const results: Product[] = [];
  const CHUNK_SIZE = 100;

  for (let i = 0; i < products.length; i += CHUNK_SIZE) {
    const chunk = products.slice(i, i + CHUNK_SIZE);
    results.push(...chunk.filter(p => expensiveSearch(p, query)));

    // Cede o controle ao navegador entre chunks para processar interações do usuário
    await new Promise(r => scheduler.postTask(r, { priority: 'user-blocking' }));
  }

  return results.sort(...);
}
```

---

## Padrões e Boas Práticas

- **Alvos de LCP:** imagem hero, texto acima da dobra, poster de vídeo — use `priority` ou `fetchpriority="high"`
- **Alvos de CLS:** sempre defina `width` e `height` em imagens; reserve espaço para anúncios e iframes
- **Alvos de INP:** quebre tarefas longas com `setTimeout(0)` ou `scheduler.postTask()`; evite computação pesada em interações
- **Alvos de TTFB:** resposta do servidor abaixo de 800ms; use CDN para conteúdo estático; otimize queries de banco nos caminhos críticos
- **Meça em campo, não apenas em laboratório** — o Lighthouse é sintético; o monitoramento de usuários reais captura condições reais

---

## Anti-Padrões a Evitar

- Medir apenas a página inicial — meça as páginas de alto tráfego em todo o site
- Tratar a pontuação do Lighthouse como única métrica — ela se correlaciona com dados de campo, mas não é idêntica
- Ignorar CLS em páginas com conteúdo dinâmico (anúncios, UI dependente de autenticação)
- Não coletar vitals de usuários reais — métricas de laboratório ignoram dispositivos e redes lentos

---

## Leituras Complementares

- [web.dev: Core Web Vitals](https://web.dev/vitals/)
- [Google Search Console: relatório Core Web Vitals](https://search.google.com/search-console/)
- [Documentação do Lighthouse](https://developer.chrome.com/docs/lighthouse/)
- [Biblioteca web-vitals](https://github.com/GoogleChrome/web-vitals)
- Track 10: [Otimização de JavaScript](javascript-optimization.md)

---

## Resumo

Os Core Web Vitals — LCP, INP e CLS — são as três métricas que definem uma boa experiência web e impactam diretamente o SEO. O LCP mede velocidade de carregamento, o INP mede responsividade e o CLS mede estabilidade visual. Meça em campo com a biblioteca `web-vitals` e colete os resultados em um backend de analytics. Ferramentas de laboratório como o Lighthouse são úteis para desenvolvimento, mas não substituem o monitoramento de usuários reais. Cada métrica tem um conjunto distinto de causas e correções: LCP responde ao pré-carregamento e otimização de imagens; CLS responde à reserva de espaço para conteúdo dinâmico; INP responde à quebra de tarefas JavaScript longas.
