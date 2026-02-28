# React Patterns

## 1. What & Why

React's component model is a small API with enormous expressive power. The patterns in this file are recurring solutions to recurring problems: sharing state implicitly between a parent and its children, reusing behavior across unrelated components, separating data concerns from display concerns, and giving consumers control over behavior without opening up internals.

Knowing these patterns by name and knowing when to apply each one is the difference between a React developer who copies code and one who designs systems.

---

## 2. Core Concepts

- **Composition over inheritance** — React's fundamental design principle. Build complex UIs by combining smaller, focused components.
- **Inversion of control** — some patterns (render props, state reducer) deliberately hand control back to the consumer. Flexibility at the cost of API surface.
- **Implicit vs explicit state sharing** — Context shares state implicitly (no prop threading); direct props are explicit and traceable.
- **Co-location** — keep state as close to where it's used as possible. Lift only when necessary.

---

## 3. How It Works

Each pattern is a specific way of using React's core primitives (components, props, context, state) to solve a recurring problem. None requires special APIs — they are architectural decisions about how components relate to each other.

---

## 4. Code Examples

### Compound Components

Components that share implicit state through Context — the parent manages state, children read and react to it without explicit prop threading.

```tsx
import {
  createContext,
  useContext,
  useState,
  ReactNode,
  HTMLAttributes,
} from "react";

// ─── Tabs ──────────────────────────────────────────────────────────────────

interface TabsContextValue {
  active: string;
  setActive: (id: string) => void;
}

const TabsContext = createContext<TabsContextValue | null>(null);

function useTabsContext() {
  const ctx = useContext(TabsContext);
  if (!ctx) throw new Error("Tab/TabPanel must be used inside <Tabs>");
  return ctx;
}

function Tabs({
  defaultTab,
  children,
}: {
  defaultTab: string;
  children: ReactNode;
}) {
  const [active, setActive] = useState(defaultTab);
  return (
    <TabsContext.Provider value={{ active, setActive }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}

function TabList({ children }: { children: ReactNode }) {
  return <div role="tablist">{children}</div>;
}

function Tab({ id, children }: { id: string; children: ReactNode }) {
  const { active, setActive } = useTabsContext();
  return (
    <button
      role="tab"
      aria-selected={active === id}
      onClick={() => setActive(id)}
    >
      {children}
    </button>
  );
}

function TabPanel({ id, children }: { id: string; children: ReactNode }) {
  const { active } = useTabsContext();
  if (active !== id) return null;
  return <div role="tabpanel">{children}</div>;
}

// Attach sub-components as static properties for ergonomic usage
Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panel = TabPanel;

// Usage — the parent doesn't need to thread "active" into every child:
function App() {
  return (
    <Tabs defaultTab="overview">
      <Tabs.List>
        <Tabs.Tab id="overview">Overview</Tabs.Tab>
        <Tabs.Tab id="details">Details</Tabs.Tab>
        <Tabs.Tab id="history">History</Tabs.Tab>
      </Tabs.List>
      <Tabs.Panel id="overview">Overview content</Tabs.Panel>
      <Tabs.Panel id="details">Details content</Tabs.Panel>
      <Tabs.Panel id="history">History content</Tabs.Panel>
    </Tabs>
  );
}
```

> 💡 **Compound components shine for UI kits** — Accordion, Select, Menu, Combobox. They give users the flexibility to compose and rearrange children without exposing raw state.

### Render Props

Pass a function as a prop that the component calls to render content. The component provides data; the consumer controls presentation.

```tsx
interface MousePosition {
  x: number;
  y: number;
}

// The component handles all the behavior
function MouseTracker({
  render,
}: {
  render: (position: MousePosition) => ReactNode;
}) {
  const [position, setPosition] = useState<MousePosition>({ x: 0, y: 0 });

  return (
    <div
      style={{ width: "100%", height: 400, border: "1px solid #ccc" }}
      onMouseMove={(e) => setPosition({ x: e.clientX, y: e.clientY })}
    >
      {render(position)}
    </div>
  );
}

// Consumers decide what to display
function CrosshairDemo() {
  return (
    <MouseTracker
      render={({ x, y }) => (
        <svg style={{ pointerEvents: "none" }}>
          <line x1={x - 10} y1={y} x2={x + 10} y2={y} stroke="red" />
          <line x1={x} y1={y - 10} x2={x} y2={y + 10} stroke="red" />
        </svg>
      )}
    />
  );
}

// Children-as-function is a common variant (same idea, different prop name)
function Toggle({ children }: { children: (on: boolean, toggle: () => void) => ReactNode }) {
  const [on, setOn] = useState(false);
  return <>{children(on, () => setOn((v) => !v))}</>;
}

// Usage:
function ToggleDemo() {
  return (
    <Toggle>
      {(on, toggle) => (
        <button onClick={toggle}>{on ? "ON" : "OFF"}</button>
      )}
    </Toggle>
  );
}
```

