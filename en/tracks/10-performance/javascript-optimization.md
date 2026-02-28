# JavaScript Optimization

## Overview

JavaScript performance has two domains: how quickly the browser downloads and parses your bundles (load performance), and how quickly your code executes at runtime (runtime performance). This chapter covers both: bundle optimization with modern build tools, tree-shaking, code splitting, lazy loading, and runtime patterns that keep the main thread responsive.

---

## Prerequisites

- JavaScript/TypeScript fundamentals
- Basic understanding of module bundlers (Vite, webpack)
- Track 02 — Frontend: React basics

---

## Hands-On Examples

### Bundle analysis — finding what's large

```bash
# Vite bundle visualization
npm install --save-dev rollup-plugin-visualizer

# vite.config.ts
import { visualizer } from 'rollup-plugin-visualizer';
export default {
  plugins: [
    visualizer({ open: true, gzipSize: true, filename: 'bundle-stats.html' }),
  ],
};

# Build and open the treemap
npm run build
```

The treemap shows which modules contribute most to bundle size. Common culprits: `moment.js` (replace with `date-fns`), `lodash` (use individual imports or native methods), large icon libraries (use SVG sprites or lazy load).

### Tree-shaking — eliminating dead code

```typescript
// Bad — imports entire lodash (530kB!)
import _ from 'lodash';
const unique = _.uniq(arr);

// Good — import only what you need (cherry-pick)
import { uniq } from 'lodash-es'; // ES module version is tree-shakeable

// Best — use native equivalent (zero bundle cost)
const unique = [...new Set(arr)];
```

```typescript
// Bad — barrel files often break tree-shaking
// src/utils/index.ts
export * from './format.js';
export * from './validate.js';
export * from './crypto.js'; // 50kB — imported by everyone now

// Good — import directly
import { formatDate } from './utils/format.js'; // only format.js is bundled
```

### Code splitting — splitting by route

```typescript
// React Router with lazy loading
import { lazy, Suspense } from 'react';
import { Routes, Route } from 'react-router-dom';

// Each page is a separate chunk — only loaded when navigated to
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings = lazy(() => import('./pages/Settings'));
const AdminPanel = lazy(() => import('./pages/admin/Panel'));

export function App() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
        <Route path="/admin" element={<AdminPanel />} />
      </Routes>
    </Suspense>
  );
}
```

### Dynamic imports — lazy loading heavy components

```typescript
// Load a heavy chart library only when the user navigates to a chart page
import { useState, useEffect } from 'react';

function ChartPage() {
  const [ChartComponent, setChartComponent] = useState<React.ComponentType | null>(null);

  useEffect(() => {
    // Loads the chunk only when this component mounts
    import('./components/HeavyChart').then((module) => {
      setChartComponent(() => module.HeavyChart);
    });
  }, []);

  if (!ChartComponent) return <Skeleton />;
  return <ChartComponent />;
}
```

```typescript
// Or with React.lazy — cleaner syntax
const HeavyChart = lazy(() => import('./components/HeavyChart'));

// Only loaded when <HeavyChart /> is rendered
function ChartPage() {
  const [show, setShow] = useState(false);
  return (
    <>
      <button onClick={() => setShow(true)}>Load Chart</button>
      {show && (
        <Suspense fallback={<Skeleton />}>
          <HeavyChart />
        </Suspense>
      )}
    </>
  );
}
```

### Preloading critical chunks

```typescript
// Preload a chunk that will be needed soon (e.g., on hover)
function NavLink({ href, children }: { href: string; children: React.ReactNode }) {
  const handleMouseEnter = () => {
    // User is about to click — start loading the chunk
    import('./pages/Dashboard'); // triggers download without executing
  };

  return (
    <a href={href} onMouseEnter={handleMouseEnter}>
      {children}
    </a>
  );
}
```

### Avoiding layout thrashing

Layout thrashing occurs when you read layout properties (width, height, offsetTop) and write DOM properties in an alternating loop:

```typescript
// Bad — causes layout thrashing
function expandCards(cards: HTMLElement[]) {
  cards.forEach((card) => {
    const height = card.offsetHeight; // FORCED LAYOUT (read)
    card.style.height = `${height + 50}px`; // write
    // Next iteration: read forces recalculation because of the previous write
  });
}

// Good — batch reads, then batch writes
function expandCardsOptimized(cards: HTMLElement[]) {
  // Read phase (all layout reads in one pass)
  const heights = cards.map((card) => card.offsetHeight);

  // Write phase (all DOM writes after all reads)
  cards.forEach((card, i) => {
    card.style.height = `${heights[i] + 50}px`;
  });
}
```

### Debouncing and throttling event handlers

