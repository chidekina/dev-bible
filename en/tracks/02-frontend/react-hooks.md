# React Hooks

## 1. What & Why

React Hooks, introduced in React 16.8, allow function components to use state, side effects, context, and other React features that were previously only available in class components. They solve several long-standing problems: logic reuse between components (without HOC wrapper hell), separation of concerns inside a component (vs lifecycle methods that forced unrelated logic together), and the confusion of `this` binding in class components.

Hooks are not a replacement for understanding React — they are built on the same mental model. Understanding them deeply separates developers who use React from developers who understand React.

---

## 2. Core Concepts

- **State** is a snapshot. State updates schedule a re-render with the new value — they do not mutate the current render's variables.
- **Effects** synchronize a component with an external system. Every effect runs after a committed render.
- **Refs** are escape hatches — mutable boxes that don't trigger re-renders.
- **Rules of Hooks** are not arbitrary: they exist because React tracks hook calls by call order in a linked list.
- **Custom hooks** are ordinary functions whose name starts with `use` — they can call other hooks and extract reusable stateful logic.

---

## 3. How It Works

React maintains a "fiber" (internal node) for every component instance. Each fiber has a linked list of "hook state" nodes. When a component renders, React walks through this list in order, matching each `useState`, `useEffect`, etc. call to its corresponding node. This is why hooks must always be called in the same order — any conditional call would desync the list.

```
Fiber node
  └─ memoizedState (linked list)
        ├─ hook[0]: useState(0)       → { memoizedState: 0, queue: ... }
        ├─ hook[1]: useState("hello") → { memoizedState: "hello", ... }
        └─ hook[2]: useEffect(...)    → { tag: HookPassive, ... }
```

If you call `useState` inside an `if`, the list shifts and React reads the wrong state for every subsequent hook — undefined behavior.

---

## 4. Code Examples

### useState

```tsx
import { useState } from "react";

// Basic usage
function Counter() {
  const [count, setCount] = useState(0);

  // State updates are async — count still holds the OLD value in this render
  const handleClick = () => {
    setCount(count + 1); // uses snapshot
    console.log(count);  // still logs old value — not the updated one
  };

  // Functional update — always gets the latest queued state
  // Use this when the next state depends on the previous state
  const safeIncrement = () => {
    setCount((prev) => prev + 1);
    setCount((prev) => prev + 1); // stacks correctly: +2 total
  };

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={handleClick}>Increment (snapshot)</button>
      <button onClick={safeIncrement}>Increment (functional)</button>
    </div>
  );
}

// Never mutate state directly — React won't re-render
function MutationExample() {
  const [items, setItems] = useState([1, 2, 3]);

  // WRONG: mutates original array — same reference, no re-render triggered
  const badAdd = () => {
    items.push(4); // mutation
    setItems(items); // same reference, React bails out
  };

  // CORRECT: new array reference — React detects change and re-renders
  const goodAdd = (n: number) => {
    setItems((prev) => [...prev, n]);
  };

  return <button onClick={() => goodAdd(4)}>Add 4</button>;
}

// Lazy initializer — only runs once on mount, useful for expensive initial state
function ExpensiveInit() {
  const [data, setData] = useState<unknown>(() => {
    // This function only runs on mount, not on every render
    try {
      return JSON.parse(localStorage.getItem("data") ?? "null");
    } catch {
      return null;
    }
  });

  return <div>{JSON.stringify(data)}</div>;
}
```

### useEffect

