# React Performance

## 1. What & Why

React is fast by default for most applications. When it isn't, the cause is almost always one of a small set of patterns: unnecessary re-renders, large lists rendered without virtualization, too much work happening on every keystroke, or bundle sizes that slow initial load. Understanding React's rendering model is the prerequisite — you cannot optimize what you don't understand.

The principle: **measure first, optimize second.** Most performance work in React is wasted because developers optimize components that weren't slow. Profile before you memoize.

---

## 2. Core Concepts

- **Reconciliation** — React's process of comparing the previous and next virtual DOM trees to determine the minimal set of DOM operations needed.
- **Render** — when React calls your component function. Renders are cheap on their own; the problem is unnecessary renders that cascade.
- **Commit** — when React applies changes to the real DOM. Only changed nodes are updated.
- **Bailout** — when React skips re-rendering a component because it detected no change.
- **Key** — tells React which list item is which across renders. Wrong keys cause incorrect reconciliation.

---

## 3. How It Works

### Reconciliation and the Diffing Algorithm

When state changes, React creates a new virtual DOM tree and diffs it against the previous one. The diff algorithm uses two heuristics:

1. **Different types at the same position** — React destroys the old subtree and builds a new one from scratch. `<div>` → `<span>` at the same spot tears down everything inside.

2. **Lists use keys** — React matches elements by `key` across renders. Without keys (or with index as key on a reordering list), React cannot tell if an item was added, removed, or moved — it re-renders all items after the change.

React re-renders a component when:
- Its state changes
- Its parent re-renders (even if props didn't change — this is the most common source of avoidable renders)
- Its context value changes
- A `forceUpdate` is called (class components)

React bails out (skips re-render) when:
- `React.memo` wraps the component and props haven't changed (shallow equal)
- `PureComponent` / `shouldComponentUpdate` returns false
- State is set to the same value (identical by `Object.is`)

---

## 4. Code Examples

### Profiling Before Optimizing

```tsx
// React DevTools Profiler — use the browser extension
// Enable "Record why each component rendered" in DevTools settings

// Programmatic profiling with the Profiler API
import { Profiler, ProfilerOnRenderCallback } from "react";

const onRender: ProfilerOnRenderCallback = (
  id,           // component tree identifier
  phase,        // "mount" or "update"
  actualDuration,  // time spent rendering
  baseDuration,    // estimated time without memoization
  startTime,
  commitTime
) => {
  if (actualDuration > 16) {
    // More than one frame — investigate
    console.warn(`[Profiler] ${id} took ${actualDuration.toFixed(2)}ms (${phase})`);
  }
};

function App() {
  return (
    <Profiler id="ProductList" onRender={onRender}>
      <ProductList />
    </Profiler>
  );
}
```

### React.memo

```tsx
import { memo, useState } from "react";

// Without memo: re-renders every time parent renders, even if item didn't change
// With memo: only re-renders when item or onSelect changes (shallow comparison)

interface Product {
  id: number;
  name: string;
  price: number;
}

const ProductRow = memo(function ProductRow({
  product,
  onSelect,
  isSelected,
}: {
  product: Product;
  onSelect: (id: number) => void;
  isSelected: boolean;
}) {
  console.log(`ProductRow ${product.id} rendered`); // trace renders in dev

  return (
    <tr
      onClick={() => onSelect(product.id)}
      style={{ background: isSelected ? "#e3f2fd" : "transparent" }}
    >
      <td>{product.name}</td>
      <td>${product.price.toFixed(2)}</td>
    </tr>
  );
});

// Custom comparison — when shallow compare isn't enough
const MemoWithCustomCompare = memo(
  function ExpensiveChart({ data }: { data: number[][] }) {
    return <canvas>…</canvas>;
  },
  (prev, next) => {
    // Return true to SKIP re-render (props are equal)
    // Only re-render when length changes (ignore value changes for this use case)
    return prev.data.length === next.data.length;
  }
);
```

> ⚠️ **React.memo only does shallow comparison.** If you pass a new object or array literal as a prop, the comparison always fails and memo does nothing. Fix the reference upstream with `useMemo` or `useCallback`.

### useMemo for Reference Stability

```tsx
import { useMemo, useState, useCallback } from "react";

function ProductList({ category }: { category: string }) {
  const [products, setProducts] = useState<Product[]>([
    { id: 1, name: "Widget", price: 9.99 },
    { id: 2, name: "Gadget", price: 24.99 },
    { id: 3, name: "Doohickey", price: 4.99 },
  ]);
  const [search, setSearch] = useState("");
  const [selectedId, setSelectedId] = useState<number | null>(null);

  // 1. Memoize expensive derived value
  const filtered = useMemo(
    () =>
      products
        .filter((p) => p.name.toLowerCase().includes(search.toLowerCase()))
        .sort((a, b) => a.name.localeCompare(b.name)),
    [products, search]
  );

  // 2. Stable function reference for memo'd child
  const handleSelect = useCallback((id: number) => {
    setSelectedId(id);
  }, []); // stable — only uses functional setSelectedId

  // 3. Stable object reference (e.g., for a context value or config object)
  const tableConfig = useMemo(
    () => ({ striped: true, bordered: true, category }),
    [category]
  );

  return (
    <div>
      <input value={search} onChange={(e) => setSearch(e.target.value)} />
      <table>
        <tbody>
          {filtered.map((product) => (
            <ProductRow
              key={product.id}
              product={product}
              onSelect={handleSelect}
              isSelected={selectedId === product.id}
            />
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

### Code Splitting with React.lazy and Suspense

```tsx
import { lazy, Suspense, useState } from "react";

// The import() call is the split point — webpack/Vite creates a separate chunk
const AdminPanel = lazy(() => import("./AdminPanel"));
const Analytics = lazy(() =>
  import("./Analytics").then((mod) => ({ default: mod.AnalyticsDashboard }))
);

// Route-based splitting (with React Router v6)
import { Routes, Route } from "react-router-dom";

const HomePage = lazy(() => import("./pages/HomePage"));
const ProductPage = lazy(() => import("./pages/ProductPage"));
const CheckoutPage = lazy(() => import("./pages/CheckoutPage"));

function Router() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path="/" element={<HomePage />} />
        <Route path="/products/:id" element={<ProductPage />} />
        <Route path="/checkout" element={<CheckoutPage />} />
      </Routes>
    </Suspense>
  );
}

