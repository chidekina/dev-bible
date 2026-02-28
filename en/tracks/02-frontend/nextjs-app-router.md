# Next.js App Router

## 1. What & Why

The App Router (introduced in Next.js 13, stable in 13.4) is a fundamental rethink of how Next.js applications are structured. It moves from a pages-first model — where every file in `pages/` is a route — to a layouts-first model built on React Server Components.

The core shift: **components are server components by default.** They run on the server, produce HTML, and ship zero JavaScript to the client. Only components that need interactivity or browser APIs opt in to client-side rendering with a `"use client"` directive.

This matters because:
- **Bundle size shrinks** — server components never appear in the client bundle
- **Data fetching is co-located** — components fetch their own data (no `getServerSideProps`)
- **Layouts persist** — nested layouts survive route changes without unmounting
- **Streaming is built in** — loading states are first-class via `loading.tsx` and `<Suspense>`

---

## 2. Core Concepts

| Concept | Description |
|---------|-------------|
| Server Component | Renders on the server; async; can access DB; no hooks/browser APIs |
| Client Component | Renders on client (and server for initial HTML); has hooks and browser APIs |
| Layout | Wraps all children in a segment; persists across navigations |
| Template | Like a layout but re-mounts on each navigation |
| Loading | Automatic Suspense boundary for a route segment |
| Error | Error boundary for a route segment |
| Route Handler | API endpoint in the App Router (`route.ts`) |
| Server Action | Async server function callable from client components |
| Parallel Routes | Simultaneous rendering of multiple segments in a layout |
| Intercepting Routes | Intercept a navigation and show it in a different context (modals) |

---

## 3. How It Works

### File-System Routing

```
app/
├── layout.tsx           ← root layout (wraps everything)
├── page.tsx             ← / route
├── loading.tsx          ← Suspense boundary for /
├── error.tsx            ← Error boundary for /
├── not-found.tsx        ← 404 for /
├── dashboard/
│   ├── layout.tsx       ← layout wrapping all dashboard routes
│   ├── page.tsx         ← /dashboard
│   ├── loading.tsx      ← Suspense boundary for /dashboard/*
│   ├── settings/
│   │   └── page.tsx     ← /dashboard/settings
│   └── @analytics/      ← parallel route slot named "analytics"
│       └── page.tsx
├── products/
│   ├── page.tsx         ← /products
│   └── [id]/
│       └── page.tsx     ← /products/:id (dynamic segment)
└── api/
    └── users/
        └── route.ts     ← /api/users (Route Handler)
```

---

## 4. Code Examples

### Server Components

```tsx
// app/products/page.tsx
// No "use client" directive = Server Component
// This runs on the server — can be async, can query DB directly

import { db } from "@/lib/db";
import { ProductCard } from "./product-card";

// Props come from the URL — searchParams for query strings
interface Props {
  searchParams: Promise<{ category?: string; page?: string }>;
}

export default async function ProductsPage({ searchParams }: Props) {
  // In Next.js 15+, searchParams is a Promise
  const { category, page = "1" } = await searchParams;

  // Direct DB access — no API round-trip needed
  const products = await db.product.findMany({
    where: category ? { category } : undefined,
    skip: (Number(page) - 1) * 20,
    take: 20,
    orderBy: { createdAt: "desc" },
  });

  return (
    <section>
      <h1>Products {category ? `in ${category}` : ""}</h1>
      <div className="grid">
        {products.map((product) => (
          // ProductCard is a Server Component too — renders HTML, no JS
          <ProductCard key={product.id} product={product} />
        ))}
      </div>
    </section>
  );
}

// Metadata for SEO — also runs on the server
export async function generateMetadata({ searchParams }: Props) {
  const { category } = await searchParams;
  return {
    title: category ? `${category} Products` : "All Products",
    description: "Browse our product catalog",
  };
}
```

### Client Components