```tsx
import { useEffect, useState } from "react";

function DataFetcher({ userId }: { userId: string }) {
  const [user, setUser] = useState<{ name: string } | null>(null);
  const [loading, setLoading] = useState(true);

  // Empty array = run once on mount, cleanup on unmount
  useEffect(() => {
    console.log("mounted");
    return () => console.log("unmounted"); // cleanup
  }, []);

  // No array = runs after EVERY render (rarely what you want)
  useEffect(() => {
    document.title = `Viewing user ${userId}`;
  }); // ← no dependency array

  // With deps = runs when userId changes
  useEffect(() => {
    let cancelled = false; // cleanup flag for async operations

    setLoading(true);

    fetch(`/api/users/${userId}`)
      .then((res) => res.json())
      .then((data) => {
        if (!cancelled) {
          // Guard: don't update state if component unmounted or userId changed
          setUser(data);
          setLoading(false);
        }
      });

    return () => {
      cancelled = true; // cleanup: cancel any pending state updates
    };
  }, [userId]); // re-runs each time userId changes

  if (loading) return <p>Loading…</p>;
  return <p>{user?.name}</p>;
}

// Subscription pattern with cleanup
function WindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);

  useEffect(() => {
    const handler = () => setWidth(window.innerWidth);
    window.addEventListener("resize", handler);

    return () => {
      // Cleanup: remove listener when component unmounts
      // Without this, the handler would run forever (memory leak)
      window.removeEventListener("resize", handler);
    };
  }, []); // subscribe once on mount, unsubscribe on unmount

  return <p>Width: {width}px</p>;
}

// Using AbortController for cancellable fetches
function SearchResults({ query }: { query: string }) {
  const [results, setResults] = useState<string[]>([]);

  useEffect(() => {
    if (!query) return;

    const controller = new AbortController();

    fetch(`/api/search?q=${encodeURIComponent(query)}`, {
      signal: controller.signal,
    })
      .then((r) => r.json())
      .then(setResults)
      .catch((err) => {
        if (err.name !== "AbortError") console.error(err);
      });

    return () => controller.abort(); // cancel pending request on query change
  }, [query]);

  return <ul>{results.map((r) => <li key={r}>{r}</li>)}</ul>;
}
```

> ⚠️ **Missing dependency warning is almost always right.** If you suppress it with `// eslint-disable-next-line`, you almost certainly have a stale closure bug. Fix the dependency instead of silencing the lint rule.

### useRef

```tsx
import { useRef, useEffect, forwardRef, useImperativeHandle } from "react";

// DOM ref — direct element access
function FocusInput() {
  const inputRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    // Runs after mount — element exists in DOM
    inputRef.current?.focus();
  }, []);

  return <input ref={inputRef} placeholder="Auto-focused" />;
}

// Mutable value that does NOT trigger re-render
// Use for: interval IDs, previous values, counters you don't display
function Timer() {
  const intervalRef = useRef<ReturnType<typeof setInterval> | null>(null);
  const [running, setRunning] = useState(false);
  const ticksRef = useRef(0); // tracks ticks without causing renders

  const start = () => {
    setRunning(true);
    intervalRef.current = setInterval(() => {
      ticksRef.current += 1; // mutating ref — intentional, no re-render
      console.log(`Tick ${ticksRef.current}`);
    }, 1000);
  };

  const stop = () => {
    setRunning(false);
    if (intervalRef.current) clearInterval(intervalRef.current);
  };

  return (
    <div>
      <button onClick={start} disabled={running}>Start</button>
      <button onClick={stop} disabled={!running}>Stop</button>
    </div>
  );
}

// Storing the latest callback without re-subscribing effects
function useLatestCallback<T extends (...args: never[]) => unknown>(fn: T): T {
  const ref = useRef(fn);
  ref.current = fn; // always latest
  // Return stable wrapper that always calls the latest fn
  return useRef((...args: Parameters<T>) => ref.current(...args) as ReturnType<T>).current as T;
}

// forwardRef — expose a ref from a child component to a parent
const TextInput = forwardRef<HTMLInputElement, { label: string }>(
  ({ label }, ref) => (
    <label>
      {label}
      <input ref={ref} type="text" />
    </label>
  )
);
TextInput.displayName = "TextInput";

// useImperativeHandle — expose only specific methods (not the whole element)
interface DialogHandle {
  open: () => void;
  close: () => void;
}

const Dialog = forwardRef<DialogHandle, { children: React.ReactNode }>(
  ({ children }, ref) => {
    const [open, setOpen] = useState(false);

    useImperativeHandle(ref, () => ({
      open: () => setOpen(true),
      close: () => setOpen(false),
    }));

    if (!open) return null;
    return <div role="dialog">{children}</div>;
  }
);
Dialog.displayName = "Dialog";

function Parent() {
  const dialogRef = useRef<DialogHandle>(null);

  return (
    <>
      <button onClick={() => dialogRef.current?.open()}>Open</button>
      <Dialog ref={dialogRef}>
        <p>Hello from dialog</p>
        <button onClick={() => dialogRef.current?.close()}>Close</button>
      </Dialog>
    </>
  );
}
```

