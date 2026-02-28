# Next.js: SSR, SSG, and ISR

## 1. What & Why

Next.js offers multiple rendering strategies, and choosing between them is one of the most impactful architectural decisions in a Next.js application. The choice determines:

- **How fast pages load** for users
- **How fresh the data is** when they arrive
- **How much your server/CDN pays** to serve the page
- **How well search engines can index** your content

The App Router simplifies this considerably: instead of choosing a strategy per-page with explicit functions, you configure it per-fetch via cache options. Next.js infers the rendering strategy automatically.

---

## 2. Core Concepts

| Strategy | When page is rendered | Data freshness | Cost |
|---------|----------------------|----------------|------|
| **SSG** (Static) | Build time | Stale until rebuild | Cheapest (CDN serves HTML) |
| **ISR** (Incremental Static Regeneration) | Build time + periodic revalidation | Up to N seconds stale | Low (CDN + occasional server) |
| **SSR** (Dynamic per-request) | Each request | Always fresh | Higher (server on every request) |
| **Client-side** | In browser | Fetched by browser | Server-light, TTFB fast, content delayed |

The App Router expresses these strategies through `fetch()` options — or automatically through whether you use dynamic APIs.

---

## 3. How It Works

### Static vs Dynamic Rendering Decision

Next.js determines at build time (and per-request in dev) whether a route is static or dynamic:

**Static rendering** when:
- All `fetch()` calls use `cache: 'force-cache'` (or no cache option — default is force-cache)
- No dynamic APIs are used: `cookies()`, `headers()`, `searchParams`, `connection()`

**Dynamic rendering** when:
- Any `fetch()` uses `cache: 'no-store'`
- Any dynamic API is accessed: `cookies()`, `headers()`, `searchParams`
- A route segment exports `export const dynamic = 'force-dynamic'`

```
Next.js build output:
○ /about           (static)        — pre-rendered HTML
○ /blog/[slug]     (static)        — pre-rendered for all slugs
λ /dashboard       (dynamic)       — rendered per-request
λ /api/users       (dynamic)       — route handler, no caching
```

---

## 4. Code Examples

### Static Generation (SSG)

```tsx
// app/blog/page.tsx — statically rendered at build time
// fetch() defaults to force-cache in Next.js (equivalent to SSG)

export default async function BlogPage() {
  // This fetch is cached indefinitely until revalidated
  const posts = await fetch("https://api.example.com/posts", {
    cache: "force-cache",
  }).then((r) => r.json());

  return (
    <ul>
      {posts.map((post: { id: string; title: string }) => (
        <li key={post.id}>
          <a href={`/blog/${post.id}`}>{post.title}</a>
        </li>
      ))}
    </ul>
  );
}

// Dynamic routes with SSG — pre-generate all slugs at build time
// app/blog/[slug]/page.tsx
export async function generateStaticParams() {
  const posts = await fetch("https://api.example.com/posts", {
    cache: "force-cache",
  }).then((r) => r.json());

  // Return all param combinations — Next.js pre-renders one page per entry
  return posts.map((post: { slug: string }) => ({ slug: post.slug }));
}

export default async function BlogPost({
  params,
}: {
  params: Promise<{ slug: string }>;
}) {
  const { slug } = await params;
  const post = await fetch(`https://api.example.com/posts/${slug}`, {
    cache: "force-cache",
  }).then((r) => r.json());

  return (
    <article>
      <h1>{post.title}</h1>
      <div dangerouslySetInnerHTML={{ __html: post.content }} />
    </article>
  );
}

// What happens for slugs NOT in generateStaticParams?
// By default: 404. Override with:
export const dynamicParams = true; // generate on first request, then cache (default)
// export const dynamicParams = false; // 404 for unknown slugs
```

> 💡 **generateStaticParams runs at build time.** For 10,000 blog posts, it would pre-render 10,000 pages. For large content sets, use `dynamicParams = true` and only pre-render the most popular pages in `generateStaticParams`.

### Server-Side Rendering (SSR)

```tsx
// app/dashboard/page.tsx — dynamic rendering (per-request)

import { cookies } from "next/headers"; // using dynamic API = forces dynamic rendering