```tsx
// app/products/add-to-cart-button.tsx
"use client"; // opt in to client-side rendering

import { useState } from "react";
import { addToCart } from "@/lib/cart";

interface Props {
  productId: string;
  productName: string;
}

// This component uses useState and click events — needs to be a Client Component
export function AddToCartButton({ productId, productName }: Props) {
  const [loading, setLoading] = useState(false);
  const [added, setAdded] = useState(false);

  const handleClick = async () => {
    setLoading(true);
    try {
      await addToCart(productId);
      setAdded(true);
    } finally {
      setLoading(false);
    }
  };

  return (
    <button onClick={handleClick} disabled={loading || added}>
      {added ? "Added!" : loading ? "Adding…" : `Add ${productName} to Cart`}
    </button>
  );
}
```

### Mixing Server and Client Components

```tsx
// app/products/[id]/page.tsx — Server Component
import { db } from "@/lib/db";
import { AddToCartButton } from "../add-to-cart-button"; // Client Component
import { ReviewList } from "./review-list";              // Server Component

export default async function ProductPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;
  const product = await db.product.findUniqueOrThrow({ where: { id } });

  return (
    <div>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <p>${product.price.toFixed(2)}</p>

      {/* Client Component receives serializable props (no functions, no class instances) */}
      <AddToCartButton productId={product.id} productName={product.name} />

      {/* Server Component — fetches its own reviews data */}
      <ReviewList productId={product.id} />
    </div>
  );
}
```

> ⚠️ **Server Components cannot be imported by Client Components.** You can pass a Server Component as `children` to a Client Component (React renders the server part first), but you cannot `import` it inside a `"use client"` file.

```tsx
// WRONG — importing a server component from a client component
"use client";
import { ServerOnlyComponent } from "./server-only"; // ❌ Error

// CORRECT — pass as children prop
// In the server component:
<ClientWrapper>
  <ServerOnlyComponent />
</ClientWrapper>

// In the client wrapper:
"use client";
function ClientWrapper({ children }: { children: React.ReactNode }) {
  const [open, setOpen] = useState(false);
  return <div onClick={() => setOpen(true)}>{children}</div>;
}
```

### Layouts

```tsx
// app/layout.tsx — root layout (required)
import type { Metadata } from "next";
import { Inter } from "next/font/google";
import { Navbar } from "@/components/navbar";
import "./globals.css";

const inter = Inter({ subsets: ["latin"] });

export const metadata: Metadata = {
  title: { default: "My App", template: "%s | My App" },
  description: "A Next.js application",
};

// Root layout wraps every page — never unmounts
export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body className={inter.className}>
        <Navbar />
        <main>{children}</main>
      </body>
    </html>
  );
}

// app/dashboard/layout.tsx — nested layout
import { Sidebar } from "@/components/sidebar";
import { requireAuth } from "@/lib/auth";

export default async function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  // Fetch session once here — all dashboard pages share this
  const session = await requireAuth(); // redirects if not authenticated

  return (
    <div className="dashboard">
      <Sidebar user={session.user} />
      <div className="dashboard__content">{children}</div>
    </div>
  );
}
```

### Loading and Error Boundaries

```tsx
// app/dashboard/loading.tsx — automatic Suspense boundary
// Next.js wraps the page in <Suspense fallback={<Loading />}>
export default function DashboardLoading() {
  return (
    <div className="skeleton-grid">
      {Array.from({ length: 6 }, (_, i) => (
        <div key={i} className="skeleton-card" aria-busy="true" />
      ))}
    </div>
  );
}

// app/dashboard/error.tsx — must be a Client Component (uses onClick/retry)
"use client";
import { useEffect } from "react";

interface ErrorProps {
  error: Error & { digest?: string };
  reset: () => void; // retry — re-renders the segment
}

export default function DashboardError({ error, reset }: ErrorProps) {
  useEffect(() => {
    // Log to error tracking service
    console.error("[DashboardError]", error);
  }, [error]);

  return (
    <div role="alert">
      <h2>Something went wrong</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  );
}
```

### Route Handlers