### useContext

```tsx
import { createContext, useContext, useState, useMemo, ReactNode } from "react";

interface ThemeContextValue {
  theme: "light" | "dark";
  toggle: () => void;
}

// 1. Create context with a null default (guarded by custom hook)
const ThemeContext = createContext<ThemeContextValue | null>(null);

// 2. Provider — memoize the value to prevent unnecessary consumer re-renders
function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<"light" | "dark">("light");

  // Without useMemo, a new object is created on every parent render
  // → all consumers re-render even if theme didn't change
  const value = useMemo(
    () => ({ theme, toggle: () => setTheme((t) => (t === "light" ? "dark" : "light")) }),
    [theme]
  );

  return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>;
}

// 3. Custom hook with guard — clear error if used outside provider
function useTheme(): ThemeContextValue {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error("useTheme must be used inside <ThemeProvider>");
  return ctx;
}

// 4. Consumer — only re-renders when ThemeContext value changes
function ThemedButton() {
  const { theme, toggle } = useTheme();

  return (
    <button
      onClick={toggle}
      style={{
        background: theme === "dark" ? "#1a1a1a" : "#ffffff",
        color: theme === "dark" ? "#ffffff" : "#1a1a1a",
        border: "1px solid currentColor",
      }}
    >
      Current theme: {theme}
    </button>
  );
}
```

> ⚠️ **Context re-renders ALL consumers when the value changes.** Split contexts by update frequency — ThemeContext, AuthContext, NotificationsContext — rather than one giant AppContext. A theme change shouldn't re-render auth consumers.

### useReducer

```tsx
import { useReducer, Dispatch } from "react";

// Prefer useReducer when:
// - State has multiple sub-values that update together
// - Next state depends on previous in complex, named ways
// - You want all state transitions documented in one place (testable independently)

type State = {
  count: number;
  step: number;
  history: number[];
};

type Action =
  | { type: "INCREMENT" }
  | { type: "DECREMENT" }
  | { type: "SET_STEP"; payload: number }
  | { type: "RESET" };

const initialState: State = { count: 0, step: 1, history: [] };

// Reducer is a pure function — easy to unit-test in isolation
function counterReducer(state: State, action: Action): State {
  switch (action.type) {
    case "INCREMENT": {
      const next = state.count + state.step;
      return { ...state, count: next, history: [...state.history, next] };
    }
    case "DECREMENT": {
      const next = state.count - state.step;
      return { ...state, count: next, history: [...state.history, next] };
    }
    case "SET_STEP":
      return { ...state, step: action.payload };
    case "RESET":
      return initialState;
    default:
      return state;
  }
}

function CounterWithHistory() {
  const [state, dispatch] = useReducer(counterReducer, initialState);

  return (
    <div>
      <p>Count: {state.count} (step: {state.step})</p>
      <button onClick={() => dispatch({ type: "INCREMENT" })}>+</button>
      <button onClick={() => dispatch({ type: "DECREMENT" })}>-</button>
      <input
        type="number"
        value={state.step}
        min={1}
        onChange={(e) =>
          dispatch({ type: "SET_STEP", payload: Number(e.target.value) })
        }
      />
      <button onClick={() => dispatch({ type: "RESET" })}>Reset</button>
      <p>History: {state.history.join(" → ")}</p>
    </div>
  );
}

// dispatch is always stable — safe to pass down without useCallback
// This makes useReducer + Context a good pattern for complex global state
const DispatchContext = createContext<Dispatch<Action> | null>(null);
```

### useMemo