export default async function DashboardPage() {
  const cookieStore = await cookies();
  const sessionToken = cookieStore.get("session")?.value;

  if (!sessionToken) {
    redirect("/login");
  }

  // Each request gets fresh data
  const stats = await fetch("https://api.example.com/stats", {
    cache: "no-store", // explicitly opt out of caching
    headers: { Authorization: `Bearer ${sessionToken}` },
  }).then((r) => r.json());

  return (
    <div>
      <h1>Dashboard</h1>
      <p>Active users: {stats.activeUsers}</p>
      <p>Revenue today: ${stats.revenueToday.toFixed(2)}</p>
    </div>
  );
}

// Alternative: force dynamic with config export (no need to use a dynamic API)
export const dynamic = "force-dynamic"; // always render per-request
```

### Incremental Static Regeneration (ISR)

```tsx
// app/products/page.tsx — ISR: static, revalidated every 60 seconds

export default async function ProductsPage() {
  // Stale-while-revalidate model:
  // - Serve cached HTML immediately
  // - If cache is older than `revalidate` seconds, revalidate in background
  // - Next request gets the freshly generated page
  const products = await fetch("https://api.example.com/products", {
    next: { revalidate: 60 }, // revalidate at most every 60 seconds
  }).then((r) => r.json());

  return (
    <div>
      <h1>Products</h1>
      <p>Last updated: {new Date().toISOString()}</p>
      <ul>
        {products.map((p: { id: string; name: string }) => (
          <li key={p.id}>{p.name}</li>
        ))}
      </ul>
    </div>
  );
}

// Route-level revalidation (applies to all fetches without explicit config)
export const revalidate = 60; // ISR for the whole page

// Tag-based revalidation — more precise than time-based
export default async function TaggedPage() {
  const data = await fetch("https://api.example.com/data", {
    next: { tags: ["products", "catalog"] }, // tag this cached response
  }).then((r) => r.json());

  return <div>{JSON.stringify(data)}</div>;
}
```

### On-Demand Revalidation

```tsx
// app/actions/revalidate.ts — Server Action to revalidate on demand
"use server";

import { revalidatePath, revalidateTag } from "next/cache";

// Revalidate a specific URL path
export async function revalidateProductsPage() {
  revalidatePath("/products");           // exact path
  revalidatePath("/products", "page");   // just the page.tsx
  revalidatePath("/products/[id]", "page"); // all dynamic product pages
}

// Revalidate by tag — more targeted, revalidates all fetches with that tag
export async function revalidateProductCache() {
  revalidateTag("products"); // invalidates all fetches tagged with "products"
}

// Route Handler for webhook-triggered revalidation
// app/api/revalidate/route.ts
import { NextRequest, NextResponse } from "next/server";
import { revalidateTag } from "next/cache";

export async function POST(request: NextRequest) {
  const { searchParams } = request.nextUrl;
  const secret = searchParams.get("secret");

  if (secret !== process.env.REVALIDATION_SECRET) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  const body = await request.json() as { tag?: string; path?: string };

  if (body.tag) revalidateTag(body.tag);
  if (body.path) revalidatePath(body.path);

  return NextResponse.json({ revalidated: true, at: new Date().toISOString() });
}

// Your CMS calls: POST /api/revalidate?secret=xxx  { "tag": "products" }
// This invalidates the products cache immediately without rebuilding
```

### Streaming with Suspense

```tsx
// app/dashboard/page.tsx — stream slow components independently

import { Suspense } from "react";

// Immediate — renders fast with static data
async function QuickStats() {
  const stats = await fetch("/api/quick-stats", { cache: "no-store" })
    .then((r) => r.json());
  return <div className="stats">{stats.activeUsers} active users</div>;
}

// Slow — takes 2-3 seconds to generate
async function RevenueChart() {
  const data = await fetch("/api/revenue-heavy", { cache: "no-store" })
    .then((r) => r.json());
  return <Chart data={data} />;
}

// Slowest — AI-generated insights
async function AIInsights() {
  const insights = await fetch("/api/ai-insights", { cache: "no-store" })
    .then((r) => r.json());
  return <ul>{insights.map((i: string) => <li key={i}>{i}</li>)}</ul>;
}