```tsx
// app/api/users/route.ts
import { NextRequest, NextResponse } from "next/server";
import { db } from "@/lib/db";
import { z } from "zod";

// GET /api/users?page=1&limit=20
export async function GET(request: NextRequest) {
  const { searchParams } = request.nextUrl;
  const page = Number(searchParams.get("page") ?? 1);
  const limit = Math.min(Number(searchParams.get("limit") ?? 20), 100);

  const [users, total] = await Promise.all([
    db.user.findMany({ skip: (page - 1) * limit, take: limit }),
    db.user.count(),
  ]);

  return NextResponse.json({ users, total, page, limit });
}

const CreateUserSchema = z.object({
  name: z.string().min(1).max(100),
  email: z.string().email(),
});

// POST /api/users
export async function POST(request: NextRequest) {
  const body = await request.json();
  const parsed = CreateUserSchema.safeParse(body);

  if (!parsed.success) {
    return NextResponse.json(
      { error: "Validation failed", details: parsed.error.flatten() },
      { status: 400 }
    );
  }

  const user = await db.user.create({ data: parsed.data });
  return NextResponse.json(user, { status: 201 });
}

// app/api/users/[id]/route.ts — dynamic route handler
export async function GET(
  _request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;
  const user = await db.user.findUnique({ where: { id } });
  if (!user) return NextResponse.json({ error: "Not found" }, { status: 404 });
  return NextResponse.json(user);
}
```

### Server Actions

```tsx
// app/actions/product.ts
"use server";

import { revalidatePath, revalidateTag } from "next/cache";
import { redirect } from "next/navigation";
import { db } from "@/lib/db";
import { z } from "zod";

const ProductSchema = z.object({
  name: z.string().min(1),
  price: z.coerce.number().positive(),
  category: z.string(),
});

// Server Action — called directly from Client Components or form actions
export async function createProduct(formData: FormData) {
  const parsed = ProductSchema.safeParse({
    name: formData.get("name"),
    price: formData.get("price"),
    category: formData.get("category"),
  });

  if (!parsed.success) {
    return { error: parsed.error.flatten() };
  }

  const product = await db.product.create({ data: parsed.data });

  // Invalidate cached pages that show products
  revalidatePath("/products");
  revalidateTag("products"); // if you tagged fetch calls

  redirect(`/products/${product.id}`);
}

// Client Component using the Server Action
// app/products/new/page.tsx
"use client";
import { useFormState } from "react-dom";
import { createProduct } from "@/app/actions/product";

const initialState = { error: null };

export default function NewProductPage() {
  const [state, formAction] = useFormState(createProduct, initialState);

  return (
    <form action={formAction}>
      <input name="name" placeholder="Product name" required />
      <input name="price" type="number" step="0.01" placeholder="Price" required />
      <select name="category">
        <option value="electronics">Electronics</option>
        <option value="clothing">Clothing</option>
      </select>
      {state?.error && (
        <ul>
          {Object.entries(state.error.fieldErrors).map(([field, errors]) =>
            errors?.map((e) => <li key={`${field}-${e}`}>{field}: {e}</li>)
          )}
        </ul>
      )}
      <button type="submit">Create Product</button>
    </form>
  );
}
```

### Parallel Routes

```tsx
// app/dashboard/layout.tsx — renders @analytics and @team simultaneously
export default function DashboardLayout({
  children,
  analytics, // slot: app/dashboard/@analytics/page.tsx
  team,      // slot: app/dashboard/@team/page.tsx
}: {
  children: React.ReactNode;
  analytics: React.ReactNode;
  team: React.ReactNode;
}) {
  return (
    <div>
      <div>{children}</div>
      <aside>
        <section>{analytics}</section>
        <section>{team}</section>
      </aside>
    </div>
  );
}
```

### Intercepting Routes

```tsx
// Show a photo in a modal when clicked from the feed,
// but show the full page when navigated to directly

// app/photos/[id]/page.tsx — full page view
export default async function PhotoPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const photo = await fetchPhoto(id);
  return <FullPhotoView photo={photo} />;
}

// app/feed/(.)photos/[id]/page.tsx — intercepted modal view
// (.) = intercept from the same level; (..) = one level up; (...) = from root
"use client";
import { useRouter } from "next/navigation";
import { Modal } from "@/components/modal";

export default async function PhotoModal({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const photo = await fetchPhoto(id);
  const router = useRouter();

  return (
    <Modal onClose={() => router.back()}>
      <img src={photo.url} alt={photo.alt} />
    </Modal>
  );
}
```

### Metadata API