// Component-level splitting — load heavy components on demand
function Dashboard() {
  const [showAnalytics, setShowAnalytics] = useState(false);

  return (
    <div>
      <button onClick={() => setShowAnalytics(true)}>Show Analytics</button>
      {showAnalytics && (
        <Suspense fallback={<div>Loading analytics…</div>}>
          <Analytics />
        </Suspense>
      )}
    </div>
  );
}
```

### Virtualization with react-window

```tsx
import { FixedSizeList, VariableSizeList } from "react-window";

// Problem: rendering 10,000 rows creates 10,000 DOM nodes — kills performance
// Solution: only render the ~10 rows visible in the viewport at any moment

interface ListItem {
  id: number;
  name: string;
  description: string;
}

const ITEM_HEIGHT = 60;
const LIST_HEIGHT = 600;

function VirtualizedList({ items }: { items: ListItem[] }) {
  return (
    <FixedSizeList
      height={LIST_HEIGHT}    // viewport height (px)
      width="100%"
      itemCount={items.length}
      itemSize={ITEM_HEIGHT}  // each row height (px)
      itemData={items}        // passed to each Row as data prop
    >
      {Row}
    </FixedSizeList>
  );
}

// Row must be memoized — it's called for every visible item on scroll
const Row = memo(function Row({
  index,
  style,
  data,
}: {
  index: number;
  style: React.CSSProperties;
  data: ListItem[];
}) {
  const item = data[index];
  return (
    // style from react-window provides position and size — always apply it
    <div style={style} className="list-row">
      <strong>{item.name}</strong>
      <span>{item.description}</span>
    </div>
  );
});

// Variable height rows — more complex, requires getItemSize
function VariableList({ items }: { items: { id: number; text: string }[] }) {
  const getItemSize = (index: number) => {
    // Estimate based on text length
    return items[index].text.length > 100 ? 100 : 60;
  };

  return (
    <VariableSizeList
      height={LIST_HEIGHT}
      width="100%"
      itemCount={items.length}
      itemSize={getItemSize}
    >
      {({ index, style }) => (
        <div style={style}>{items[index].text}</div>
      )}
    </VariableSizeList>
  );
}
```

### Concurrent Features

```tsx
import { useTransition, useDeferredValue, useState } from "react";