export default function DashboardPage() {
  // Each Suspense boundary streams independently:
  // QuickStats HTML arrives first, then RevenueChart, then AIInsights
  // User sees content progressively — no white page waiting for the slowest query
  return (
    <div>
      <Suspense fallback={<Skeleton />}>
        <QuickStats />
      </Suspense>

      <Suspense fallback={<ChartSkeleton />}>
        <RevenueChart />
      </Suspense>

      <Suspense fallback={<InsightsSkeleton />}>
        <AIInsights />
      </Suspense>
    </div>
  );
}
```

> 💡 **Streaming reduces Time to First Byte (TTFB) perceptually.** The browser receives and renders HTML in chunks — users see content incrementally rather than waiting for the slowest database query before seeing anything.

### Edge vs Node.js Runtime

```tsx
// Route can opt into the Edge runtime
// app/api/geo/route.ts
export const runtime = "edge"; // or "nodejs" (default)

import { NextRequest } from "next/server";

export function GET(request: NextRequest) {
  // Edge runtime has access to:
  const country = request.geo?.country ?? "Unknown"; // geolocation
  const city = request.geo?.city ?? "Unknown";

  // Edge runtime does NOT have:
  // - fs (no file system)
  // - Most Node.js built-in modules
  // - Prisma (uses native binaries)

  return Response.json({ country, city });
}

// Edge middleware — runs before every request at the CDN level
// middleware.ts (at project root)
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

export function middleware(request: NextRequest) {
  const country = request.geo?.country;
  const url = request.nextUrl.clone();

  // Redirect users from specific countries
  if (country === "BR") {
    url.pathname = `/pt${url.pathname}`;
    return NextResponse.rewrite(url);
  }

  // Auth check at the edge — no DB access (use JWT, not session lookup)
  const token = request.cookies.get("token")?.value;
  if (!token && url.pathname.startsWith("/dashboard")) {
    url.pathname = "/login";
    return NextResponse.redirect(url);
  }

  return NextResponse.next();
}

export const config = {
  matcher: ["/dashboard/:path*", "/api/protected/:path*"],
};
```

**Edge vs Node.js comparison:**

| | Edge | Node.js |
|--|------|---------|
| Cold start | ~0ms | ~100-500ms |
| Execution | CDN-distributed (Vercel Edge Network) | Single region (or multi with config) |
| Memory | 128MB | 1-3GB |
| Node APIs | No `fs`, no native modules | Full Node.js |
| Prisma | Not supported | Supported |
| JWT validation | Yes | Yes |
| Database queries | Only via HTTP (no TCP drivers) | Direct TCP (Postgres, MySQL) |

### Pages Router Reference (for comparison)

```tsx
// pages/products/index.tsx — old Pages Router

// SSG
export async function getStaticProps() {
  const products = await fetchProducts();
  return {
    props: { products },
    revalidate: 60, // ISR: revalidate every 60 seconds
  };
}

// SSR
export async function getServerSideProps(context) {
  const { req, res, query } = context;
  const session = await getSession(req);
  const data = await fetchData(session.userId);
  return { props: { data } };
}

// Dynamic SSG routes
export async function getStaticPaths() {
  const products = await fetchProducts();
  return {
    paths: products.map((p) => ({ params: { id: p.id } })),
    fallback: "blocking", // generate unknown paths on first request
  };
}

// App Router equivalents:
// getStaticProps      → fetch() with force-cache / next: { revalidate }
// getServerSideProps  → fetch() with no-store / using cookies()/headers()
// getStaticPaths      → generateStaticParams()
// fallback: 'blocking' → dynamicParams = true (default)
```

---

## 5. Common Mistakes & Pitfalls

**1. Fetching in a loop instead of in parallel**
```tsx
// WRONG — sequential fetches, each waits for the previous
async function Page({ ids }: { ids: string[] }) {
  const items = [];
  for (const id of ids) {
    items.push(await fetch(`/api/items/${id}`).then((r) => r.json()));
  }
  return <List items={items} />;
}