```tsx
// Static metadata
export const metadata = {
  title: "About Us",
  description: "Learn more about our company",
  openGraph: {
    title: "About Us",
    images: ["/og-about.png"],
  },
};

// Dynamic metadata — can fetch data
export async function generateMetadata({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;
  const product = await db.product.findUnique({ where: { id } });

  if (!product) return { title: "Product not found" };

  return {
    title: product.name,
    description: product.description,
    openGraph: {
      title: product.name,
      images: [product.imageUrl],
    },
  };
}
```

---

## 5. Common Mistakes & Pitfalls

**1. Putting too much in Client Components**
```tsx
// WRONG — entire page is a client component just to handle one button
"use client";
async function Page() { // async in client component doesn't work for data fetching
  const data = await fetch(...); // this runs on client, not server
  return <div><ComplexUI data={data} /><button>Click</button></div>;
}

// CORRECT — server component fetches data, client component handles interaction
async function Page() {
  const data = await fetch(...); // server
  return <div><ComplexUI data={data} /><InteractiveButton /></div>;
}
"use client";
function InteractiveButton() { return <button onClick={...}>Click</button>; }
```

**2. Importing a server-only library in a Client Component**
```tsx
// This would bundle fs, prisma, etc. to the client — breaks and leaks secrets
"use client";
import { db } from "@/lib/db"; // ❌ — db uses Prisma which can't run in browser
```

**3. Not awaiting params/searchParams in Next.js 15+**
```tsx
// WRONG in Next.js 15 (params is now a Promise)
export default function Page({ params }: { params: { id: string } }) {
  const { id } = params; // ❌ TypeScript error and runtime issue
}

// CORRECT
export default async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
}
```

**4. Using cookies/headers in a Server Component breaks static rendering**
```tsx
import { cookies } from "next/headers";

// This makes the page dynamically rendered (per-request) — intentional but be aware
export default async function Page() {
  const cookieStore = await cookies();
  const token = cookieStore.get("token");
  // ...
}
```

---

## 6. When to Use / Not Use

| Feature | Use when | Avoid when |
|---------|----------|------------|
| Server Component | Fetching data, rendering static content, DB access | Need hooks, event handlers, browser APIs |
| Client Component | Interactivity, useState/useEffect, onClick, forms | Just rendering — unnecessary client bundle weight |
| Server Actions | Form submissions, mutations with revalidation | Complex client-side optimistic UI (use route handler + client fetch) |
| Parallel Routes | Simultaneous independent sections in a layout | Sequential data that belongs in one component |
| Intercepting Routes | Modals that intercept navigations (photo galleries, quick edit) | Standard navigations |
| Route Handlers | REST API, webhooks, third-party callbacks | Mutations from client components (use Server Actions instead) |

---

## 7. Real-World Scenario

A product detail page with server-rendered product info, a client-side cart button, and streaming reviews.

```tsx
// app/products/[id]/page.tsx
import { Suspense } from "react";
import { db } from "@/lib/db";
import { notFound } from "next/navigation";
import { AddToCartButton } from "./add-to-cart-button";
import { ReviewList } from "./review-list"; // slow — stream it

export default async function ProductPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;
  const product = await db.product.findUnique({ where: { id } });
  if (!product) notFound();

  return (
    <div>
      {/* Rendered on server — zero client JS for this section */}
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <p className="price">${product.price.toFixed(2)}</p>

      {/* Client component — interactive */}
      <AddToCartButton productId={product.id} productName={product.name} />

      {/* Suspense — streams reviews independently */}
      {/* Page HTML is sent immediately; reviews stream in when ready */}
      <Suspense fallback={<ReviewsSkeleton />}>
        <ReviewList productId={product.id} />
      </Suspense>
    </div>
  );
}

// review-list.tsx — Server Component, slow fetch
async function ReviewList({ productId }: { productId: string }) {
  // This fetch delays; wrapped in Suspense so it doesn't block the rest
  const reviews = await db.review.findMany({
    where: { productId },
    take: 10,
    orderBy: { createdAt: "desc" },
  });

  return (
    <section>
      <h2>Reviews</h2>
      {reviews.map((r) => <ReviewCard key={r.id} review={r} />)}
    </section>
  );
}
```

---

## 8. Interview Questions

**Q1: Server vs Client Component — when to use each?**