> 💡 **When render props are better than HOCs:** when you need to pass data from deep in a lifecycle to the render output, or when you want the consumer to compose multiple behaviors without nesting.

> ⚠️ **Render props create new function references every render.** If the child is `React.memo`'d, memoize the render function with `useCallback` or use hooks instead (they have the same power without the JSX overhead).

### Higher-Order Components (HOC)

A function that takes a component and returns an enhanced component with additional behavior.

```tsx
import { ComponentType, useEffect, useRef } from "react";

// withAuth — redirect unauthenticated users
function withAuth<P extends object>(WrappedComponent: ComponentType<P>) {
  function AuthGuard(props: P) {
    const isAuthenticated = useAuthStore((s) => s.isAuthenticated);

    if (!isAuthenticated) {
      return <Redirect to="/login" />;
    }

    return <WrappedComponent {...props} />;
  }

  // Important: preserve the display name for debugging in React DevTools
  AuthGuard.displayName = `withAuth(${WrappedComponent.displayName ?? WrappedComponent.name})`;

  return AuthGuard;
}

// withLogger — log render timing
function withLogger<P extends object>(
  WrappedComponent: ComponentType<P>,
  name?: string
) {
  function Logger(props: P) {
    const renderCount = useRef(0);
    renderCount.current += 1;

    useEffect(() => {
      const componentName = name ?? WrappedComponent.name;
      console.log(`[${componentName}] render #${renderCount.current}`);
    });

    return <WrappedComponent {...props} />;
  }

  Logger.displayName = `withLogger(${name ?? WrappedComponent.name})`;
  return Logger;
}

// Usage
function Dashboard(props: { userId: string }) {
  return <div>Dashboard for {props.userId}</div>;
}

const ProtectedDashboard = withAuth(withLogger(Dashboard));

// ─── HOC Drawbacks ──────────────────────────────────────────────────────────
// 1. Wrapper hell in React DevTools: withAuth(withLogger(withTheme(Dashboard)))
// 2. Prop naming conflicts — HOC props can shadow wrapped component props
// 3. Static methods are not forwarded automatically
// 4. Refs don't pass through without forwardRef
// 5. TypeScript inference can break with deep HOC composition
```

> ⚠️ **HOCs are largely superseded by hooks for logic reuse.** Use them when you need to wrap a component imperatively (e.g., from a third-party library that expects a component class) or when integrating with systems that don't support hooks directly.

### Controlled vs Uncontrolled Components

```tsx
import { useState, useRef } from "react";

// ─── Controlled: state lives in React ───────────────────────────────────────
// Single source of truth. Easier to validate, synchronize, and test.

function ControlledForm() {
  const [name, setName] = useState("");
  const [email, setEmail] = useState("");

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    // State is always up-to-date here
    console.log({ name, email });
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={name}           // controlled: value comes from React state
        onChange={(e) => setName(e.target.value)}
        placeholder="Name"
      />
      <input
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        type="email"
        placeholder="Email"
      />
      <button type="submit">Submit</button>
    </form>
  );
}

// ─── Uncontrolled: state lives in the DOM ───────────────────────────────────
// Access values via ref.current.value. Simpler when you don't need live validation
// or to synchronize the value with other state.

function UncontrolledForm() {
  const nameRef = useRef<HTMLInputElement>(null);
  const emailRef = useRef<HTMLInputElement>(null);

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    // Read DOM values at submit time only
    console.log({
      name: nameRef.current?.value,
      email: emailRef.current?.value,
    });
  };

  return (
    <form onSubmit={handleSubmit}>
      <input ref={nameRef} defaultValue="" placeholder="Name" />
      <input ref={emailRef} defaultValue="" type="email" placeholder="Email" />
      <button type="submit">Submit</button>
    </form>
  );
}

// ─── Hybrid: expose both controlled and uncontrolled from a component ───────
// The "value" + "onChange" controlled contract — same as HTML inputs

interface InputProps {
  value?: string;
  defaultValue?: string;
  onChange?: (value: string) => void;
}