// useTransition — mark a state update as non-urgent
// React will not interrupt urgent updates (typing) to process the transition
function SearchPage() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState<string[]>([]);
  const [isPending, startTransition] = useTransition();

  const handleSearch = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;
    setQuery(value); // urgent: update input immediately

    // Non-urgent: search results can wait
    // This prevents the input from feeling sluggish on slow devices
    startTransition(() => {
      setResults(expensiveSearch(value));
    });
  };

  return (
    <div>
      <input value={query} onChange={handleSearch} />
      {isPending && <span>Searching…</span>}
      <ul>
        {results.map((r) => <li key={r}>{r}</li>)}
      </ul>
    </div>
  );
}

// useDeferredValue — defer the expensive part of a render
// The input stays responsive; the list rerenders at lower priority
function DeferredList({ query }: { query: string }) {
  // deferredQuery lags behind query during updates
  // React renders with the old deferredQuery first, then updates it
  const deferredQuery = useDeferredValue(query);

  // This component only re-renders when deferredQuery changes
  // The parent (with query) re-renders more often but this tree is deferred
  const results = useMemo(
    () => expensiveFilter(deferredQuery),
    [deferredQuery]
  );

  return (
    <ul style={{ opacity: query !== deferredQuery ? 0.6 : 1 }}>
      {results.map((r) => <li key={r}>{r}</li>)}
    </ul>
  );
}

function expensiveSearch(query: string): string[] {
  // Simulate expensive computation
  return Array.from({ length: 1000 }, (_, i) => `Result ${i} for "${query}"`);
}

function expensiveFilter(query: string): string[] {
  return expensiveSearch(query).filter((_, i) => i < 50);
}
```

---

## 5. Common Mistakes & Pitfalls

**1. Inline objects/arrays/functions in JSX — kills memo**

```tsx
// WRONG — new object reference every render → ProductRow always re-renders
function Parent() {
  return (
    <ProductRow
      style={{ color: "red" }} // new object
      filters={["active"]}     // new array
      onSelect={(id) => console.log(id)} // new function
    />
  );
}

// CORRECT — stable references
const STYLE = { color: "red" };
const FILTERS = ["active"];

function Parent() {
  const handleSelect = useCallback((id: number) => console.log(id), []);
  return <ProductRow style={STYLE} filters={FILTERS} onSelect={handleSelect} />;
}
```

**2. Using index as key when the list reorders**

```tsx
// WRONG — if items reorder, React maps old state to wrong items
const [items, setItems] = useState(["Apple", "Banana", "Cherry"]);
return items.map((item, index) => (
  <input key={index} defaultValue={item} /> // controlled by index, not item
));

// CORRECT — use a stable, unique identity
const [items, setItems] = useState([
  { id: "a1", name: "Apple" },
  { id: "b2", name: "Banana" },
]);
return items.map((item) => (
  <input key={item.id} defaultValue={item.name} />
));
```

**3. Missing keys**

```tsx
// React warns and forces re-mount all children on every render
return items.map((item) => <Row item={item} />); // ← no key

// With key: React can efficiently reuse existing DOM nodes
return items.map((item) => <Row key={item.id} item={item} />);
```

**4. Premature memoization**

```tsx
// WRONG — memoizing a trivial computation wastes memory and adds comparison overhead
const doubled = useMemo(() => count * 2, [count]); // this is not expensive!

