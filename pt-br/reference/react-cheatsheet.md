# React Cheatsheet

## Referência de Hooks

| Hook | Finalidade | Regra principal |
|------|-----------|-----------------|
| `useState` | State local do componente | O setter é assíncrono — leia no próximo render |
| `useEffect` | Efeitos colaterais, assinaturas | Retorne função de limpeza; observe deps com cuidado |
| `useLayoutEffect` | Medição do DOM antes da pintura | Bloqueia a pintura; prefira `useEffect` |
| `useRef` | Valor mutável sem re-render | Refs DOM, valor anterior, IDs de timer |
| `useMemo` | Memoiza computação custosa | Apenas quando o profiling mostra que ajuda |
| `useCallback` | Referência estável de função | Necessário quando fn é passada para filho memoizado |
| `useContext` | Lê o valor do context mais próximo | Não impede re-renders de state não relacionado |
| `useReducer` | State complexo com transições | Prefira quando o state tem múltiplos sub-valores |
| `useId` | ID único e estável | Para `aria-*` / `htmlFor` em SSR |
| `useTransition` | Marca atualização como não urgente | Mantém a UI responsiva durante renders pesados |
| `useDeferredValue` | Adia render custoso de filho | Alternativa ao debounce para state derivado |
| `useImperativeHandle` | Expõe API imperativa via ref | Raro; prefira props para controle |
| `useSyncExternalStore` | Assina stores externos | Para autores de biblioteca; não para código de app |

---

## useState

```tsx
const [count, setCount] = useState(0);
const [user, setUser] = useState<User | null>(null);

// State inicial preguiçoso (inicialização custosa roda apenas uma vez)
const [items, setItems] = useState(() => JSON.parse(localStorage.getItem('items') ?? '[]'));

// Atualização funcional — sempre use quando o novo state depende do anterior
setCount((prev) => prev + 1);

// Atualização parcial para objetos
setUser((prev) => prev ? { ...prev, name: 'Alice' } : null);
```

---

## useEffect

```tsx
// Roda a cada render (raramente o que você quer)
useEffect(() => { /* ... */ });

// Roda uma vez na montagem
useEffect(() => { /* ... */ }, []);

// Roda quando a dep muda
useEffect(() => {
  const controller = new AbortController();
  fetchData(id, controller.signal).then(setData);
  return () => controller.abort(); // limpeza
}, [id]);

// Padrão de assinatura
useEffect(() => {
  const handler = (e: Event) => { /* ... */ };
  window.addEventListener('resize', handler);
  return () => window.removeEventListener('resize', handler);
}, []);
```

**Regra de lint de deps:** inclua todo valor reativo lido dentro do effect. Se precisar escapar da regra de dep, use um `useRef` para o valor.

---

## useRef

```tsx
// Ref de DOM
const inputRef = useRef<HTMLInputElement>(null);
inputRef.current?.focus();

// Valor mutável — mudar não dispara re-render
const timerRef = useRef<NodeJS.Timeout | null>(null);
timerRef.current = setTimeout(() => {}, 1000);

// Padrão de valor anterior
function usePrevious<T>(value: T) {
  const ref = useRef<T>(undefined);
  useEffect(() => { ref.current = value; });
  return ref.current;
}
```

---

## useMemo / useCallback

```tsx
// useMemo — derivação custosa
const sortedItems = useMemo(
  () => [...items].sort((a, b) => a.name.localeCompare(b.name)),
  [items]
);

// useCallback — handler estável para filho memoizado
const handleDelete = useCallback(
  (id: string) => dispatch({ type: 'DELETE', id }),
  [dispatch] // dispatch do useReducer é estável
);

// React.memo — pula re-render quando props não mudaram
const ItemRow = React.memo(function ItemRow({ item, onDelete }: Props) {
  return <div onClick={() => onDelete(item.id)}>{item.name}</div>;
});
```

**Quando NÃO memoizar:** Componentes simples, componentes que sempre recebem novas props, quando o profiling não mostra um problema.

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
// Define context com padrão null — força consumidores a verificar o Provider
const ThemeContext = createContext<Theme | null>(null);

