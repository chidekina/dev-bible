# React Cheatsheet

## Hooks Reference

| Hook | Purpose | Key rule |
|------|---------|----------|
| `useState` | Local component state | Setter is async — read next render |
| `useEffect` | Side effects, subscriptions | Return cleanup fn; watch deps carefully |
| `useLayoutEffect` | DOM measurement before paint | Blocks paint; prefer `useEffect` |
| `useRef` | Mutable value without re-render | DOM refs, prev value, timer IDs |
| `useMemo` | Memoize expensive computation | Only when profiling shows it helps |
| `useCallback` | Stable function reference | Required when fn passed to memoized child |
| `useContext` | Read nearest context value | Does not prevent re-renders on unrelated state |
| `useReducer` | Complex state with transitions | Prefer when state has multiple sub-values |
| `useId` | Stable unique ID | For `aria-*` / `htmlFor` in SSR |
| `useTransition` | Mark update as non-urgent | Keeps UI responsive during heavy renders |
| `useDeferredValue` | Defer expensive child render | Alternative to debounce for derived state |
| `useImperativeHandle` | Expose imperative API via ref | Rare; prefer props for control |
| `useSyncExternalStore` | Subscribe to external stores | Library authors; not for app code |

---

## useState

```tsx
const [count, setCount] = useState(0);
const [user, setUser] = useState<User | null>(null);

// Lazy initial state (expensive initialization runs only once)
const [items, setItems] = useState(() => JSON.parse(localStorage.getItem('items') ?? '[]'));

// Functional update — always use when new state depends on old state
setCount((prev) => prev + 1);

// Partial update for objects
setUser((prev) => prev ? { ...prev, name: 'Alice' } : null);
```

---

## useEffect

```tsx
// Run on every render (rarely what you want)
useEffect(() => { /* ... */ });

// Run once on mount
useEffect(() => { /* ... */ }, []);

// Run when dep changes
useEffect(() => {
  const controller = new AbortController();
  fetchData(id, controller.signal).then(setData);
  return () => controller.abort(); // cleanup
}, [id]);

// Subscription pattern
useEffect(() => {
  const handler = (e: Event) => { /* ... */ };
  window.addEventListener('resize', handler);
  return () => window.removeEventListener('resize', handler);
}, []);
```

**Deps lint rule:** include every reactive value read inside the effect. If you need to escape the dep rule, use a `useRef` for the value.

---

## useRef

```tsx
// DOM ref
const inputRef = useRef<HTMLInputElement>(null);
inputRef.current?.focus();

// Mutable value — changing does not trigger re-render
const timerRef = useRef<NodeJS.Timeout | null>(null);
timerRef.current = setTimeout(() => {}, 1000);

// Previous value pattern
function usePrevious<T>(value: T) {
  const ref = useRef<T>(undefined);
  useEffect(() => { ref.current = value; });
  return ref.current;
}
```

---

## useMemo / useCallback

```tsx
// useMemo — expensive derivation
const sortedItems = useMemo(
  () => [...items].sort((a, b) => a.name.localeCompare(b.name)),
  [items]
);

// useCallback — stable handler for memoized child
const handleDelete = useCallback(
  (id: string) => dispatch({ type: 'DELETE', id }),
  [dispatch] // dispatch from useReducer is stable
);

// React.memo — skip re-render when props unchanged
const ItemRow = React.memo(function ItemRow({ item, onDelete }: Props) {
  return <div onClick={() => onDelete(item.id)}>{item.name}</div>;
});
```

**When NOT to memo:** Simple components, components that always get new props, when profiling does not show a problem.

---

## useReducer

```tsx
type State = { count: number; step: number };
type Action =
  | { type: 'INCREMENT' }
  | { type: 'DECREMENT' }
  | { type: 'RESET' }
  | { type: 'SET_STEP'; step: number };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'INCREMENT': return { ...state, count: state.count + state.step };
    case 'DECREMENT': return { ...state, count: state.count - state.step };
    case 'RESET':     return { count: 0, step: state.step };
    case 'SET_STEP':  return { ...state, step: action.step };
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0, step: 1 });
  return <button onClick={() => dispatch({ type: 'INCREMENT' })}>{state.count}</button>;
}
```

---

## Context

```tsx
// Define context with a null default — forces consumers to check for Provider
const ThemeContext = createContext<Theme | null>(null);

// Custom hook enforces provider presence
export function useTheme(): Theme {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error('useTheme must be used within ThemeProvider');
  return ctx;
}

// Provider
function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<Theme>('light');
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}
```

**Context re-render caveat:** Every consumer re-renders when the context value changes. Split context into separate providers if state changes at different frequencies (e.g., user vs theme vs notifications).

---

## Compound Components

```tsx
// Parent exposes sub-components via static properties
function Tabs({ children, defaultTab }: TabsProps) {
  const [activeTab, setActiveTab] = useState(defaultTab);
  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      {children}
    </TabsContext.Provider>
  );
}

Tabs.List = function TabList({ children }: { children: React.ReactNode }) {
  return <div role="tablist">{children}</div>;
};

Tabs.Tab = function Tab({ id, children }: { id: string; children: React.ReactNode }) {
  const { activeTab, setActiveTab } = useTabsContext();
  return (
    <button
      role="tab"
      aria-selected={activeTab === id}
      onClick={() => setActiveTab(id)}
    >
      {children}
    </button>
  );
};

Tabs.Panel = function TabPanel({ id, children }: { id: string; children: React.ReactNode }) {
  const { activeTab } = useTabsContext();
  if (activeTab !== id) return null;
  return <div role="tabpanel">{children}</div>;
};

// Usage
<Tabs defaultTab="overview">
  <Tabs.List>
    <Tabs.Tab id="overview">Overview</Tabs.Tab>
    <Tabs.Tab id="settings">Settings</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panel id="overview"><Overview /></Tabs.Panel>
  <Tabs.Panel id="settings"><Settings /></Tabs.Panel>
</Tabs>
```