// CORRECT — just compute it
const doubled = count * 2;
```

**5. Wrapping memo'd children in non-stable functions**

```tsx
// WRONG — ProductRow is memo'd, but handleSelect is new every render
// memo's shallow comparison fails every time on the function prop
function List({ products }: { products: Product[] }) {
  return products.map((p) => (
    <ProductRow
      key={p.id}
      product={p}
      onSelect={(id) => deleteProduct(id)} // new function every render!
    />
  ));
}
```

---

## 6. When to Use / Not Use

| Optimization | Use when | Avoid when |
|-------------|----------|------------|
| `React.memo` | Child renders are measurably slow AND props are stable | Props change every render anyway (memo has no effect) |
| `useMemo` | Computation takes >1ms or you need reference stability | Simple values like `count * 2` |
| `useCallback` | Function passed to `memo`'d child or `useEffect` dep | Function not passed anywhere stable |
| Code splitting | Routes, heavy libraries (charts, editors, maps) | Small components — overhead not worth it |
| Virtualization | Lists with >100 items | Small lists (adds complexity) |
| `useTransition` | State updates that trigger expensive renders (search, filter) | Already fast renders |
| `useDeferredValue` | Deferring an expensive child render while input stays responsive | Simple components |

---

## 7. Real-World Scenario

A product catalog with 10,000 items, live search, sorting, and a detail panel that loads on selection.

```tsx
import { useState, useCallback, useMemo, useTransition, lazy, Suspense, memo } from "react";
import { FixedSizeList } from "react-window";

const ProductDetail = lazy(() => import("./ProductDetail"));

interface Product {
  id: number;
  name: string;
  category: string;
  price: number;
}

// Memoized row — only re-renders when its product or selection changes
const ProductRow = memo(function ProductRow({
  index,
  style,
  data,
}: {
  index: number;
  style: React.CSSProperties;
  data: {
    items: Product[];
    onSelect: (id: number) => void;
    selectedId: number | null;
  };
}) {
  const { items, onSelect, selectedId } = data;
  const product = items[index];

  return (
    <div
      style={{
        ...style,
        background: selectedId === product.id ? "#e3f2fd" : "transparent",
        padding: "8px 16px",
        cursor: "pointer",
        borderBottom: "1px solid #eee",
      }}
      onClick={() => onSelect(product.id)}
    >
      <strong>{product.name}</strong>
      <span style={{ float: "right" }}>${product.price.toFixed(2)}</span>
    </div>
  );
});