// CORRECT — parallel fetches
async function Page({ ids }: { ids: string[] }) {
  const items = await Promise.all(
    ids.map((id) => fetch(`/api/items/${id}`).then((r) => r.json()))
  );
  return <List items={items} />;
}
```

**2. No-store fetch in a layout makes the whole subtree dynamic**
```tsx
// app/layout.tsx — root layout
// If you fetch with no-store here, EVERY page becomes dynamic
export default async function RootLayout({ children }) {
  const config = await fetch("/api/config", { cache: "no-store" }); // ❌
  // ...
}

// CORRECT — use appropriate caching, or use revalidate
const config = await fetch("/api/config", { next: { revalidate: 3600 } });
```

**3. revalidatePath with wrong segment type**
```tsx
// Only revalidates the specific path (not /products/123 etc.)
revalidatePath("/products");

// Revalidate all pages in the products segment
revalidatePath("/products/[id]", "page"); // ← correct for dynamic pages
revalidatePath("/products", "layout");    // ← revalidates the layout too
```

**4. Forgetting generateStaticParams for dynamic SSG routes**
```tsx
// Without generateStaticParams, [slug] is rendered dynamically on every request
// Even if your data never changes!
export default async function Page({ params }) {
  const { slug } = await params;
  const data = await fetch(`/api/${slug}`, { cache: "force-cache" }).then(r => r.json());
  // ← this is still dynamic at runtime without generateStaticParams
}
```

---

## 6. When to Use / Not Use

| Strategy | Use when | Avoid when |
|---------|----------|------------|
| SSG | Marketing pages, blog posts, docs — content changes infrequently | User-specific data, real-time content |
| ISR | Product catalogs, news feeds — tolerate N seconds of staleness | Real-time dashboards, per-user content |
| SSR | Auth-gated pages, personalized content, real-time data | Public, infrequently-changing pages (use ISR instead) |
| Streaming | Pages with multiple data sources of varying speeds | Simple single-fetch pages |
| Edge runtime | Auth middleware, geolocation routing, JWT validation | Database queries, Prisma, file system access |

---

## 7. Real-World Scenario

An e-commerce site: static home + ISR product catalog + dynamic cart + streaming checkout.

```tsx
// app/page.tsx — fully static (SSG)
export const revalidate = false; // cache forever, manually revalidate from CMS webhook

export default async function HomePage() {
  const featuredProducts = await fetch("https://cms.example.com/featured", {
    cache: "force-cache",
  }).then((r) => r.json());

  return <FeaturedSection products={featuredProducts} />;
}

// app/products/page.tsx — ISR, fresh every 5 minutes
export const revalidate = 300;

export default async function ProductsPage() {
  const products = await fetch("https://api.example.com/products", {
    next: { tags: ["products"], revalidate: 300 },
  }).then((r) => r.json());

  return <ProductGrid products={products} />;
}

// app/cart/page.tsx — fully dynamic, always fresh
export const dynamic = "force-dynamic";

export default async function CartPage() {
  const session = await getSession(); // reads cookies → forces dynamic
  const cart = await db.cart.findUnique({ where: { userId: session.userId } });
  return <CartView cart={cart} />;
}

// app/checkout/page.tsx — streaming checkout experience
export default async function CheckoutPage() {
  const session = await getSession();
  return (
    <div>
      <Suspense fallback={<CartSummaryLoader />}>
        <CartSummary userId={session.userId} />
      </Suspense>
      <Suspense fallback={<ShippingLoader />}>
        <ShippingOptions userId={session.userId} />
      </Suspense>
      <PaymentForm /> {/* Client Component — interactive */}
    </div>
  );
}
```

When the CMS publishes a new featured product:
```bash
# CMS webhook calls:
POST /api/revalidate?secret=xxx
{ "tag": "products" }