Default to Server Components. Add `"use client"` only when the component needs: React hooks (`useState`, `useEffect`, etc.), browser APIs (`window`, `localStorage`, etc.), event handlers (`onClick`, `onChange`), or third-party libraries that use browser APIs. The rule of thumb: if it can be static HTML with no interactivity, it should be a Server Component.

**Q2: What is a Server Action?**

An async function marked with `"use server"` that runs on the server but can be called directly from a Client Component or used as an HTML form's `action`. Server Actions are the preferred way to handle mutations in the App Router — they can use `revalidatePath`/`revalidateTag` to invalidate cached pages and redirect after completion.

**Q3: How do layouts work in App Router?**

Layouts are defined in `layout.tsx` files. They wrap all routes in their segment and all nested segments. Critically, layouts **persist across navigations** — when navigating between child routes, the layout doesn't unmount and re-mount. This allows persistent UI like sidebars and headers. Layouts can be async Server Components that fetch shared data.

**Q4: Difference between loading.tsx and Suspense?**

`loading.tsx` is a convenience file that Next.js automatically wraps around `page.tsx` in a `<Suspense>` boundary. It's equivalent to writing `<Suspense fallback={<Loading />}><Page /></Suspense>`. Manual `<Suspense>` gives finer control — you can wrap individual slow components within a page (like the reviews example above) and stream them independently while the rest of the page renders immediately.

**Q5: How do you fetch data in Server Components?**

Use `async/await` directly in the component — Server Components can be async functions. You can call your database, an ORM, or `fetch()`. Next.js extends the native `fetch` with caching options: `cache: 'force-cache'` (static, cached), `cache: 'no-store'` (dynamic, per-request), or `next: { revalidate: 60 }` (ISR, revalidate every 60 seconds).

**Q6: What are parallel routes for?**

Rendering multiple independent route segments simultaneously in the same layout using the `@slot` folder convention. Common use case: a dashboard with separate sections (analytics, activity feed, team members) that each fetch their own data independently and can have their own loading/error states.

**Q7: How does caching work in the App Router?**

Four caching layers: (1) Request memoization — duplicate `fetch()` calls in one render pass are deduplicated. (2) Data Cache — `fetch()` results are cached on the server with `force-cache`; invalidated with `revalidatePath`/`revalidateTag`. (3) Full Route Cache — statically rendered routes are cached as HTML at build time. (4) Router Cache — client-side cache of prefetched route segments.

**Q8: What replaced getServerSideProps?**

Server Components with `cache: 'no-store'` or using dynamic APIs (`cookies()`, `headers()`, `searchParams`). A Server Component that reads `cookies()` or `headers()` is automatically rendered per-request. The App Router infers whether to render statically or dynamically based on which APIs the component uses — no explicit flag needed.

---

## 9. Exercises

**Exercise 1: Build an async layout with auth guard**

Create `app/dashboard/layout.tsx` that calls `getSession()` (mock it). If no session, `redirect('/login')`. If session exists, render `children` with a sidebar showing the user's name.

**Exercise 2: Server Action for a contact form**

Build a contact form page with a Server Action that validates input with Zod, saves to a database, and uses `revalidatePath` to clear the form page cache. Show validation errors in the form.

**Exercise 3: Parallel routes dashboard**

Build a `/dashboard` route with two parallel slots: `@stats` (shows user stats) and `@activity` (shows recent activity). Each slot has its own `loading.tsx`.

**Exercise 4: Intercepting route photo gallery**

Build a photo feed at `/feed`. When clicking a photo, intercept the navigation to show the photo in a modal (using `(.)photos/[id]/page.tsx`). The full page at `/photos/[id]` still works when navigated to directly.

---

## 10. Further Reading

- [Next.js Docs — App Router](https://nextjs.org/docs/app)
- [Next.js Docs — Server Components](https://nextjs.org/docs/app/getting-started/server-and-client-components)
- [Next.js Docs — Server Actions](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations)
- [Next.js Docs — Routing](https://nextjs.org/docs/app/building-your-application/routing)
- [React Docs — Server Components](https://react.dev/reference/rsc/server-components)
- [Vercel — Understanding Next.js Caching](https://nextjs.org/docs/app/building-your-application/caching)