function ProductCatalog({ products }: { products: Product[] }) {
  const [search, setSearch] = useState("");
  const [category, setCategory] = useState("all");
  const [selectedId, setSelectedId] = useState<number | null>(null);
  const [isPending, startTransition] = useTransition();

  // Stable callback — ProductRow won't re-render when other state changes
  const handleSelect = useCallback((id: number) => setSelectedId(id), []);

  // Only recalculates when products, search, or category changes
  const filtered = useMemo(() => {
    const q = search.toLowerCase();
    return products.filter(
      (p) =>
        (category === "all" || p.category === category) &&
        p.name.toLowerCase().includes(q)
    );
  }, [products, search, category]);

  // Stable itemData object — FixedSizeList won't re-render rows unnecessarily
  const itemData = useMemo(
    () => ({ items: filtered, onSelect: handleSelect, selectedId }),
    [filtered, handleSelect, selectedId]
  );

  const handleSearch = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;
    // Input update is urgent; filtering is non-urgent
    startTransition(() => setSearch(value));
  };

  return (
    <div style={{ display: "flex", gap: 16 }}>
      <div style={{ flex: 1 }}>
        <input
          placeholder="Search products…"
          onChange={handleSearch}
        />
        <select value={category} onChange={(e) => setCategory(e.target.value)}>
          <option value="all">All</option>
          <option value="electronics">Electronics</option>
          <option value="clothing">Clothing</option>
        </select>
        {isPending && <span>Filtering…</span>}
        <FixedSizeList
          height={500}
          width="100%"
          itemCount={filtered.length}
          itemSize={56}
          itemData={itemData}
        >
          {ProductRow}
        </FixedSizeList>
        <p>{filtered.length} products</p>
      </div>

      {selectedId !== null && (
        <div style={{ width: 300 }}>
          <Suspense fallback={<div>Loading details…</div>}>
            <ProductDetail productId={selectedId} />
          </Suspense>
        </div>
      )}
    </div>
  );
}
```

---

## 8. Interview Questions

**Q1: When should you use React.memo?**

When you can measure that a component re-renders unnecessarily and those renders have a measurable cost. `React.memo` does a shallow comparison of props — it's only effective if the parent can provide stable prop references. If props change every render anyway (e.g., an inline function or object literal), `memo` does nothing. Profile first; memoize second.

**Q2: What is reconciliation?**

Reconciliation is React's process of comparing the previous and next virtual DOM trees to determine what actually changed in the real DOM. React uses two heuristics: (1) components of different types are treated as different subtrees — the old one is torn down and a new one built; (2) lists use keys to match elements across renders. Without keys, React cannot detect moves — it re-renders all elements from the changed index onward.

**Q3: Why is index as key bad?**

When a list reorders, React matches elements by key. If you use index as key, `key=0` still points to the first rendered element, not the same data item. If the data moves but index keys don't follow, React maps stale state (inputs, scroll positions, focus) to wrong items. Use stable, unique IDs (database IDs, UUIDs) as keys.

**Q4: What does useTransition do?**

It marks a state update as non-urgent (a "transition"). React processes urgent updates first (typing in an input) and defers the transition update. If a new urgent update arrives before the transition completes, React discards the transition work and starts fresh. This keeps the UI responsive during expensive renders. The `isPending` boolean indicates the transition is in progress.

**Q5: What is virtualization?**

Rendering only the visible subset of a large list rather than all items. Libraries like `react-window` position items absolutely and only mount ~10–20 DOM nodes at a time, regardless of list size. As the user scrolls, items outside the viewport are unmounted and items entering it are mounted. This keeps the DOM size constant.

**Q6: Difference between useMemo and React.memo?**

`useMemo` is a hook that memoizes a **computed value** inside a component (skips recomputation when deps haven't changed). `React.memo` is a HOC that memoizes a **component's render output** (skips re-rendering when props haven't shallowly changed). They solve different problems and are often used together: `React.memo` on the child, `useMemo`/`useCallback` on the parent to provide stable props.

**Q7: How does code splitting work in React?**

`React.lazy(() => import('./Component'))` tells the bundler (webpack/Vite) to split `Component` into a separate chunk. When React renders a `lazy` component for the first time, it dynamically loads the chunk from the network. `Suspense` shows a fallback while loading. The common pattern is route-based splitting: each route's page component is lazy-loaded so users only download the code for the route they visit.

**Q8: How do you profile a React app?**

1. React DevTools Profiler tab — records renders, shows which components rendered, why, and how long. Enable "Record why each component rendered" in DevTools settings.
2. Profiler API — wrap a component tree in `<Profiler id="..." onRender={callback}>` to capture render timing programmatically.
3. Chrome DevTools Performance tab — flame chart for full JavaScript execution, useful for finding long tasks.
4. `console.time` / `console.timeEnd` around expensive computations in development.
Never optimize in production based on feel — measure in dev with React DevTools first.

---

## 9. Exercises

**Exercise 1: Fix the render cascade**

Given a `Parent` component with a list of 100 `Child` items, clicking a button in `Parent` causes all 100 `Child` items to re-render even though their props didn't change. Fix it using `React.memo` and `useCallback`.

**Exercise 2: Virtualize a large list**

Given an array of 50,000 strings, render them with `react-window`'s `FixedSizeList`. Each row is 40px tall; the list container is 500px tall. Add a search input to filter items.

**Exercise 3: Code-split a heavy component**

Wrap a `<RichTextEditor>` component (which imports a large library) with `React.lazy` and `Suspense`. Only load it when the user clicks "Edit".

**Exercise 4: useTransition for a slow filter**

Build a search input that filters a list of 10,000 items. Without optimization it feels laggy. Add `useTransition` so the input updates immediately and the list updates as a non-urgent transition. Show a spinner during the transition.

---

## 10. Further Reading

- [React Docs — Rendering Performance](https://react.dev/learn/render-and-commit)
- [React Docs — useTransition](https://react.dev/reference/react/useTransition)
- [React Docs — Code Splitting](https://react.dev/reference/react/lazy)
- [react-window — Virtualize Large Lists](https://react-window.vercel.app/)
- [Kent C. Dodds — Fix the Slow Render Before You Fix the Re-render](https://kentcdodds.com/blog/fix-the-slow-render-before-you-fix-the-re-render)
- [Chrome DevTools — Performance](https://developer.chrome.com/docs/devtools/performance/)
- [Web.dev — Optimize Long Tasks](https://web.dev/articles/optimize-long-tasks)