function SmartInput({ value, defaultValue = "", onChange }: InputProps) {
  // Internal state used only when uncontrolled (no value prop)
  const [internalValue, setInternalValue] = useState(defaultValue);
  const isControlled = value !== undefined;
  const currentValue = isControlled ? value : internalValue;

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    if (!isControlled) setInternalValue(e.target.value);
    onChange?.(e.target.value);
  };

  return <input value={currentValue} onChange={handleChange} />;
}
```

**When to use each:**

| Scenario | Use |
|----------|-----|
| Live validation as user types | Controlled |
| Dependent fields (select country → filter cities) | Controlled |
| Simple form, read at submit only | Uncontrolled |
| Integrating with non-React libraries | Uncontrolled (ref) |
| Form library (React Hook Form) | Uncontrolled internally, abstracts it |

### Provider Pattern

Context + custom hook that throws when used outside the provider — a defensive, ergonomic pattern for feature-scoped state.

```tsx
import { createContext, useContext, useState, ReactNode } from "react";

// Auth provider — a real-world pattern
interface User {
  id: string;
  name: string;
  email: string;
  roles: string[];
}

interface AuthContextValue {
  user: User | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
  hasRole: (role: string) => boolean;
}

const AuthContext = createContext<AuthContextValue | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);

  const login = async (email: string, password: string) => {
    const res = await fetch("/api/auth/login", {
      method: "POST",
      body: JSON.stringify({ email, password }),
      headers: { "Content-Type": "application/json" },
    });
    if (!res.ok) throw new Error("Invalid credentials");
    const data = await res.json() as User;
    setUser(data);
  };

  const logout = () => {
    setUser(null);
    fetch("/api/auth/logout", { method: "POST" });
  };

  const hasRole = (role: string) => user?.roles.includes(role) ?? false;

  return (
    <AuthContext.Provider value={{ user, login, logout, hasRole }}>
      {children}
    </AuthContext.Provider>
  );
}

// The guard ensures misuse is a loud error, not a silent bug
export function useAuth(): AuthContextValue {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error("useAuth must be used inside <AuthProvider>");
  return ctx;
}

// Usage in any component deep in the tree
function ProfileMenu() {
  const { user, logout, hasRole } = useAuth();
  return (
    <div>
      <p>Hello, {user?.name}</p>
      {hasRole("admin") && <a href="/admin">Admin Panel</a>}
      <button onClick={logout}>Sign Out</button>
    </div>
  );
}
```

### Container / Presentational Split

Separate data-fetching and business logic (container) from rendering (presentational). Historically done with two components; today mostly done with a single component + custom hook.

```tsx
// ─── Presentational — pure rendering, no side effects ───────────────────────
interface UserCardProps {
  name: string;
  email: string;
  avatarUrl: string;
  isLoading: boolean;
  error: string | null;
  onRetry: () => void;
}

function UserCard({ name, email, avatarUrl, isLoading, error, onRetry }: UserCardProps) {
  if (isLoading) return <div className="skeleton" aria-busy="true" />;
  if (error) return (
    <div role="alert">
      <p>{error}</p>
      <button onClick={onRetry}>Retry</button>
    </div>
  );
  return (
    <div className="user-card">
      <img src={avatarUrl} alt={`${name}'s avatar`} />
      <h2>{name}</h2>
      <p>{email}</p>
    </div>
  );
}

// ─── Container — data fetching and state management ──────────────────────────
function UserCardContainer({ userId }: { userId: string }) {
  const { data, loading, error, refetch } = useFetch<{
    name: string;
    email: string;
    avatarUrl: string;
  }>(`/api/users/${userId}`);

  return (
    <UserCard
      name={data?.name ?? ""}
      email={data?.email ?? ""}
      avatarUrl={data?.avatarUrl ?? ""}
      isLoading={loading}
      error={error?.message ?? null}
      onRetry={refetch}
    />
  );
}

// Modern approach: custom hook does the separation without two components
function useUser(userId: string) {
  const { data, loading, error, refetch } = useFetch<User>(`/api/users/${userId}`);
  return { user: data, loading, error, refetch };
}

function UserProfile({ userId }: { userId: string }) {
  const { user, loading, error, refetch } = useUser(userId);
  return <UserCard {...{ name: user?.name ?? "", isLoading: loading, error: error?.message ?? null, onRetry: refetch, email: user?.email ?? "", avatarUrl: user?.avatarUrl ?? "" }} />;
}
```

### Slot Pattern

Use `children` and named props to create composable, flexible layouts — analogous to HTML `<slot>` in web components.

```tsx
import { ReactNode } from "react";