```tsx
import { useMemo, useState } from "react";

function FilteredList({ items }: { items: { id: number; value: string }[] }) {
  const [query, setQuery] = useState("");
  const [sortAsc, setSortAsc] = useState(true);

  // Without useMemo: recalculates on every render
  // With useMemo: only recalculates when items, query, or sortAsc change
  const processed = useMemo(() => {
    const filtered = items.filter((item) =>
      item.value.toLowerCase().includes(query.toLowerCase())
    );
    return filtered.sort((a, b) =>
      sortAsc ? a.value.localeCompare(b.value) : b.value.localeCompare(a.value)
    );
  }, [items, query, sortAsc]);

  // Stable object reference for child components or effects
  const observerOptions = useMemo(
    () => ({ threshold: 0.5, rootMargin: "100px" }),
    [] // constant — computed once
  );

  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} placeholder="Filter..." />
      <button onClick={() => setSortAsc((a) => !a)}>
        Sort {sortAsc ? "↓" : "↑"}
      </button>
      <ul>
        {processed.map((item) => <li key={item.id}>{item.value}</li>)}
      </ul>
    </>
  );
}
```

> ⚠️ **Don't memoize everything.** `useMemo` itself has cost — memory, the comparison itself, indirection. Profile first. Only memoize when: (a) the computation measurably takes >1ms, or (b) you need reference equality for downstream `memo`/`useEffect`.

### useCallback

```tsx
import { useCallback, useState, memo } from "react";

// Child re-renders only when its props change (reference equality)
const Row = memo(function Row({
  item,
  onDelete,
}: {
  item: { id: number; name: string };
  onDelete: (id: number) => void;
}) {
  console.log(`Rendering Row ${item.id}`);
  return (
    <li>
      {item.name}
      <button onClick={() => onDelete(item.id)}>Delete</button>
    </li>
  );
});

function List() {
  const [items, setItems] = useState([
    { id: 1, name: "Apple" },
    { id: 2, name: "Banana" },
  ]);
  const [other, setOther] = useState(0);

  // WITHOUT useCallback: new function reference on every render
  // → Row re-renders on every "Change Other" click even though items didn't change

  // WITH useCallback: stable reference across renders
  // → Row only re-renders when items changes (which it doesn't on "Change Other")
  const handleDelete = useCallback((id: number) => {
    setItems((prev) => prev.filter((item) => item.id !== id));
  }, []); // no deps — only uses the functional form of setItems

  return (
    <div>
      <ul>
        {items.map((item) => (
          <Row key={item.id} item={item} onDelete={handleDelete} />
        ))}
      </ul>
      <button onClick={() => setOther((o) => o + 1)}>
        Change Other ({other})
      </button>
    </div>
  );
}
```

> 💡 **useCallback is only valuable when the stabilized function is passed to a `memo`'d child or used as a `useEffect` dependency.** Without either condition, `useCallback` only adds overhead.

### useLayoutEffect

```tsx
import { useLayoutEffect, useRef, useState, useEffect } from "react";

// Timeline comparison:
// useEffect:       render → commit → browser paint → effect
// useLayoutEffect: render → commit → effect → browser paint

// Use case: measure DOM and position an element before paint (no flicker)
function Tooltip({
  text,
  anchorRef,
}: {
  text: string;
  anchorRef: React.RefObject<HTMLElement>;
}) {
  const tooltipRef = useRef<HTMLDivElement>(null);
  const [style, setStyle] = useState<React.CSSProperties>({
    visibility: "hidden",
  });

  useLayoutEffect(() => {
    const anchor = anchorRef.current;
    const tooltip = tooltipRef.current;
    if (!anchor || !tooltip) return;

    const anchorRect = anchor.getBoundingClientRect();
    const tooltipRect = tooltip.getBoundingClientRect();

    // Calculate position — runs before paint, so no flicker
    setStyle({
      position: "fixed",
      top: anchorRect.bottom + 4,
      left: anchorRect.left + anchorRect.width / 2 - tooltipRect.width / 2,
      visibility: "visible",
    });
  });

  return (
    <div ref={tooltipRef} style={style} role="tooltip">
      {text}
    </div>
  );
}

// Safe SSR pattern: use useEffect as fallback to avoid SSR warning
const useIsomorphicLayoutEffect =
  typeof window !== "undefined" ? useLayoutEffect : useEffect;
```

---

## 5. Common Mistakes & Pitfalls

