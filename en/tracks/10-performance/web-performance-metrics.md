# Web Performance Metrics

## Overview

You cannot improve what you do not measure. Web performance metrics give precise, objective numbers to the user experience. Google's Core Web Vitals are the industry-standard set of metrics — they directly correlate with user engagement, conversion rates, and SEO rankings. This chapter explains what each metric measures, how to collect them, and what targets to aim for.

---

## Prerequisites

- Basic understanding of how browsers load pages
- Familiarity with HTML, CSS, and JavaScript
- Track 02 — Frontend basics

---

## Core Concepts

### Core Web Vitals (Google's current metrics)

| Metric | What it measures | Good | Needs improvement | Poor |
|--------|-----------------|------|-------------------|------|
| **LCP** (Largest Contentful Paint) | Loading speed — when the largest visible element renders | < 2.5s | 2.5–4.0s | > 4.0s |
| **INP** (Interaction to Next Paint) | Responsiveness — delay from user input to visual update | < 200ms | 200–500ms | > 500ms |
| **CLS** (Cumulative Layout Shift) | Visual stability — how much the page shifts during load | < 0.1 | 0.1–0.25 | > 0.25 |

INP replaced FID (First Input Delay) in March 2024. It is a more comprehensive measure of interactivity.

### Additional important metrics

| Metric | Description |
|--------|-------------|
| **TTFB** (Time to First Byte) | Server response time — how long before the first byte arrives |
| **FCP** (First Contentful Paint) | When any content is first rendered |
| **TTI** (Time to Interactive) | When the page becomes reliably interactive |
| **TBT** (Total Blocking Time) | Total time the main thread is blocked (lab metric) |
| **Speed Index** | How quickly content is visually populated |

---

## Hands-On Examples

### Measuring with Lighthouse (CLI)

```bash
# Install Lighthouse CLI
npm install -g lighthouse

# Run against a URL
lighthouse https://your-site.com --output html --output-path report.html

# Run in CI mode (no browser needed, returns JSON)
lighthouse https://your-site.com --output json --chrome-flags="--headless" | jq '.categories.performance.score'
```

Lighthouse runs a simulated mobile device at slow 4G. It gives a composite performance score (0–100) and individual metric values.

### Measuring with the Performance API (in-browser)

```typescript
// src/lib/performance-monitor.ts
export function collectWebVitals() {
  // LCP — when the largest element renders
  const lcpObserver = new PerformanceObserver((list) => {
    const entries = list.getEntries();
    const lastEntry = entries[entries.length - 1] as PerformanceEventTiming;

    reportMetric('LCP', lastEntry.startTime);
  });
  lcpObserver.observe({ type: 'largest-contentful-paint', buffered: true });

  // CLS — accumulated layout shift score
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

  // INP — worst interaction latency
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

  // TTFB — from navigation start to first byte
  const [navEntry] = performance.getEntriesByType('navigation') as PerformanceNavigationTiming[];
  if (navEntry) {
    reportMetric('TTFB', navEntry.responseStart - navEntry.requestStart);
  }
}

function reportMetric(name: string, value: number) {
  // Send to analytics
  if (typeof navigator.sendBeacon === 'function') {
    navigator.sendBeacon('/api/metrics', JSON.stringify({ metric: name, value, url: location.href }));
  }
}
```

### Using web-vitals library (recommended)

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
// In Next.js: app/layout.tsx
'use client';
import { useEffect } from 'react';
import { initVitals } from '../lib/vitals.js';

export function VitalsReporter() {
  useEffect(() => { initVitals(); }, []);
  return null;
}
```

### Backend metrics collection endpoint

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

  // Store in time-series database or send to analytics provider
  fastify.log.info({ vital }, 'Web vital received');

  // Send to your analytics backend (Posthog, Amplitude, custom InfluxDB, etc.)
  await analyticsClient.track('web_vital', vital);

  return { ok: true };
});
```

### Diagnosing LCP issues

Common LCP elements and fixes:

```html
<!-- Hero image — most common LCP element -->
<!-- Bad: lazy loaded (browser delays loading) -->
<img src="hero.jpg" loading="lazy" alt="Hero" />

<!-- Good: preloaded, not lazy -->
<link rel="preload" href="hero.jpg" as="image" />
<img src="hero.jpg" loading="eager" fetchpriority="high" alt="Hero" />
```

```typescript
// Next.js Image component with priority flag
import Image from 'next/image';

function Hero() {
  return (
    <Image
      src="/hero.jpg"
      alt="Hero"
      width={1200}
      height={600}
      priority  // ← disables lazy loading, adds preload link
    />
  );
}
```

### Diagnosing CLS issues

Layout shifts happen when elements load after the initial render and push other content:

```css
/* Bad — image with no dimensions causes layout shift when it loads */
img { max-width: 100%; }

/* Good — reserve space with aspect-ratio */
img {
  max-width: 100%;
  aspect-ratio: 16 / 9;
}

/* Or use width/height attributes in HTML */
/* <img src="photo.jpg" width="800" height="450" alt="..."> */
```

```css
/* Ad slots — reserve space to prevent shift when ad loads */
.ad-slot {
  min-height: 250px; /* standard ad unit height */
  contain: layout;
}
```

### Diagnosing INP issues

INP is caused by long JavaScript tasks that block the main thread:

```typescript
// Bad — long synchronous computation on main thread
function filterProducts(products: Product[], query: string) {
  return products
    .filter(p => expensiveSearch(p, query)) // blocks main thread for 500ms
    .sort(...);
}

// Good — break into chunks using scheduler
async function filterProductsAsync(products: Product[], query: string): Promise<Product[]> {
  const results: Product[] = [];
  const CHUNK_SIZE = 100;

  for (let i = 0; i < products.length; i += CHUNK_SIZE) {
    const chunk = products.slice(i, i + CHUNK_SIZE);
    results.push(...chunk.filter(p => expensiveSearch(p, query)));

    // Yield to browser between chunks to process user input
    await new Promise(r => scheduler.postTask(r, { priority: 'user-blocking' }));
  }

  return results.sort(...);
}
```

---

## Common Patterns & Best Practices

- **LCP targets:** hero image, above-the-fold text, video poster — use `priority` or `fetchpriority="high"`
- **CLS targets:** always set `width` and `height` on images; reserve space for ads and iframes
- **INP targets:** break long tasks with `setTimeout(0)` or `scheduler.postTask()`; avoid heavy computation on interaction
- **TTFB targets:** server response under 800ms; use CDN for static content; optimize DB queries on critical paths
- **Measure in the field, not just lab** — Lighthouse is synthetic; real user monitoring captures actual conditions

---

## Anti-Patterns to Avoid

- Measuring only the homepage — measure high-traffic pages across the site
- Treating Lighthouse score as the only metric — it correlates with field data but is not identical
- Ignoring CLS on pages with dynamic content (ads, auth-dependent UI)
- Not collecting vitals from real users — lab metrics miss slow devices and networks

---

## Further Reading

- [web.dev: Core Web Vitals](https://web.dev/vitals/)
- [Google Search Console: Core Web Vitals report](https://search.google.com/search-console/)
- [Lighthouse documentation](https://developer.chrome.com/docs/lighthouse/)
- [web-vitals library](https://github.com/GoogleChrome/web-vitals)
- Track 10: [JavaScript Optimization](javascript-optimization.md)

---

## Summary

Core Web Vitals — LCP, INP, and CLS — are the three metrics that define a good web experience and directly impact SEO. LCP measures loading speed, INP measures responsiveness, and CLS measures visual stability. Measure in the field with the `web-vitals` library and collect results in an analytics backend. Lab tools (Lighthouse) are useful for development but do not replace real user monitoring. Each metric has a distinct set of causes and fixes: LCP responds to preloading and image optimization; CLS responds to reserving space for dynamic content; INP responds to breaking long JavaScript tasks.
