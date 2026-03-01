# Otimização de JavaScript

## Visão Geral

A performance de JavaScript tem dois domínios: a rapidez com que o navegador faz o download e analisa seus bundles (performance de carregamento) e a rapidez com que o código executa em tempo de execução (performance de runtime). Este capítulo cobre os dois: otimização de bundle com ferramentas de build modernas, tree-shaking, code splitting, lazy loading e padrões de runtime que mantêm a thread principal responsiva.

---

## Pré-requisitos

- Fundamentos de JavaScript/TypeScript
- Entendimento básico de bundlers (Vite, webpack)
- Track 02 — Frontend: fundamentos de React

---

## Exemplos Práticos

### Análise de bundle — encontrando o que pesa

```bash
# Visualização de bundle com Vite
npm install --save-dev rollup-plugin-visualizer

# vite.config.ts
import { visualizer } from 'rollup-plugin-visualizer';
export default {
  plugins: [
    visualizer({ open: true, gzipSize: true, filename: 'bundle-stats.html' }),
  ],
};

# Fazer o build e abrir o treemap
npm run build
```

O treemap mostra quais módulos mais contribuem para o tamanho do bundle. Suspeitos comuns: `moment.js` (substitua por `date-fns`), `lodash` (use imports individuais ou métodos nativos), bibliotecas de ícones grandes (use sprites SVG ou lazy load).

### Tree-shaking — eliminando código morto

```typescript
// Ruim — importa o lodash inteiro (530kB!)
import _ from 'lodash';
const unique = _.uniq(arr);

// Bom — importa apenas o necessário (cherry-pick)
import { uniq } from 'lodash-es'; // versão ES module é tree-shakeable

// Melhor — usa equivalente nativo (zero custo no bundle)
const unique = [...new Set(arr)];
```

```typescript
// Ruim — barrel files frequentemente quebram tree-shaking
// src/utils/index.ts
export * from './format.js';
export * from './validate.js';
export * from './crypto.js'; // 50kB — agora importado por todos

// Bom — importa diretamente
import { formatDate } from './utils/format.js'; // apenas format.js é incluído no bundle
```

### Code splitting — dividindo por rota

```typescript
// React Router com lazy loading
import { lazy, Suspense } from 'react';
import { Routes, Route } from 'react-router-dom';

// Cada página é um chunk separado — carregado apenas quando navegado
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

### Dynamic imports — lazy loading de componentes pesados

```typescript
// Carrega uma biblioteca de gráficos pesada apenas quando o usuário navega para a página de gráficos
import { useState, useEffect } from 'react';

function ChartPage() {
  const [ChartComponent, setChartComponent] = useState<React.ComponentType | null>(null);

  useEffect(() => {
    // Carrega o chunk apenas quando este componente é montado
    import('./components/HeavyChart').then((module) => {
      setChartComponent(() => module.HeavyChart);
    });
  }, []);

  if (!ChartComponent) return <Skeleton />;
  return <ChartComponent />;
}
```

```typescript
// Ou com React.lazy — sintaxe mais limpa
const HeavyChart = lazy(() => import('./components/HeavyChart'));

// Carregado apenas quando <HeavyChart /> é renderizado
function ChartPage() {
  const [show, setShow] = useState(false);
  return (
    <>
      <button onClick={() => setShow(true)}>Carregar Gráfico</button>
      {show && (
        <Suspense fallback={<Skeleton />}>
          <HeavyChart />
        </Suspense>
      )}
    </>
  );
}
```

### Pré-carregando chunks críticos

```typescript
// Pré-carrega um chunk que será necessário em breve (ex: no hover)
function NavLink({ href, children }: { href: string; children: React.ReactNode }) {
  const handleMouseEnter = () => {
    // Usuário está prestes a clicar — começa a carregar o chunk
    import('./pages/Dashboard'); // dispara o download sem executar
  };

  return (
    <a href={href} onMouseEnter={handleMouseEnter}>
      {children}
    </a>
  );
}
```

### Evitando layout thrashing

Layout thrashing ocorre quando você lê propriedades de layout (width, height, offsetTop) e escreve propriedades no DOM de forma intercalada:

```typescript
// Ruim — causa layout thrashing
function expandCards(cards: HTMLElement[]) {
  cards.forEach((card) => {
    const height = card.offsetHeight; // FORÇAR LAYOUT (leitura)
    card.style.height = `${height + 50}px`; // escrita
    // Próxima iteração: a leitura força recalculo por causa da escrita anterior
  });
}