**1. Direct state mutation**
```tsx
// WRONG — same reference, React bails out (no re-render)
const [user, setUser] = useState({ name: "Alice", age: 30 });
const birthday = () => {
  user.age += 1; // mutation
  setUser(user); // same reference object
};

// CORRECT — new object
const birthday = () => setUser((prev) => ({ ...prev, age: prev.age + 1 }));
```

**2. Stale closure in intervals**
```tsx
// WRONG — count is captured as 0, never updates
const [count, setCount] = useState(0);
useEffect(() => {
  const id = setInterval(() => setCount(count + 1), 1000);
  return () => clearInterval(id);
}, []); // stale closure: count is always 0

// CORRECT — functional update never reads stale state
useEffect(() => {
  const id = setInterval(() => setCount((c) => c + 1), 1000);
  return () => clearInterval(id);
}, []);
```

**3. Missing cleanup causes memory leaks**
```tsx
// WRONG — if component unmounts mid-fetch, sets state on unmounted component
useEffect(() => {
  fetch("/api/data").then((r) => r.json()).then(setData);
}, []);

// CORRECT
useEffect(() => {
  const controller = new AbortController();
  fetch("/api/data", { signal: controller.signal })
    .then((r) => r.json())
    .then(setData)
    .catch((e) => { if (e.name !== "AbortError") throw e; });
  return () => controller.abort();
}, []);
```

**4. Object/array in dependency array causes infinite loop**
```tsx
// WRONG — { page: 1 } is a new object every render → effect loops
function Bad({ page }: { page: number }) {
  useEffect(() => {
    fetch(`/api?page=${page}`);
  }, [{ page }]); // new object each render!
}

// CORRECT — use primitive values as deps
function Good({ page }: { page: number }) {
  useEffect(() => {
    fetch(`/api?page=${page}`);
  }, [page]);
}
```

**5. Calling hooks conditionally**
```tsx
// WRONG — breaks the rules of hooks
function Bad({ show }: { show: boolean }) {
  if (show) {
    const [count, setCount] = useState(0); // conditional call!
  }
}

// CORRECT — always call hooks, conditionally use the value
function Good({ show }: { show: boolean }) {
  const [count, setCount] = useState(0);
  if (!show) return null;
  return <p>{count}</p>;
}
```

---

## 6. When to Use / Not Use

| Hook | Use when | Avoid when |
|------|----------|------------|
| `useState` | Simple local state | Deeply nested state with many transitions (prefer `useReducer`) |
| `useEffect` | Syncing with external systems (APIs, subscriptions, timers) | Computing derived values (just compute inline during render) |
| `useRef` | DOM access or mutable container that shouldn't trigger renders | Storing state that should trigger renders |
| `useContext` | Sharing state across the component tree | Passing props 1–2 levels (prop drilling at this depth is fine) |
| `useReducer` | Complex state with named transitions, multiple sub-values | Simple single-value boolean/string/number state |
| `useMemo` | Expensive computed values or stable object/array references | Simple values — the overhead outweighs the benefit |
| `useCallback` | Functions passed to `memo`'d children or used in `useEffect` deps | Functions not passed anywhere stable |
| `useLayoutEffect` | DOM measurement before paint (tooltips, popovers, animations) | Server-rendered components (use `useEffect` or `useIsomorphicLayoutEffect`) |

---

## 7. Real-World Scenario

A live dashboard needs: fetch user on ID change, subscribe to WebSocket for live updates, and debounce a search input to avoid excessive API calls.