# Next.js invalidates all fetches tagged "products"
# Next visitor to /products gets fresh HTML
```

---

## 8. Interview Questions

**Q1: What is ISR and how does it work?**

Incremental Static Regeneration renders a page statically at build time but revalidates it periodically. When a request comes in and the cache is stale (older than `revalidate` seconds), Next.js serves the stale page immediately (fast) and regenerates it in the background. The next request gets the fresh page. This is the stale-while-revalidate pattern. Configured with `next: { revalidate: N }` in fetch options.

**Q2: When would you use SSR vs SSG?**

SSG for content that doesn't change per-user and changes infrequently: marketing pages, blog posts, documentation. SSR (dynamic rendering) for content that is user-specific (auth-gated dashboards) or must be real-time (live inventory, personalized recommendations). ISR covers the middle ground: content that changes but doesn't need to be per-user (product catalogs, news).

**Q3: How does streaming work in Next.js?**

Next.js uses React's Suspense to stream HTML in chunks. The server sends the initial HTML immediately (without waiting for slow data), then streams additional chunks as Suspense boundaries resolve. Wrap slow Server Components in `<Suspense fallback={<Skeleton />}>`. The user sees the page progressively rather than waiting for the slowest query.

**Q4: What is the difference between revalidatePath and revalidateTag?**

`revalidatePath('/products')` invalidates the cached page at that exact URL. `revalidateTag('products')` invalidates all `fetch()` responses that were tagged with that string (via `next: { tags: ['products'] }`). Tags are more flexible — a single CMS publish can invalidate all pages that show product data regardless of their URL.

**Q5: What is the edge runtime?**

The edge runtime runs JavaScript at CDN nodes globally (e.g., Vercel's Edge Network) rather than a central Node.js server. It starts in ~0ms (no cold start), supports a limited subset of Node.js APIs (no `fs`, no native addons, no Prisma), and has access to request geolocation. It's best for middleware, auth token validation, and geolocation routing — not for database queries.

**Q6: How does Next.js decide to render statically vs dynamically?**

At build time, Next.js analyzes each route. If any component in the route tree calls a dynamic API (`cookies()`, `headers()`, `connection()`) or fetches with `cache: 'no-store'`, the entire route is marked as dynamic (per-request). Otherwise, it's statically rendered. You can override with `export const dynamic = 'force-static'` or `'force-dynamic'`.

**Q7: How do you handle dynamic routes with SSG?**

Export `generateStaticParams()` from the page file. It returns an array of param objects (e.g., `[{ slug: 'hello' }, { slug: 'world' }]`). Next.js pre-renders one page per entry at build time. For slugs not in the list, `dynamicParams = true` (default) renders them on first request and caches; `dynamicParams = false` returns 404.

---

## 9. Exercises

**Exercise 1: Blog with ISR**

Build a blog with SSG for individual posts (`/blog/[slug]`) and ISR with 5-minute revalidation for the post list (`/blog`). Implement `generateStaticParams` for the top 10 posts. Add a revalidation webhook at `/api/revalidate`.

**Exercise 2: Streaming dashboard**

Build a dashboard with three slow data sections (stats, chart, activity feed). Each should have its own `Suspense` boundary with a skeleton fallback. The sections should stream in independently.

**Exercise 3: Edge middleware auth guard**

Write `middleware.ts` that reads a JWT from a cookie, validates it (without a DB call), and redirects unauthenticated users from `/dashboard/*` to `/login`. Bonus: use `request.geo` to redirect non-US users to `/global`.

**Exercise 4: On-demand revalidation**

Build a simple CMS admin page (server action) that publishes a new product. On publish, call `revalidateTag('products')` to invalidate the product catalog cache. Verify that the catalog page shows fresh data without a rebuild.

---

## 10. Further Reading

- [Next.js Docs — Data Fetching](https://nextjs.org/docs/app/building-your-application/data-fetching/fetching)
- [Next.js Docs — Caching](https://nextjs.org/docs/app/building-your-application/caching)
- [Next.js Docs — Rendering Strategies](https://nextjs.org/docs/app/building-your-application/rendering)
- [Next.js Docs — Streaming](https://nextjs.org/docs/app/building-your-application/routing/loading-ui-and-streaming)
- [Next.js Docs — Edge Runtime](https://nextjs.org/docs/app/building-your-application/rendering/edge-and-nodejs-runtimes)
- [Vercel — ISR Deep Dive](https://vercel.com/docs/incremental-static-regeneration)
- [Web.dev — Rendering on the Web](https://web.dev/articles/rendering-on-the-web)