// Bom — agrupa leituras, depois agrupa escritas
function expandCardsOptimized(cards: HTMLElement[]) {
  // Fase de leitura (todas as leituras de layout em uma passagem)
  const heights = cards.map((card) => card.offsetHeight);

  // Fase de escrita (todas as escritas no DOM após todas as leituras)
  cards.forEach((card, i) => {
    card.style.height = `${heights[i] + 50}px`;
  });
}
```

### Debouncing e throttling de event handlers

```typescript
// Campo de busca — debounce para evitar disparo a cada tecla
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

// Handler de scroll — throttle a 60fps
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

### Movendo computação pesada para um Web Worker

```typescript
// src/workers/filter.worker.ts
self.onmessage = (event: MessageEvent<{ products: Product[]; query: string }>) => {
  const { products, query } = event.data;
  const result = products.filter((p) => p.name.toLowerCase().includes(query.toLowerCase()));
  self.postMessage(result);
};

// Uso no componente
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

### Padrões de performance no React

```typescript
// Memoize computações custosas
const sortedProducts = useMemo(
  () => [...products].sort((a, b) => a.price - b.price),
  [products] // recalcula apenas quando products muda
);

// Memoize componentes que recebem as mesmas props
const ProductCard = React.memo(function ProductCard({ product }: { product: Product }) {
  return <div>{product.name}</div>;
});

// Referências estáveis de callbacks
const handleClick = useCallback((id: string) => {
  addToCart(id);
}, [addToCart]); // addToCart também deve ser estável
```

---

## Padrões e Boas Práticas

- **Analise seu bundle antes de otimizar** — o visualizador mostra onde focar o esforço
- **Use APIs nativas no lugar de bibliotecas** sempre que possível — `Array.filter`, `Set`, `fetch`, `Intl`
- **Faça code splitting por rota** — usuários só baixam código das páginas que visitam
- **Faça lazy load de componentes pesados** — visualizadores de PDF, bibliotecas de gráficos, editores rich text
- **Use `passive: true` em listeners de scroll/touch** — avisa o navegador para não esperar seu handler antes de rolar
- **Meça com o painel Performance do DevTools** — identifica funções específicas causando tarefas longas

---

## Anti-Padrões a Evitar

- Importar bibliotecas inteiras quando você usa uma função (`moment`, `lodash`, conjuntos completos de ícones)
- Barrel files que re-exportam tudo — frequentemente quebram tree-shaking
- Computação pesada na thread principal — use Web Workers ou agende com `scheduler.postTask`
- Leituras síncronas de DOM dentro de loops de animação — use `requestAnimationFrame` e agrupe leituras/escritas

---

## Debugging e Resolução de Problemas

**"Meu bundle tem 2MB e não sei por quê"**
Execute o visualizador. Procure `node_modules` inesperadamente grandes. Surpresas comuns: `moment` (240kB), `lodash` (530kB), `date-fns` carregado por completo em vez de cherry-picked, uma biblioteca de ícones inteira carregada para 3 ícones.

**"A página trava durante o scroll"**
Abra o DevTools → Performance → grave enquanto faz scroll. Procure frames longos (vermelho na timeline). Event handlers demorados ou layout thrashing são os culpados habituais.

**"O React está re-renderizando com muita frequência"**
Use o React DevTools Profiler para gravar interações e ver quais componentes re-renderizam e por quê. Adicione `React.memo`, `useMemo` e `useCallback` estrategicamente após profiling — não especulativamente.

---

## Leituras Complementares

- [web.dev: Optimize JavaScript execution](https://web.dev/optimize-javascript-execution/)
- [Chrome DevTools: Analyze runtime performance](https://developer.chrome.com/docs/devtools/performance/)
- [Vite: build optimization](https://vitejs.dev/guide/features.html#build-optimizations)
- Track 10: [Ferramentas de Profiling](profiling-tools.md)
- Track 10: [Métricas de Performance Web](web-performance-metrics.md)

---

## Resumo

A otimização de JavaScript opera em dois níveis: tamanho do bundle (quanto código o navegador baixa) e performance de runtime (com que eficiência esse código é executado). A análise do bundle identifica dependências excessivas; tree-shaking e code splitting as eliminam ou adiam. A otimização de runtime foca em manter a thread principal desbloqueada: aplique debounce em event handlers, agrupe leituras e escritas no DOM, mova computação pesada para Web Workers e use `requestAnimationFrame` para atualizações visuais. Sempre faça profiling antes de otimizar — o gargalo raramente está onde você espera.