// Generic Card with named slots
interface CardProps {
  header?: ReactNode;
  children: ReactNode; // default slot (body)
  footer?: ReactNode;
  actions?: ReactNode;
}

function Card({ header, children, footer, actions }: CardProps) {
  return (
    <article className="card">
      {header && <header className="card__header">{header}</header>}
      <div className="card__body">{children}</div>
      {footer && <footer className="card__footer">{footer}</footer>}
      {actions && <div className="card__actions">{actions}</div>}
    </article>
  );
}

// Consumer composes freely — no prop-drilling for layout
function ProductCard({ product }: { product: { name: string; price: number; description: string } }) {
  return (
    <Card
      header={<h3>{product.name}</h3>}
      footer={<span className="price">${product.price}</span>}
      actions={
        <>
          <button>Add to Cart</button>
          <button>Wishlist</button>
        </>
      }
    >
      <p>{product.description}</p>
    </Card>
  );
}

// Modal with slot pattern
function Modal({
  title,
  children,
  footer,
  onClose,
}: {
  title: string;
  children: ReactNode;
  footer?: ReactNode;
  onClose: () => void;
}) {
  return (
    <div role="dialog" aria-modal="true" aria-labelledby="modal-title">
      <div className="modal__header">
        <h2 id="modal-title">{title}</h2>
        <button onClick={onClose} aria-label="Close">×</button>
      </div>
      <div className="modal__body">{children}</div>
      {footer && <div className="modal__footer">{footer}</div>}
    </div>
  );
}
```

### State Reducer Pattern

Invert control by letting consumers provide their own reducer or override specific actions — useful for highly flexible component libraries.

```tsx
import { useReducer, ReactNode } from "react";

type ToggleState = { on: boolean };
type ToggleAction = { type: "TOGGLE" } | { type: "RESET" } | { type: "ON" } | { type: "OFF" };

function defaultToggleReducer(state: ToggleState, action: ToggleAction): ToggleState {
  switch (action.type) {
    case "TOGGLE": return { on: !state.on };
    case "ON":     return { on: true };
    case "OFF":    return { on: false };
    case "RESET":  return { on: false };
    default:       return state;
  }
}

interface ToggleProps {
  initialOn?: boolean;
  // Consumer can override the reducer to customize behavior
  reducer?: (state: ToggleState, action: ToggleAction) => ToggleState;
  children: (args: {
    on: boolean;
    toggle: () => void;
    reset: () => void;
    dispatch: React.Dispatch<ToggleAction>;
  }) => ReactNode;
}

function Toggle({
  initialOn = false,
  reducer = defaultToggleReducer,
  children,
}: ToggleProps) {
  const [{ on }, dispatch] = useReducer(reducer, { on: initialOn });

  return (
    <>
      {children({
        on,
        toggle: () => dispatch({ type: "TOGGLE" }),
        reset: () => dispatch({ type: "RESET" }),
        dispatch,
      })}
    </>
  );
}

// Consumer wants to prevent toggling off once it's on
function NoTurnOffDemo() {
  function customReducer(state: ToggleState, action: ToggleAction): ToggleState {
    if (action.type === "TOGGLE" && state.on) {
      return state; // ignore toggle when already on
    }
    return defaultToggleReducer(state, action);
  }

  return (
    <Toggle reducer={customReducer}>
      {({ on, toggle }) => (
        <button onClick={toggle}>{on ? "ON (locked)" : "OFF — click to turn on"}</button>
      )}
    </Toggle>
  );
}
```

---

## 5. Common Mistakes & Pitfalls

**1. Context for everything (performance)**
```tsx
// WRONG — one giant context causes all consumers to re-render on any change
const AppContext = createContext({ user, theme, cart, notifications });

// CORRECT — split by update domain
const UserContext = createContext(user);
const ThemeContext = createContext(theme);
const CartContext = createContext(cart);
```

**2. HOC prop collision**
```tsx
// HOC injects `isAuthenticated` — what if the wrapped component also has that prop?
function withAuth<P>(Component: ComponentType<P & { isAuthenticated: boolean }>) {
  return (props: P) => <Component {...props} isAuthenticated={true} />;
}
// Consumer's own isAuthenticated prop is silently overwritten
```

**3. Forgetting displayName on HOCs**
```tsx
// Without displayName, React DevTools shows "Component" for all HOCs
function withFoo<P>(Wrapped: ComponentType<P>) {
  function Foo(props: P) { return <Wrapped {...props} />; }
  Foo.displayName = `withFoo(${Wrapped.displayName ?? Wrapped.name})`;
  return Foo;
}
```

**4. Render prop causes unnecessary re-renders**
```tsx
// WRONG — inline arrow function creates new reference every render
<MouseTracker render={(pos) => <Dot {...pos} />} />