---

## Render Patterns

```tsx
// Conditional rendering
{isLoggedIn && <Dashboard />}
{isLoggedIn ? <Dashboard /> : <Login />}

// Early return (cleaner for complex conditions)
function Page() {
  if (isLoading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;
  return <Content data={data} />;
}

// Render props
function MouseTracker({ render }: { render: (pos: { x: number; y: number }) => React.ReactNode }) {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  return <div onMouseMove={(e) => setPos({ x: e.clientX, y: e.clientY })}>{render(pos)}</div>;
}

// Children as function (same pattern, different API)
<MouseTracker>{(pos) => <span>{pos.x}, {pos.y}</span>}</MouseTracker>
```

---

## Error Boundaries

```tsx
// Class component required — no hook equivalent
class ErrorBoundary extends React.Component<
  { fallback: React.ReactNode; children: React.ReactNode },
  { hasError: boolean; error: Error | null }
> {
  state = { hasError: false, error: null };

  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, info: React.ErrorInfo) {
    reportToSentry(error, info.componentStack);
  }

  render() {
    if (this.state.hasError) return this.props.fallback;
    return this.props.children;
  }
}

// Usage
<ErrorBoundary fallback={<ErrorPage />}>
  <App />
</ErrorBoundary>

// react-error-boundary library — hook-friendly
import { ErrorBoundary, useErrorBoundary } from 'react-error-boundary';

function DataLoader() {
  const { showBoundary } = useErrorBoundary();
  useEffect(() => {
    fetchData().catch(showBoundary); // throw errors into boundary
  }, []);
}
```

---

## Portals

```tsx
import { createPortal } from 'react-dom';

function Modal({ isOpen, onClose, children }: ModalProps) {
  if (!isOpen) return null;

  return createPortal(
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal-content" onClick={(e) => e.stopPropagation()}>
        {children}
      </div>
    </div>,
    document.getElementById('modal-root')! // render outside component tree
  );
}
```

---

## TypeScript with React

```tsx
// Component props
type ButtonProps = {
  variant: 'primary' | 'secondary' | 'ghost';
  size?: 'sm' | 'md' | 'lg';
  onClick?: (event: React.MouseEvent<HTMLButtonElement>) => void;
  children: React.ReactNode;
  disabled?: boolean;
} & React.ButtonHTMLAttributes<HTMLButtonElement>; // extend HTML attrs

// Generic component
function List<T extends { id: string }>({
  items,
  renderItem,
}: {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
}) {
  return <ul>{items.map((item) => <li key={item.id}>{renderItem(item)}</li>)}</ul>;
}

// forwardRef with TypeScript
const Input = React.forwardRef<HTMLInputElement, InputProps>(
  function Input({ label, ...props }, ref) {
    return (
      <label>
        {label}
        <input ref={ref} {...props} />
      </label>
    );
  }
);

// Event handlers
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => setValue(e.target.value);
const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => { e.preventDefault(); /* ... */ };
const handleKeyDown = (e: React.KeyboardEvent<HTMLDivElement>) => { /* ... */ };

// useRef typed for nullable DOM node
const ref = useRef<HTMLDivElement>(null); // null = not attached yet
const valueRef = useRef<string>('');      // non-null = mutable box

// Children types
React.ReactNode      // anything renderable (string, element, null, array)
React.ReactElement   // JSX element only
React.FC             // avoid — doesn't add much, removes displayName inference
```

---

## Controlled Inputs

```tsx
// Controlled (React owns value)
function ControlledInput() {
  const [value, setValue] = useState('');
  return (
    <input
      value={value}
      onChange={(e) => setValue(e.target.value)}
    />
  );
}

// Uncontrolled (DOM owns value, accessed via ref)
function UncontrolledForm() {
  const nameRef = useRef<HTMLInputElement>(null);
  const handleSubmit = () => console.log(nameRef.current?.value);
  return <input ref={nameRef} defaultValue="" />;
}

// File inputs are always uncontrolled
const fileRef = useRef<HTMLInputElement>(null);
const file = fileRef.current?.files?.[0];
```

---

## Performance Patterns

```tsx
// useTransition — mark state update as low priority
const [isPending, startTransition] = useTransition();

function handleSearch(query: string) {
  setInputValue(query); // urgent: updates input immediately
  startTransition(() => {
    setSearchQuery(query); // non-urgent: can be interrupted
  });
}

// useDeferredValue — defer expensive child render
const deferredQuery = useDeferredValue(searchQuery);
// deferredQuery lags behind searchQuery; pass to expensive child
<SearchResults query={deferredQuery} />

// Lazy loading with Suspense
const HeavyComponent = React.lazy(() => import('./HeavyComponent'));

<Suspense fallback={<Skeleton />}>
  <HeavyComponent />
</Suspense>

// Code splitting by route
const SettingsPage = React.lazy(() => import('./pages/Settings'));
```

---

## Key Patterns to Remember

- **Keys must be stable and unique** — use IDs, not array indexes (indexes break animations and state)
- **State should be minimal** — derive everything you can from existing state
- **Lift state up** when siblings need to share it; move it down when only one component uses it
- **Avoid derived state in state** — compute from props/state in render instead
- **Effects are for synchronization** — if you're using an effect to update state when props change, consider deriving the value instead
- **Batching:** React 18 batches all state updates including those in timeouts and promises