```tsx
import { useState, useEffect, useRef, useCallback } from "react";

// Reusable custom hook: debounce any value
function useDebouncedValue<T>(value: T, delay: number): T {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debounced;
}

// Reusable custom hook: stable WebSocket subscription
function useWebSocket(url: string, onMessage: (data: unknown) => void) {
  // Store callback in ref so we never need to re-subscribe when it changes
  const callbackRef = useRef(onMessage);
  callbackRef.current = onMessage;

  useEffect(() => {
    const ws = new WebSocket(url);
    ws.onmessage = (e) => {
      try {
        callbackRef.current(JSON.parse(e.data));
      } catch {
        // ignore malformed messages
      }
    };
    return () => ws.close();
  }, [url]); // only re-subscribe when URL changes
}

function Dashboard({ userId }: { userId: string }) {
  const [search, setSearch] = useState("");
  const [searchResults, setSearchResults] = useState<string[]>([]);
  const [liveEvents, setLiveEvents] = useState<unknown[]>([]);
  const debouncedSearch = useDebouncedValue(search, 300);

  // WebSocket for live updates — stable reference via ref
  useWebSocket(`wss://api.example.com/live/${userId}`, (data) => {
    setLiveEvents((prev) => [data, ...prev].slice(0, 50));
  });

  // Fetch search results when debounced query changes
  useEffect(() => {
    if (!debouncedSearch.trim()) {
      setSearchResults([]);
      return;
    }

    const controller = new AbortController();

    fetch(`/api/search?q=${encodeURIComponent(debouncedSearch)}&user=${userId}`, {
      signal: controller.signal,
    })
      .then((r) => r.json())
      .then(setSearchResults)
      .catch((e) => { if (e.name !== "AbortError") console.error(e); });

    return () => controller.abort();
  }, [debouncedSearch, userId]);

  return (
    <div>
      <input
        value={search}
        onChange={(e) => setSearch(e.target.value)}
        placeholder="Search…"
      />
      {searchResults.length > 0 && (
        <ul>
          {searchResults.map((r) => <li key={r}>{r}</li>)}
        </ul>
      )}
      <section>
        <h3>Live Events</h3>
        <ul>
          {liveEvents.map((e, i) => <li key={i}>{JSON.stringify(e)}</li>)}
        </ul>
      </section>
    </div>
  );
}
```

---

## 8. Interview Questions

**Q1: Why can't you call hooks conditionally?**

React identifies each hook by its position in the call order. It maintains a linked list where position N always corresponds to the same hook. If a hook is inside an `if`, the list position of all subsequent hooks shifts depending on the condition — React reads the wrong state for each one. This is not a design choice that could be relaxed — it is fundamental to how hooks work without a runtime object per hook.

**Q2: When does useEffect run?**

After every committed render by default (no dependency array). With an empty array `[]`, only after the first render (mount). With a deps array, after the first render and after any subsequent render where at least one dep changed (compared with `Object.is`). The cleanup function runs before the next effect and on unmount.

**Q3: Difference between useMemo and useCallback?**

`useMemo` memoizes the **return value** of a function: `useMemo(() => compute(), [deps])` returns the computed result. `useCallback` memoizes the **function itself**: `useCallback(fn, deps)` returns a stable function reference. Technically: `useCallback(fn, deps)` is equivalent to `useMemo(() => fn, deps)`.

**Q4: What is useLayoutEffect for?**

It fires synchronously after all DOM mutations but before the browser paints the screen. Use it when you need to measure a DOM node and update something — like a tooltip position — before the user sees it. If you use `useEffect` for this, you get a visible flicker (wrong position, then correct position). For most effects, prefer `useEffect`.

**Q5: When would you choose useReducer over useState?**

When state has multiple interconnected sub-values that update together, when transitions are complex enough to deserve named actions (INCREMENT, RESET, SET_STEP), or when you want to test the state logic independently of the component. Also useful when passing dispatch deep in the tree — `dispatch` is always stable (no `useCallback` needed), unlike many setter functions.

**Q6: How do you debounce in React?**

Write a `useDebouncedValue` custom hook that stores the debounced value in state, sets a `setTimeout` in a `useEffect`, and clears it on cleanup. The component uses the debounced value for API calls. Never debounce the setState call directly (you'd still update on every keystroke). Alternatively use a library like `use-debounce`.

**Q7: What is a custom hook?**

A function whose name starts with `use` that can call other hooks. It is the primary mechanism for extracting and sharing stateful logic between components without restructuring the component tree. Unlike HOCs or render props, custom hooks don't add wrapper components. Examples: `useFetch`, `useLocalStorage`, `useDebounce`, `useIntersectionObserver`.

**Q8: What happens if you don't include a dependency in useEffect?**

You get a stale closure. The effect captures the value from the render it was created in and never sees updates. This typically shows up as a timer or subscription always operating on the initial value, or a fetch that ignores prop changes. The ESLint `exhaustive-deps` rule catches this. The fix is usually: add the dep, use a functional state update to avoid reading the dep, or move the value into a ref.

---

## 9. Exercises

**Exercise 1: useFetch hook**

Build `useFetch<T>(url: string)` returning `{ data: T | null; loading: boolean; error: Error | null }`. Handle abort on cleanup and re-fetch when `url` changes.

```tsx
// Solution
function useFetch<T>(url: string) {
  const [state, setState] = useState<{
    data: T | null;
    loading: boolean;
    error: Error | null;
  }>({ data: null, loading: true, error: null });

  useEffect(() => {
    const controller = new AbortController();
    setState({ data: null, loading: true, error: null });

    fetch(url, { signal: controller.signal })
      .then((r) => {
        if (!r.ok) throw new Error(`HTTP ${r.status}`);
        return r.json() as Promise<T>;
      })
      .then((data) => setState({ data, loading: false, error: null }))
      .catch((err) => {
        if (err.name !== "AbortError") {
          setState({ data: null, loading: false, error: err as Error });
        }
      });

    return () => controller.abort();
  }, [url]);

  return state;
}
```

**Exercise 2: useDebounce hook**

Build `useDebounce<T>(value: T, delay: number): T`. Returns the debounced value that updates only after `delay` ms of inactivity.

```tsx
// Hint: setTimeout in useEffect, clearTimeout in cleanup
function useDebounce<T>(value: T, delay: number): T {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);
  return debounced;
}
```

**Exercise 3: useLocalStorage hook**

Build `useLocalStorage<T>(key: string, initialValue: T): [T, (val: T) => void]`. Should sync state to localStorage on update and read from localStorage on mount.

```tsx
// Solution
function useLocalStorage<T>(key: string, initialValue: T) {
  const [value, setValue] = useState<T>(() => {
    try {
      const item = localStorage.getItem(key);
      return item !== null ? (JSON.parse(item) as T) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setStoredValue = useCallback(
    (val: T) => {
      setValue(val);
      try {
        localStorage.setItem(key, JSON.stringify(val));
      } catch {
        console.warn(`Failed to write localStorage key "${key}"`);
      }
    },
    [key]
  );

  return [value, setStoredValue] as const;
}
```

**Exercise 4: useIntersectionObserver**

Build `useIntersectionObserver(ref: RefObject<Element>, options?: IntersectionObserverInit): boolean` that returns `true` when the element enters the viewport — useful for lazy loading.

```tsx
// Solution
function useIntersectionObserver(
  ref: React.RefObject<Element>,
  options?: IntersectionObserverInit
): boolean {
  const [isVisible, setIsVisible] = useState(false);

  useEffect(() => {
    const el = ref.current;
    if (!el) return;

    const observer = new IntersectionObserver(([entry]) => {
      setIsVisible(entry.isIntersecting);
    }, options);

    observer.observe(el);
    return () => observer.disconnect();
  }, [ref, options]);

  return isVisible;
}

// Usage:
function LazyImage({ src, alt }: { src: string; alt: string }) {
  const ref = useRef<HTMLDivElement>(null);
  const isVisible = useIntersectionObserver(ref, { threshold: 0.1 });
  return (
    <div ref={ref} style={{ minHeight: 200 }}>
      {isVisible && <img src={src} alt={alt} loading="lazy" />}
    </div>
  );
}
```

---

## 10. Further Reading

- [React Docs — Hooks Reference](https://react.dev/reference/react)
- [React Docs — Synchronizing with Effects](https://react.dev/learn/synchronizing-with-effects)
- [React Docs — You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect)
- [Dan Abramov — A Complete Guide to useEffect](https://overreacted.io/a-complete-guide-to-useeffect/)
- [Kent C. Dodds — When to useMemo and useCallback](https://kentcdodds.com/blog/usememo-and-usecallback)
- [React Docs — Rules of Hooks](https://react.dev/reference/rules/rules-of-hooks)
- [TkDodo — Practical React Query](https://tkdodo.eu/blog/practical-react-query) (covers data-fetching hooks)