// CORRECT — memoize or move outside component
const renderDot = (pos: MousePosition) => <Dot {...pos} />;
<MouseTracker render={renderDot} />
```

**5. Context value object rebuilt on every render**
```tsx
// WRONG — new object reference every render → all consumers re-render
function Provider({ children }) {
  const [user, setUser] = useState(null);
  return (
    <Ctx.Provider value={{ user, setUser }}> {/* new object every render */}
      {children}
    </Ctx.Provider>
  );
}

// CORRECT
const value = useMemo(() => ({ user, setUser }), [user]);
return <Ctx.Provider value={value}>{children}</Ctx.Provider>;
```

---

## 6. When to Use / Not Use

| Pattern | Use when | Avoid when |
|---------|----------|------------|
| Compound Components | Building UI kits with flexible layouts (Tabs, Accordion, Select) | Simple single-purpose components |
| Render Props | You need to share behavior AND data from inside the component's lifecycle | A custom hook can extract the same logic (simpler) |
| HOC | Wrapping third-party or class components, cross-cutting concerns (auth, logging) | Logic reuse within function components (use a hook) |
| Controlled Components | Live validation, cross-field dependencies, syncing with external state | Simple forms where you only need the value on submit |
| Provider Pattern | App-wide concerns (auth, theme, feature flags) | State used by just a few nearby components |
| Container/Presentational | Separating network concerns from UI for testability | Simple components with minimal data requirements |
| Slot Pattern | Flexible layouts where consumers control content | Simple, single-purpose components |
| State Reducer | Library components that need user customization of behavior | App components — over-engineered for app use |

---

## 7. Real-World Scenario

A component library's `Accordion` built with Compound Components + State Reducer, so consumers can control whether multiple panels can be open simultaneously.

```tsx
import { createContext, useContext, useReducer, ReactNode } from "react";

// ─── State ──────────────────────────────────────────────────────────────────
interface AccordionState {
  openIds: Set<string>;
  allowMultiple: boolean;
}

type AccordionAction =
  | { type: "TOGGLE"; id: string }
  | { type: "OPEN_ALL" }
  | { type: "CLOSE_ALL" };

function defaultAccordionReducer(
  state: AccordionState,
  action: AccordionAction
): AccordionState {
  switch (action.type) {
    case "TOGGLE": {
      const next = new Set(state.openIds);
      if (next.has(action.id)) {
        next.delete(action.id);
      } else {
        if (!state.allowMultiple) next.clear(); // close others if not multiple
        next.add(action.id);
      }
      return { ...state, openIds: next };
    }
    case "OPEN_ALL":
      return { ...state, openIds: new Set() }; // handled outside for real IDs
    case "CLOSE_ALL":
      return { ...state, openIds: new Set() };
    default:
      return state;
  }
}

// ─── Context ─────────────────────────────────────────────────────────────────
interface AccordionCtx {
  openIds: Set<string>;
  dispatch: React.Dispatch<AccordionAction>;
}
const AccordionContext = createContext<AccordionCtx | null>(null);
const useAccordion = () => {
  const ctx = useContext(AccordionContext);
  if (!ctx) throw new Error("Must be inside <Accordion>");
  return ctx;
};

// ─── Components ──────────────────────────────────────────────────────────────
function Accordion({
  allowMultiple = false,
  reducer = defaultAccordionReducer,
  children,
}: {
  allowMultiple?: boolean;
  reducer?: typeof defaultAccordionReducer;
  children: ReactNode;
}) {
  const [state, dispatch] = useReducer(reducer, {
    openIds: new Set<string>(),
    allowMultiple,
  });

  return (
    <AccordionContext.Provider value={{ openIds: state.openIds, dispatch }}>
      <div className="accordion">{children}</div>
    </AccordionContext.Provider>
  );
}