```typescript
// Search input — debounce to avoid firing on every keystroke
function SearchInput() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  const search = useMemo(
    () =>
      debounce(async (q: string) => {
        if (q.length < 2) return;
        const data = await fetch(`/api/search?q=${encodeURIComponent(q)}`).then(r => r.json());
        setResults(data);
      }, 300),
    []
  );

  return (
    <input
      value={query}
      onChange={(e) => {
        setQuery(e.target.value);
        search(e.target.value);
      }}
    />
  );
}

// Scroll handler — throttle to 60fps
function useScrollThrottle(handler: (y: number) => void) {
  useEffect(() => {
    let ticking = false;
    const onScroll = () => {
      if (!ticking) {
        requestAnimationFrame(() => {
          handler(window.scrollY);
          ticking = false;
        });
        ticking = true;
      }
    };
    window.addEventListener('scroll', onScroll, { passive: true });
    return () => window.removeEventListener('scroll', onScroll);
  }, [handler]);
}
```

### Moving heavy computation to a Web Worker

```typescript
// src/workers/filter.worker.ts
self.onmessage = (event: MessageEvent<{ products: Product[]; query: string }>) => {
  const { products, query } = event.data;
  const result = products.filter((p) => p.name.toLowerCase().includes(query.toLowerCase()));
  self.postMessage(result);
};

// Usage in component
function useWorkerFilter(products: Product[], query: string) {
  const [filtered, setFiltered] = useState<Product[]>(products);
  const workerRef = useRef<Worker | null>(null);

  useEffect(() => {
    workerRef.current = new Worker(new URL('./workers/filter.worker.ts', import.meta.url), {
      type: 'module',
    });
    workerRef.current.onmessage = (e) => setFiltered(e.data);
    return () => workerRef.current?.terminate();
  }, []);

  useEffect(() => {
    workerRef.current?.postMessage({ products, query });
  }, [products, query]);

  return filtered;
}
```

### React performance patterns

```typescript
// Memoize expensive computations
const sortedProducts = useMemo(
  () => [...products].sort((a, b) => a.price - b.price),
  [products] // only recompute when products changes
);

// Memoize components that receive the same props
const ProductCard = React.memo(function ProductCard({ product }: { product: Product }) {
  return <div>{product.name}</div>;
});

// Stable callback references
const handleClick = useCallback((id: string) => {
  addToCart(id);
}, [addToCart]); // addToCart must also be stable
```

---

## Common Patterns & Best Practices

- **Analyze your bundle before optimizing** — the visualizer shows where to focus effort
- **Use native APIs over libraries** where possible — `Array.filter`, `Set`, `fetch`, `Intl`
- **Code-split by route** — users only download code for pages they visit
- **Lazy-load heavy components** — PDF viewers, chart libraries, rich text editors
- **Use `passive: true` on scroll/touch listeners** — tells the browser not to wait for your handler before scrolling
- **Measure with DevTools Performance panel** — identify specific functions causing long tasks

---

## Anti-Patterns to Avoid

- Importing entire libraries when you use one function (`moment`, `lodash`, full icon sets)
- Barrel files that re-export everything — often break tree-shaking
- Heavy computation on the main thread — use Web Workers or schedule with `scheduler.postTask`
- Synchronous DOM reads inside animation loops — use `requestAnimationFrame` and batch reads/writes

---

## Debugging & Troubleshooting

**"My bundle is 2MB and I don't know why"**
Run the visualizer. Look for `node_modules` that are unexpectedly large. Common surprises: `moment` (240kB), `lodash` (530kB), `date-fns` loaded in full rather than cherry-picked, a whole icon library loaded for 3 icons.

**"The page is janky during scroll"**
Open DevTools → Performance → record while scrolling. Look for long frames (red in the timeline). Long-running event handlers or layout thrashing are the usual culprits.

**"React re-renders are too frequent"**
Use React DevTools Profiler to record interactions and see which components re-render and why. Add `React.memo`, `useMemo`, and `useCallback` strategically after profiling — not speculatively.

---

## Further Reading

- [web.dev: Optimize JavaScript execution](https://web.dev/optimize-javascript-execution/)
- [Chrome DevTools: Analyze runtime performance](https://developer.chrome.com/docs/devtools/performance/)
- [Vite: build optimization](https://vitejs.dev/guide/features.html#build-optimizations)
- Track 10: [Profiling Tools](profiling-tools.md)
- Track 10: [Web Performance Metrics](web-performance-metrics.md)

---

## Summary

JavaScript optimization operates at two levels: bundle size (how much code the browser downloads) and runtime performance (how efficiently that code runs). Bundle analysis identifies oversized dependencies; tree-shaking and code-splitting eliminate or defer them. Runtime optimization focuses on keeping the main thread unblocked: debounce event handlers, batch DOM reads and writes, move heavy computation to Web Workers, and use `requestAnimationFrame` for visual updates. Always profile before optimizing — the bottleneck is rarely where you expect it to be.
