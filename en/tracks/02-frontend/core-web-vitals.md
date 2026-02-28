# Core Web Vitals

## 1. What & Why

Core Web Vitals are a set of metrics defined by Google to measure the real-world user experience of a web page. They are not academic benchmarks — they are direct signals in Google's ranking algorithm. A page that scores poorly on Core Web Vitals ranks lower in search results.

More importantly, they are good proxies for what users actually feel:

- **LCP** — Does the main content load fast?
- **INP** — Does the page respond when I interact with it?
- **CLS** — Does the page jump around while loading?

These three questions correspond to: loading, interactivity, and visual stability. A page that scores well on all three feels fast and polished. A page that scores poorly feels broken — even if it "works."

**Why developers should care:** Most performance improvements are architectural decisions, not micro-optimizations. Understanding what each metric measures, what causes poor scores, and how to fix them is frontend engineering at its most impactful.

---

## 2. Core Concepts

| Metric | Measures | Target | Poor |
|--------|----------|--------|------|
| **LCP** — Largest Contentful Paint | Loading performance | ≤ 2.5s | > 4.0s |
| **INP** — Interaction to Next Paint | Responsiveness | ≤ 200ms | > 500ms |
| **CLS** — Cumulative Layout Shift | Visual stability | ≤ 0.1 | > 0.25 |

"Good" means the 75th percentile of page loads (real user data) meets the target. Google evaluates the p75, not the average.

---

## 3. How It Works

### LCP — Largest Contentful Paint

LCP measures the time from page navigation start to when the **largest visible content element** finishes rendering. The browser considers these elements:

- `<img>` elements
- `<video>` elements with a poster image
- Elements with a CSS `background-image`
- Block-level elements containing significant text

The LCP element is usually the hero image, a heading, or the main content block — whatever is the biggest visible thing on initial load.

**LCP timeline:**
```
Navigation start
    ↓
DNS lookup → TCP connection → TLS handshake → HTTP request → Server response (TTFB)
    ↓
HTML parsing → Resource discovery → CSS/JS execution → Layout → Paint
    ↓
LCP: largest element finishes rendering ← this is what we measure
```

### INP — Interaction to Next Paint

INP replaced FID (First Input Delay) as a Core Web Vital in March 2024. The key difference:

- **FID** measured the delay before the browser could start processing the *first* interaction
- **INP** measures the full duration of *all* interactions throughout the page lifetime — from click/keypress to the next visual update

INP captures the worst latency of: `input delay` (waiting for main thread) + `processing time` (event handler execution) + `presentation delay` (browser rendering the result).

A poor INP means: user clicks a button, nothing happens visually for 500ms. The browser was busy with a long task.

### CLS — Cumulative Layout Shift

CLS measures how much visible content unexpectedly moves during the page's lifetime. Each layout shift is scored by: `impact fraction × distance fraction`. The CLS score is the sum of the largest burst of layout shifts within a 5-second window.

A layout shift happens when a DOM element moves to a different position between renders without user interaction. Common causes:

- Images without `width` and `height` attributes (browser doesn't reserve space)
- Ads, embeds, or iframes without fixed dimensions
- Dynamic content injection above existing content
- Web fonts causing FOUT (Flash of Unstyled Text) that changes text metrics

---

## 4. Code Examples

### Measuring Web Vitals

```ts
// Install: npm install web-vitals

// src/lib/web-vitals.ts
import { onCLS, onINP, onLCP, onFCP, onTTFB } from "web-vitals";

type MetricName = "CLS" | "INP" | "LCP" | "FCP" | "TTFB";

interface Metric {
  name: MetricName;
  value: number;
  rating: "good" | "needs-improvement" | "poor";
  id: string;
}

function sendToAnalytics(metric: Metric) {
  // Send to your analytics service
  // Google Analytics 4:
  if (typeof window.gtag !== "undefined") {
    window.gtag("event", metric.name, {
      value: Math.round(metric.name === "CLS" ? metric.value * 1000 : metric.value),
      metric_id: metric.id,
      metric_value: metric.value,
      metric_rating: metric.rating,
      non_interaction: true,
    });
  }

  // Or to a custom endpoint:
  if (navigator.sendBeacon) {
    navigator.sendBeacon("/api/metrics", JSON.stringify(metric));
  }
}

export function initWebVitals() {
  onCLS(sendToAnalytics);
  onINP(sendToAnalytics);
  onLCP(sendToAnalytics);
  onFCP(sendToAnalytics);
  onTTFB(sendToAnalytics);
}

// In Next.js — use the built-in hook
// app/layout.tsx or pages/_app.tsx
export function reportWebVitals(metric: { name: string; value: number }) {
  console.log(metric); // or send to analytics
}
```

```ts
// PerformanceObserver API — observe LCP in real time
const observer = new PerformanceObserver((list) => {
  const entries = list.getEntries();
  const lastEntry = entries[entries.length - 1] as PerformancePaintTiming;
  console.log("LCP candidate:", lastEntry.startTime, lastEntry.element);
});

observer.observe({ type: "largest-contentful-paint", buffered: true });

// Observe layout shifts for CLS
const clsObserver = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    const layoutShift = entry as PerformanceEntry & { value: number; hadRecentInput: boolean };
    if (!layoutShift.hadRecentInput) {
      console.log("Layout shift:", layoutShift.value, entry);
    }
  }
});
clsObserver.observe({ type: "layout-shift", buffered: true });

// Observe INP
const inpObserver = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    const interaction = entry as PerformanceEventTiming;
    const duration = interaction.processingEnd - interaction.startTime;
    if (duration > 200) {
      console.warn("Slow interaction:", interaction.name, `${duration.toFixed(0)}ms`);
    }
  }
});
inpObserver.observe({ type: "event", durationThreshold: 16, buffered: true });
```

### Fixing LCP

```tsx
// Problem 1: Hero image not preloaded — browser discovers it late in parsing

// WRONG — image loads only after CSS and JS parse
<img src="/hero.jpg" alt="Hero" />

// CORRECT — preload in <head> so the browser fetches it immediately
// In Next.js app/layout.tsx:
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <head>
        <link
          rel="preload"
          href="/hero.jpg"
          as="image"
          fetchPriority="high"
        />
      </head>
      <body>{children}</body>
    </html>
  );
}

// Or in Next.js with next/image — priority prop adds preload automatically:
import Image from "next/image";

function Hero() {
  return (
    <Image
      src="/hero.jpg"
      alt="Hero image"
      width={1200}
      height={600}
      priority          // adds <link rel="preload"> and fetchPriority="high"
      quality={85}      // balance quality and file size
      sizes="100vw"     // tells browser the image fills the viewport
    />
  );
}
```

```tsx
// Problem 2: Images not optimized — large files slow TTFB + download

// WRONG — serving a 4MB JPEG for a 400px thumbnail
<img src="/product.jpg" width={400} height={300} />

// CORRECT — next/image automatically:
// - Converts to WebP/AVIF (smaller file sizes)
// - Generates srcset for different viewport sizes
// - Lazy loads by default (set priority for LCP element)
import Image from "next/image";

function ProductCard({ product }: { product: { imageUrl: string; name: string } }) {
  return (
    <Image
      src={product.imageUrl}
      alt={product.name}
      width={400}
      height={300}
      sizes="(max-width: 640px) 100vw, (max-width: 1024px) 50vw, 400px"
    />
  );
}
```

```ts
// Problem 3: Slow TTFB — server response is slow before any rendering starts

// Solutions (infrastructure, not client-side):
// 1. Use a CDN to serve static assets and cached responses
// 2. Cache rendered pages (ISR, Varnish, CloudFront)
// 3. Move computation to edge (Next.js Edge Runtime, Cloudflare Workers)
// 4. Optimize database queries that block the initial render
// 5. Enable HTTP/2 or HTTP/3 (multiplexed requests)

// In Next.js — measure TTFB
import { onTTFB } from "web-vitals";
onTTFB((metric) => {
  if (metric.value > 600) {
    console.warn(`Slow TTFB: ${metric.value.toFixed(0)}ms — investigate server response time`);
  }
});
```

### Fixing INP

```ts
// Problem 1: Long tasks block the main thread during interactions
// The browser can't render the next frame until the current task finishes

// WRONG — synchronous, blocks main thread for the entire computation
function handleSearch(query: string) {
  const results = products.filter((p) => p.name.includes(query)); // cheap
  const enriched = results.map(expensiveEnrich); // takes 400ms
  setResults(enriched); // user sees nothing for 400ms
}

// CORRECT — break up long work into smaller chunks
async function handleSearchOptimized(query: string) {
  const results = products.filter((p) => p.name.includes(query));

  // Use scheduler.postTask (Chrome only) or setTimeout(0) to yield to browser
  const enriched = await new Promise<typeof results>((resolve) => {
    const CHUNK_SIZE = 50;
    const chunks: typeof results = [];
    let index = 0;

    function processChunk() {
      const end = Math.min(index + CHUNK_SIZE, results.length);
      for (; index < end; index++) {
        chunks.push(expensiveEnrich(results[index]));
      }
      if (index < results.length) {
        // Yield to browser between chunks — allows rendering
        setTimeout(processChunk, 0);
      } else {
        resolve(chunks);
      }
    }

    processChunk();
  });

  setResults(enriched);
}

// Even simpler with scheduler.postTask (where available)
async function processWithScheduler<T>(items: T[], fn: (item: T) => T): Promise<T[]> {
  const results: T[] = [];
  for (const item of items) {
    results.push(fn(item));
    // Yield to browser every item (or every N items for performance)
    if ("scheduler" in window) {
      await (window as Window & { scheduler: { postTask: (fn: () => void) => Promise<void> } }).scheduler.postTask(() => {}, { priority: "background" });
    }
  }
  return results;
}
```

```ts
// Problem 2: Layout thrashing — reading and writing DOM in alternation

// WRONG — forces browser to recalculate layout repeatedly (reflow per element)
function resizeElements(elements: HTMLElement[], targetWidth: number) {
  elements.forEach((el) => {
    const currentWidth = el.offsetWidth; // read → forces layout
    el.style.width = `${targetWidth - currentWidth / 2}px`; // write
    // Next iteration: read (forces layout again) → write → read → write...
  });
}

// CORRECT — batch all reads, then all writes
function resizeElementsOptimized(elements: HTMLElement[], targetWidth: number) {
  // Phase 1: read all layout values (one layout calculation)
  const widths = elements.map((el) => el.offsetWidth);

  // Phase 2: write all style changes (browser can batch into one reflow)
  elements.forEach((el, i) => {
    el.style.width = `${targetWidth - widths[i] / 2}px`;
  });
}

// Using requestAnimationFrame to sync with browser's render cycle
function animateWithRAF(el: HTMLElement, targetX: number) {
  let currentX = 0;

  function step() {
    currentX += (targetX - currentX) * 0.1; // ease
    el.style.transform = `translateX(${currentX}px)`; // GPU-composited, no reflow

    if (Math.abs(targetX - currentX) > 0.1) {
      requestAnimationFrame(step);
    }
  }

  requestAnimationFrame(step);
}
```

```tsx
// Problem 3: React re-renders on every keystroke slow down input response

// React solution: useTransition to mark state updates as non-urgent
import { useState, useTransition, useDeferredValue } from "react";

function SearchInput({ items }: { items: string[] }) {
  const [query, setQuery] = useState("");
  const [isPending, startTransition] = useTransition();

  const deferredQuery = useDeferredValue(query);

  // Expensive computation runs with deferred (lower priority) value
  const filtered = items.filter((item) =>
    item.toLowerCase().includes(deferredQuery.toLowerCase())
  );

  return (
    <div>
      <input
        value={query}
        onChange={(e) => {
          setQuery(e.target.value); // urgent: input stays responsive
          // No startTransition needed — useDeferredValue handles deferral
        }}
      />
      {isPending && <span aria-live="polite">Updating…</span>}
      <ul style={{ opacity: query !== deferredQuery ? 0.7 : 1 }}>
        {filtered.map((item) => <li key={item}>{item}</li>)}
      </ul>
    </div>
  );
}
```

### Fixing CLS

```tsx
// Problem 1: Images without dimensions — browser doesn't know how tall they are
// until they load, then the page jumps

// WRONG — no dimensions
<img src="/product.jpg" alt="Product" />

// CORRECT — explicit dimensions (browser reserves space immediately)
<img src="/product.jpg" alt="Product" width={400} height={300} />

// Or use aspect-ratio CSS to reserve space responsively:
// CSS:
// img { width: 100%; aspect-ratio: 4 / 3; }

// next/image handles this automatically — always requires width/height (or fill)
import Image from "next/image";
<Image src="/product.jpg" alt="Product" width={400} height={300} />
```

```tsx
// Problem 2: Dynamic content inserted above existing content causes a shift

// WRONG — notification banner inserted above page content
function Page() {
  const [showBanner, setShowBanner] = useState(false);

  useEffect(() => {
    // Banner appears after data loads → pushes content down → CLS!
    checkForPromotion().then((hasPromo) => setShowBanner(hasPromo));
  }, []);

  return (
    <div>
      {showBanner && <PromoBanner />} {/* shifts everything below it */}
      <main>Content...</main>
    </div>
  );
}

// CORRECT — reserve space for the banner even when not yet shown
function PageFixed() {
  const [showBanner, setShowBanner] = useState(false);

  useEffect(() => {
    checkForPromotion().then((hasPromo) => setShowBanner(hasPromo));
  }, []);

  return (
    <div>
      {/* Always reserve the space; show content when ready */}
      <div style={{ minHeight: 56 }}>
        {showBanner && <PromoBanner />}
      </div>
      <main>Content...</main>
    </div>
  );
}
```

```css
/* Problem 3: Web fonts cause FOUT/FOIT — text jumps when font loads */

/* WRONG — FOUT: text renders in fallback font, then jumps to web font */
@font-face {
  font-family: "Geist";
  src: url("/fonts/Geist.woff2") format("woff2");
  font-display: swap; /* show fallback immediately, swap when loaded — causes CLS */
}

/* BETTER — font-display: optional: if font doesn't load quickly, use fallback */
/* No visible swap = no CLS */
@font-face {
  font-family: "Geist";
  src: url("/fonts/Geist.woff2") format("woff2");
  font-display: optional;
}

/* BEST — size-adjust: tune the fallback font metrics to match the web font */
/* Reduces/eliminates the visual difference between fallback and web font */
@font-face {
  font-family: "Geist-fallback";
  src: local("Arial");
  size-adjust: 106%;
  ascent-override: 95%;
  descent-override: normal;
  line-gap-override: normal;
}

/* Next.js next/font does this automatically */
```

```tsx
// next/font — zero CLS font loading in Next.js
import { Inter, Geist } from "next/font/google";

// next/font:
// 1. Downloads fonts at build time (no runtime network request)
// 2. Self-hosts fonts (GDPR compliant)
// 3. Generates size-adjust CSS to prevent layout shift
// 4. Uses font-display: optional by default

const geist = Geist({
  subsets: ["latin"],
  display: "swap",         // or "optional" for zero CLS
  variable: "--font-geist", // CSS custom property
});

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={geist.variable}>
      <body>{children}</body>
    </html>
  );
}
```

```tsx
// Problem 4: Skeleton screens prevent CLS from dynamic content
// Instead of inserting content after load, render placeholder first

function ProductGrid({ products }: { products: Product[] | null }) {
  if (!products) {
    // Skeleton takes up the same space as real content
    return (
      <div className="grid grid-cols-3 gap-4">
        {Array.from({ length: 6 }, (_, i) => (
          <div
            key={i}
            className="skeleton"
            style={{ height: 280 }}  // must match real card height
            aria-busy="true"
            aria-label="Loading product"
          />
        ))}
      </div>
    );
  }

  return (
    <div className="grid grid-cols-3 gap-4">
      {products.map((p) => <ProductCard key={p.id} product={p} />)}
    </div>
  );
}
```

---

## 5. Common Mistakes & Pitfalls

**1. Setting priority on every image (defeats the purpose)**
```tsx
// WRONG — priority tells the browser to preload the image; too many defeats it
<Image src="/hero.jpg" priority />
<Image src="/product-1.jpg" priority />
<Image src="/product-2.jpg" priority />
// The browser ends up preloading 10 images simultaneously — bandwidth congestion

// CORRECT — priority only on the LCP element (usually hero or above-fold content)
<Image src="/hero.jpg" priority />
<Image src="/product-1.jpg" />
<Image src="/product-2.jpg" />
```

**2. Measuring in DevTools throttled mode inconsistently**
```
Chrome DevTools simulates slow networks but not slow CPUs equally.
"Slow 3G" is more realistic for INP testing.
Always measure Core Web Vitals from real user data (CrUX, web-vitals library)
— lab scores (Lighthouse) can differ significantly from field data.
```

**3. layout-thrashing in event handlers**
```ts
// This pattern causes layout thrashing → poor INP
document.addEventListener("mousemove", (e) => {
  elements.forEach((el) => {
    const rect = el.getBoundingClientRect(); // read → reflow
    el.style.left = `${e.clientX - rect.width / 2}px`; // write
    // 100 elements × read/write = 100 reflows per mousemove event
  });
});
```

**4. Not setting width/height on video poster**
```html
<!-- WRONG — browser doesn't know video dimensions until it loads -->
<video src="/promo.mp4" autoplay muted></video>

<!-- CORRECT — tell the browser the dimensions upfront -->
<video src="/promo.mp4" width="1280" height="720" autoplay muted></video>
```

**5. Third-party scripts causing long tasks (INP)**
```ts
// Analytics, chat widgets, and ad scripts often run long tasks that block interactions

// WRONG — load immediately
<script src="https://cdn.analytics.com/tracker.js"></script>

// CORRECT — defer or lazy load
<script src="https://cdn.analytics.com/tracker.js" defer></script>
// Or: load after user interaction (first click or scroll)
window.addEventListener("scroll", () => {
  const script = document.createElement("script");
  script.src = "https://cdn.analytics.com/tracker.js";
  document.head.appendChild(script);
}, { once: true });
```

---

## 6. When to Use / Not Use

| Optimization | Use when | Notes |
|-------------|----------|-------|
| `priority` on Image | LCP element (hero, above-fold image) | Only one or two per page |
| `next/image` | Any `<img>` element | Almost always — handles srcset, WebP, lazy load |
| `font-display: optional` | Any web font | Prevents CLS; uses fallback if font doesn't load fast |
| `useTransition` | Expensive React renders on interaction | Input stays responsive; list renders deferred |
| Skeleton screens | Async-loaded content sections | Must match exact dimensions of real content |
| `fetchPriority="high"` | LCP image loaded via CSS background or when `priority` isn't available | Native hint to browser |
| `preconnect` | Third-party domains the page will fetch from (fonts, CDN, APIs) | `<link rel="preconnect" href="https://fonts.gstatic.com" />` |

---

## 7. Real-World Scenario

Diagnosing and fixing a Next.js product page with poor Core Web Vitals.

**Initial scores (simulated):**
- LCP: 4.2s (Poor) — hero image loaded late
- INP: 380ms (Poor) — product filter rerenders all 500 items synchronously
- CLS: 0.28 (Poor) — product images without dimensions, promo banner loads late

**Fix 1 — LCP: preload hero image**
```tsx
// Before: image loaded after HTML parse
<img src="/hero.jpg" alt="Hero" className="hero" />

// After: next/image with priority
import Image from "next/image";
<Image
  src="/hero.jpg"
  alt="Hero"
  width={1200}
  height={600}
  priority             // adds preload link in <head>
  sizes="100vw"
/>
// LCP: 4.2s → 1.8s ✓
```

**Fix 2 — CLS: add dimensions to all product images**
```tsx
// Before: images cause layout shift on load
<img src={product.imageUrl} alt={product.name} />

// After: reserved space
<Image
  src={product.imageUrl}
  alt={product.name}
  width={400}
  height={300}
  // next/image automatically generates width/height attributes
/>
// CLS from images: 0.25 → 0.0 ✓
```

**Fix 3 — CLS: reserve space for promo banner**
```tsx
// Before: banner appears after fetch, shifts content
const [banner, setBanner] = useState<PromoBanner | null>(null);
useEffect(() => { fetchBanner().then(setBanner); }, []);
return <>{banner && <Banner data={banner} />}<main>...</main></>;

// After: skeleton placeholder
return (
  <>
    <div style={{ minHeight: banner === undefined ? 56 : 0, transition: "none" }}>
      {banner && <Banner data={banner} />}
    </div>
    <main>...</main>
  </>
);
// CLS from banner: 0.03 → 0.0 ✓
// Total CLS: 0.28 → 0.01 ✓
```

**Fix 4 — INP: defer expensive filter computation**
```tsx
// Before: 500-item filter blocks main thread on every keystroke
const filtered = products.filter((p) => p.name.includes(query)); // sync, 180ms

// After: use useDeferredValue to defer the filter
const deferredQuery = useDeferredValue(query);
const filtered = useMemo(
  () => products.filter((p) => p.name.includes(deferredQuery)),
  [products, deferredQuery]
);
// INP: 380ms → 85ms ✓
```

**Final scores:** LCP: 1.8s (Good) · INP: 85ms (Good) · CLS: 0.01 (Good)

---

## 8. Interview Questions

**Q1: What are the three Core Web Vitals?**

LCP (Largest Contentful Paint) — measures loading performance: how long until the main content is visible. Target: ≤ 2.5s. INP (Interaction to Next Paint) — measures responsiveness: how long from user interaction to the next visual update. Target: ≤ 200ms. CLS (Cumulative Layout Shift) — measures visual stability: how much content unexpectedly moves around. Target: ≤ 0.1.

**Q2: What replaced FID and why?**

INP (Interaction to Next Paint) replaced FID (First Input Delay) in March 2024. FID only measured the delay before the browser started processing the *first* interaction — it didn't measure how long the interaction actually took to complete. INP measures the full duration of all interactions throughout the page lifetime, from input to next frame. A page could have a good FID (quick to start processing) but terrible INP (slow to actually render the result of interactions).

**Q3: How do you fix CLS caused by web fonts?**

Options in order of effectiveness: (1) `font-display: optional` — browser uses the fallback font if the web font doesn't load quickly; no swap visible. (2) `font-display: swap` with `size-adjust`, `ascent-override`, `descent-override` CSS descriptors to make the fallback font metrics match the web font — reduces the visual shift on swap. (3) Use `next/font` which handles all of this automatically plus self-hosts fonts for zero network latency. The root cause is that font loading changes text metrics (line height, word spacing), which reflowing existing text.

**Q4: What is layout thrashing?**

Alternating DOM reads and writes in a loop that forces the browser to recalculate layout (reflow) on every read. Example: reading `el.offsetWidth` after writing `el.style.width` forces a synchronous layout computation because the browser must flush pending style changes before providing a measurement. Fix: batch all reads first, then all writes. Or use `requestAnimationFrame` to separate reads (before frame) from writes (in rAF).

**Q5: How does next/image help Core Web Vitals?**

It addresses all three vitals: (1) **LCP** — the `priority` prop adds a `<link rel="preload">` header so the browser fetches the image early. It also generates `srcset` and serves WebP/AVIF which are smaller. (2) **CLS** — requires `width` and `height` props, which it uses to set the correct `aspect-ratio` CSS. The browser reserves the exact space before the image loads. (3) **INP** — lazy loads images outside the viewport (using `loading="lazy"` and IntersectionObserver) so they don't compete for bandwidth with interactive resources.

**Q6: How do you measure Web Vitals in production?**

Use the `web-vitals` library to instrument client-side measurement and send metrics to your analytics service. In Next.js, use the `reportWebVitals` export. For field data at scale (real users), use Google's Chrome User Experience Report (CrUX) via PageSpeed Insights or Search Console — these show the 75th percentile across real users over 28 days. Google uses CrUX data for ranking signals, not Lighthouse scores.

**Q7: What causes a slow LCP?**

Four main causes: (1) **Slow TTFB** — the server takes too long to respond; fix with CDN caching, ISR, edge rendering. (2) **Render-blocking resources** — CSS and synchronous JS block HTML parsing; fix with `defer`/`async` on scripts, inline critical CSS. (3) **Late image discovery** — the LCP image is referenced in a JS bundle or CSS background instead of in HTML; fix with `<link rel="preload">` or `priority` prop. (4) **Large image file sizes** — the image download takes too long; fix with WebP/AVIF format, correct sizing (`srcset`), and compression.

---

## 9. Exercises

**Exercise 1: Audit a page**

Open PageSpeed Insights (`pagespeed.web.dev`) for any public website you use. Identify the LCP element, look at CLS issues in the report, and find the INP score. Read the Opportunities section and pick one fix to implement.

**Exercise 2: Fix image CLS**

Take a page with images that have no dimensions. Add width/height attributes (or replace with `next/image`). Measure CLS before and after using Chrome DevTools Performance tab (look for Layout Shift entries in the Experience track).

**Exercise 3: Fix a slow filter with useTransition**

Build a component with a text input and a list of 5,000 items. Without optimization, typing in the input is sluggish (the filter runs synchronously). Add `useDeferredValue` to defer the filter computation and verify the input feels responsive.

**Exercise 4: Measure INP in DevTools**

Open Chrome DevTools → Performance panel → record while clicking buttons and typing on a heavy page. Find "Long Tasks" (red blocks) in the flame chart. Identify which JavaScript is causing the long task.

**Exercise 5: Preload the LCP image**

For a page where the LCP element is a hero image, add a `<link rel="preload">` for that image (or use `next/image` with `priority`). Verify with Lighthouse that LCP improves. Check the Network waterfall to confirm the image starts loading earlier.

---

## 10. Further Reading

- [Web.dev — Core Web Vitals](https://web.dev/explore/vitals)
- [Web.dev — Optimize LCP](https://web.dev/articles/optimize-lcp)
- [Web.dev — Optimize INP](https://web.dev/articles/optimize-inp)
- [Web.dev — Optimize CLS](https://web.dev/articles/optimize-cls)
- [web-vitals npm package](https://github.com/GoogleChrome/web-vitals)
- [PageSpeed Insights](https://pagespeed.web.dev/)
- [Next.js — Optimizing Images](https://nextjs.org/docs/app/building-your-application/optimizing/images)
- [Next.js — Optimizing Fonts](https://nextjs.org/docs/app/building-your-application/optimizing/fonts)
- [Chrome User Experience Report (CrUX)](https://developer.chrome.com/docs/crux)
- [Web.dev — Optimize Long Tasks](https://web.dev/articles/optimize-long-tasks)