function AccordionItem({ id, title, children }: { id: string; title: string; children: ReactNode }) {
  const { openIds, dispatch } = useAccordion();
  const isOpen = openIds.has(id);

  return (
    <div className="accordion__item">
      <button
        aria-expanded={isOpen}
        onClick={() => dispatch({ type: "TOGGLE", id })}
      >
        {title}
      </button>
      {isOpen && <div className="accordion__panel">{children}</div>}
    </div>
  );
}

Accordion.Item = AccordionItem;

// ─── Usage ────────────────────────────────────────────────────────────────────
function FAQPage() {
  return (
    <Accordion allowMultiple>
      <Accordion.Item id="q1" title="What is React?">
        A library for building UIs.
      </Accordion.Item>
      <Accordion.Item id="q2" title="What are hooks?">
        Functions that let you use React state in function components.
      </Accordion.Item>
    </Accordion>
  );
}
```

---

## 8. Interview Questions

**Q1: What is a HOC and what are its drawbacks?**

A Higher-Order Component is a function that takes a component and returns a new component with additional behavior. Drawbacks: prop name collisions (the HOC and wrapped component may use the same prop names), wrapper hell in DevTools when multiple HOCs are composed, refs don't pass through without `forwardRef`, static methods aren't forwarded, and TypeScript inference breaks down with deep composition. Most logic-reuse use cases are better served by custom hooks.

**Q2: Controlled vs uncontrolled — when to use each?**

Controlled: React owns the value in state; you need live validation, cross-field dependencies, or to programmatically change the value. Uncontrolled: the DOM owns the value; you only need it on submit, or you're integrating with third-party libraries. Libraries like React Hook Form work uncontrolled internally for performance.

**Q3: When would you use compound components?**

When you're building a UI component with multiple related sub-components that share state — Tabs, Accordion, Select, Menu, Combobox. The parent manages state; children read it via context. This gives consumers flexibility to compose children in any order without threading state through props.

**Q4: What replaced render props?**

Custom hooks. Render props were primarily used for logic reuse before hooks existed. A `<MouseTracker render={...} />` is better expressed as a `useMousePosition()` hook. Hooks are simpler, don't add to the component tree, and compose better. Render props are still useful when you need to pass data from deep in a lifecycle (e.g., from a `ref`'s callback) to the render output.

**Q5: How does the provider pattern differ from prop drilling?**

Prop drilling passes data through intermediate components that don't use it — they just forward it. The provider pattern uses Context to make the data available directly to any consumer in the subtree without threading it through intermediaries. Providers are appropriate for app-wide concerns (auth, theme, feature flags); prop drilling is fine for 1–2 levels.

**Q6: What is the slot pattern?**

Using React's `children` prop and named props (like `header`, `footer`, `actions`) to define zones in a component's layout that consumers can fill with arbitrary content. It makes components composable — the Card doesn't need to know what goes in its header; the consumer decides. Analogous to `<slot>` in web components.

**Q7: How do you share logic between components?**

Prefer custom hooks — extract stateful logic into a `use*` function and call it from both components. For cross-cutting behavior that needs to wrap a component (auth gate, logging), use a HOC. For sharing state across the tree, use Context + Provider. For sharing UI patterns, use Compound Components or the Slot pattern.

---

## 9. Exercises

**Exercise 1: Build a Select compound component**

Build `<Select>` / `<Select.Option>` that manages the selected value internally via Context. `<Select>` shows the selected label; clicking an option selects it.

**Exercise 2: Convert a render prop to a custom hook**

Take the `MouseTracker` render prop component from this file and rewrite the logic as a `useMousePosition()` hook that returns `{ x, y }`.

**Exercise 3: Build an authenticated route HOC**

Build `withAuth(Component)` that reads from an auth context and either renders the component or redirects to `/login`.

**Exercise 4: Implement the state reducer pattern**

Extend the `Toggle` example so it limits how many times the user can toggle (e.g., max 3 toggles). The limit should be injected via a custom reducer.

---

## 10. Further Reading

- [Kent C. Dodds — Compound Components](https://kentcdodds.com/blog/compound-components-with-react-hooks)
- [Kent C. Dodds — State Reducer Pattern](https://kentcdodds.com/blog/the-state-reducer-pattern-with-react-hooks)
- [Kent C. Dodds — Inversion of Control](https://kentcdodds.com/blog/inversion-of-control)
- [React Docs — Passing Data Deeply with Context](https://react.dev/learn/passing-data-deeply-with-context)
- [Michael Jackson — Use a Render Prop!](https://cdb.reacttraining.com/use-a-render-prop-50de598f11ce)
- [Dan Abramov — Presentational and Container Components](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0)