// Hook personalizado garante a presença do provider
export function useTheme(): Theme {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error('useTheme deve ser usado dentro de ThemeProvider');
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

**Ressalva de re-render com Context:** Todo consumidor re-renderiza quando o valor do context muda. Divida o context em providers separados se o state muda em frequências diferentes (ex.: usuário vs tema vs notificações).

---

## Compound Components

```tsx
// O pai expõe sub-componentes via propriedades estáticas
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

// Uso
<Tabs defaultTab="overview">
  <Tabs.List>
    <Tabs.Tab id="overview">Visão Geral</Tabs.Tab>
    <Tabs.Tab id="settings">Configurações</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panel id="overview"><Overview /></Tabs.Panel>
  <Tabs.Panel id="settings"><Settings /></Tabs.Panel>
</Tabs>
```

---

## Padrões de Render

```tsx
// Renderização condicional
{isLoggedIn && <Dashboard />}
{isLoggedIn ? <Dashboard /> : <Login />}

// Retorno antecipado (mais limpo para condições complexas)
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

// Children como função (mesmo padrão, API diferente)
<MouseTracker>{(pos) => <span>{pos.x}, {pos.y}</span>}</MouseTracker>
```

---

## Error Boundaries

```tsx
// Componente de classe necessário — sem equivalente em hook
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

// Uso
<ErrorBoundary fallback={<ErrorPage />}>
  <App />
</ErrorBoundary>

// Biblioteca react-error-boundary — amigável a hooks
import { ErrorBoundary, useErrorBoundary } from 'react-error-boundary';

function DataLoader() {
  const { showBoundary } = useErrorBoundary();
  useEffect(() => {
    fetchData().catch(showBoundary); // joga erros para o boundary
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
    document.getElementById('modal-root')! // renderiza fora da árvore de componentes
  );
}
```

---

## TypeScript com React

```tsx
// Props de componente
type ButtonProps = {
  variant: 'primary' | 'secondary' | 'ghost';
  size?: 'sm' | 'md' | 'lg';
  onClick?: (event: React.MouseEvent<HTMLButtonElement>) => void;
  children: React.ReactNode;
  disabled?: boolean;
} & React.ButtonHTMLAttributes<HTMLButtonElement>; // estende atributos HTML

// Componente genérico
function List<T extends { id: string }>({
  items,
  renderItem,
}: {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
}) {
  return <ul>{items.map((item) => <li key={item.id}>{renderItem(item)}</li>)}</ul>;
}

// forwardRef com TypeScript
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

// Handlers de evento
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => setValue(e.target.value);
const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => { e.preventDefault(); /* ... */ };
const handleKeyDown = (e: React.KeyboardEvent<HTMLDivElement>) => { /* ... */ };

// useRef tipado para nó DOM anulável
const ref = useRef<HTMLDivElement>(null); // null = ainda não anexado
const valueRef = useRef<string>('');      // não nulo = caixa mutável

// Tipos de children
React.ReactNode      // qualquer coisa renderizável (string, elemento, null, array)
React.ReactElement   // apenas elemento JSX
React.FC             // evite — não acrescenta muito, remove inferência de displayName
```

---

## Inputs Controlados

```tsx
// Controlado (React detém o valor)
function ControlledInput() {
  const [value, setValue] = useState('');
  return (
    <input
      value={value}
      onChange={(e) => setValue(e.target.value)}
    />
  );
}

// Não controlado (DOM detém o valor, acessado via ref)
function UncontrolledForm() {
  const nameRef = useRef<HTMLInputElement>(null);
  const handleSubmit = () => console.log(nameRef.current?.value);
  return <input ref={nameRef} defaultValue="" />;
}

// Inputs de arquivo sempre são não controlados
const fileRef = useRef<HTMLInputElement>(null);
const file = fileRef.current?.files?.[0];
```

---

## Padrões de Performance

```tsx
// useTransition — marca atualização de state como baixa prioridade
const [isPending, startTransition] = useTransition();

function handleSearch(query: string) {
  setInputValue(query); // urgente: atualiza o input imediatamente
  startTransition(() => {
    setSearchQuery(query); // não urgente: pode ser interrompido
  });
}

// useDeferredValue — adia render custoso de filho
const deferredQuery = useDeferredValue(searchQuery);
// deferredQuery fica atrás de searchQuery; passe para filho custoso
<SearchResults query={deferredQuery} />

// Lazy loading com Suspense
const HeavyComponent = React.lazy(() => import('./HeavyComponent'));

<Suspense fallback={<Skeleton />}>
  <HeavyComponent />
</Suspense>

// Code splitting por rota
const SettingsPage = React.lazy(() => import('./pages/Settings'));
```

---

## Padrões Importantes para Lembrar

- **Keys devem ser estáveis e únicas** — use IDs, não índices de array (índices quebram animações e state)
- **State deve ser mínimo** — derive tudo que puder do state existente
- **Suba o state** quando irmãos precisam compartilhá-lo; mova para baixo quando apenas um componente usa
- **Evite state derivado no state** — compute a partir de props/state no render
- **Effects são para sincronização** — se estiver usando um effect para atualizar state quando props mudam, considere derivar o valor
- **Batching:** React 18 agrupa todas as atualizações de state inclusive as em timeouts e promises
