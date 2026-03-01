# ARIA Roles

## Visão Geral

Accessible Rich Internet Applications (ARIA) é um conjunto de atributos HTML que define formas de tornar o conteúdo web mais acessível para tecnologias assistivas. O ARIA adiciona roles, properties e states a elementos que, de outra forma, não teriam significado semântico — ou sobrescreve semânticas nativas quando necessário.

A primeira regra do ARIA é: **não use ARIA**. Se um elemento ou atributo HTML nativo atende ao seu caso de uso, use-o. Um `<button>` é sempre melhor do que um `<div role="button">`. Mas componentes interativos customizados — comboboxes, disclosure widgets, carousels, data grids — frequentemente não têm equivalente HTML nativo, e é aí que o ARIA se torna essencial.

Este capítulo cobre quando recorrer ao ARIA, como roles/properties/states funcionam, live regions para conteúdo dinâmico e a implementação completa do padrão combobox.

## Pré-requisitos

- Sólido entendimento de semantic HTML (veja `semantic-html.md`)
- JavaScript / TypeScript básico
- Familiaridade com o painel de acessibilidade das DevTools do browser

## Conceitos Fundamentais

### Os Três Atributos ARIA

**`role`** — o que o elemento é:
```html
<div role="button">Click me</div>
<div role="dialog">...</div>
<div role="alert">Error: file not found</div>
```

**Propriedades `aria-*`** — características persistentes:
```html
<input aria-label="Search" aria-required="true">
<nav aria-labelledby="nav-title">
<div role="progressbar" aria-valuemin="0" aria-valuemax="100" aria-valuenow="45">
```

**States `aria-*`** — condições dinâmicas que mudam com a interação do usuário:
```html
<button aria-expanded="false">Menu</button>
<input aria-invalid="true">
<li role="option" aria-selected="true">
```

### A Hierarquia ARIA

As roles ARIA são organizadas em uma taxonomia:

```
abstract roles (nunca use diretamente)
└── widget roles (controles de interface interativos)
    ├── button, checkbox, combobox, dialog, grid, link, listbox ...
└── document structure roles (não interativos)
    ├── article, figure, heading, list, table ...
└── landmark roles (regiões da página)
    ├── banner, complementary, contentinfo, form, main, navigation, region, search
└── live region roles
    ├── alert, log, marquee, status, timer
└── window roles
    ├── alertdialog, dialog
```

### Quando Usar ARIA vs HTML Nativo

| Caso de uso | HTML nativo | Alternativa ARIA |
|----------|-------------|-----------------|
| Button | `<button>` | `<div role="button" tabindex="0">` |
| Checkbox | `<input type="checkbox">` | `<div role="checkbox" aria-checked="false">` |
| Link | `<a href="...">` | `<div role="link" tabindex="0">` |
| Heading | `<h2>` | `<div role="heading" aria-level="2">` |
| List | `<ul>/<li>` | `<div role="list"><div role="listitem">` |
| Dialog | `<dialog>` | `<div role="dialog" aria-modal="true">` |

Prefira a coluna de HTML nativo em todos os casos. Use ARIA quando:
1. Você está construindo um widget customizado sem equivalente HTML (combobox, tree, data grid)
2. Você precisa sobrescrever semânticas nativas (por exemplo, um `<div>` em um framework que renderiza um button)
3. Você precisa comunicar mudanças de estado dinâmicas

### Propriedades ARIA Comuns

**Nomeação e rotulagem:**
```html
<!-- aria-label: label de string direta -->
<button aria-label="Close dialog">
  <svg aria-hidden="true">...</svg>
</button>

<!-- aria-labelledby: referencia o texto de outro elemento -->
<h2 id="dialog-title">Delete file</h2>
<div role="dialog" aria-labelledby="dialog-title">...</div>

<!-- aria-describedby: descrição complementar -->
<input id="pw" type="password" aria-describedby="pw-hint">
<p id="pw-hint">Must be 8+ characters with at least one number.</p>
```

**Relacionamentos:**
```html
<!-- aria-controls: este elemento controla outro -->
<button aria-controls="panel1" aria-expanded="false">Section 1</button>
<div id="panel1" hidden>Content</div>

<!-- aria-owns: indica propriedade quando a ordem do DOM não reflete -->
<ul role="listbox" aria-owns="extra-option">
  <li role="option">Option 1</li>
</ul>
<li id="extra-option" role="option">Option 2</li>
```

**States:**
```html
<button aria-pressed="true">Bold</button>      <!-- toggle button -->
<button aria-expanded="false">Menu</button>    <!-- disclosure -->
<li role="option" aria-selected="true">       <!-- seleção -->
<input aria-invalid="true">                    <!-- erro de validação -->
<div aria-busy="true">Loading...</div>         <!-- carregamento assíncrono -->
<button aria-disabled="true">Submit</button>   <!-- desabilitado (ainda focável) -->
```

### Live Regions

Live regions anunciam mudanças de conteúdo para screen readers sem mover o focus. É assim que você notifica usuários de atualizações dinâmicas — toasts, mensagens de status, contagem de resultados de busca, resumos de erro.

| Role / Atributo | Polidez | Caso de uso |
|-----------------|------------|----------|
| `role="alert"` | Assertivo | Erros, status crítico (interrompe a leitura atual) |
| `role="status"` | Educado | Mensagens de sucesso, atualizações não críticas |
| `aria-live="polite"` | Educado | Live region customizada (termina a frase atual primeiro) |
| `aria-live="assertive"` | Assertivo | Live region customizada (interrompe imediatamente) |
| `role="log"` | Educado | Mensagens de chat, logs de auditoria (conteúdo aditivo) |
| `role="timer"` | Desligado | Contagens regressivas (AT não lê automaticamente) |

```html
<!-- Notificação toast -->
<div role="status" aria-live="polite" aria-atomic="true">
  <!-- Injetado dinamicamente: "Arquivo salvo com sucesso" -->
</div>

<!-- Resumo de erros -->
<div role="alert">
  <!-- Injetado: "3 erros encontrados. Corrija-os antes de continuar." -->
</div>
```

Atributo importante: `aria-atomic="true"` — anuncia toda a região quando qualquer parte muda (em vez de apenas as partes modificadas).

### O Padrão Combobox

Um combobox é uma combinação de um input de texto e um listbox popup. É um dos padrões ARIA mais comuns, usado para busca com autocomplete, selects filtrados e inputs de múltipla seleção.

Implementação completa das WAI-ARIA Authoring Practices:

```html
<div class="combobox-wrapper">
  <label for="country-input" id="country-label">Country</label>
  <div
    role="combobox"
    aria-expanded="false"
    aria-haspopup="listbox"
    aria-owns="country-listbox"
  >
    <input
      id="country-input"
      type="text"
      aria-autocomplete="list"
      aria-controls="country-listbox"
      aria-activedescendant=""
      autocomplete="off"
    >
  </div>
  <ul
    id="country-listbox"
    role="listbox"
    aria-labelledby="country-label"
    hidden
  >
    <li id="opt-br" role="option" aria-selected="false">Brazil</li>
    <li id="opt-us" role="option" aria-selected="false">United States</li>
    <li id="opt-de" role="option" aria-selected="false">Germany</li>
  </ul>
</div>
```

## Exemplos Práticos

### Construindo um Toggle Button Acessível

```tsx
import { useState } from 'react';

interface ToggleButtonProps {
  children: React.ReactNode;
  onToggle?: (pressed: boolean) => void;
}

function ToggleButton({ children, onToggle }: ToggleButtonProps) {
  const [pressed, setPressed] = useState(false);

  function handleClick() {
    const next = !pressed;
    setPressed(next);
    onToggle?.(next);
  }

  return (
    <button
      type="button"
      aria-pressed={pressed}
      onClick={handleClick}
      className={pressed ? 'btn-active' : 'btn'}
    >
      {children}
    </button>
  );
}

// Uso: toggle de formatação Negrito
<ToggleButton onToggle={(on) => applyFormatting('bold', on)}>
  Bold
</ToggleButton>
```

### Sistema de Notificações Toast

```tsx
import { useState, useRef, useEffect } from 'react';

interface Toast {
  id: string;
  message: string;
  type: 'success' | 'error';
}

function ToastContainer() {
  const [toasts, setToasts] = useState<Toast[]>([]);
  // Ref da live region — deve existir no DOM antes de o conteúdo ser injetado
  const liveRef = useRef<HTMLDivElement>(null);

  function addToast(message: string, type: Toast['type']) {
    const id = crypto.randomUUID();
    setToasts((prev) => [...prev, { id, message, type }]);
    // Auto-dispensar após 5s
    setTimeout(() => {
      setToasts((prev) => prev.filter((t) => t.id !== id));
    }, 5000);
  }

  return (
    <>
      {/* Live region — sempre renderizada, mudanças de conteúdo são anunciadas */}
      <div
        ref={liveRef}
        role={toasts.some((t) => t.type === 'error') ? 'alert' : 'status'}
        aria-live={toasts.some((t) => t.type === 'error') ? 'assertive' : 'polite'}
        aria-atomic="true"
        className="sr-only"
      >
        {toasts[toasts.length - 1]?.message}
      </div>

      {/* Exibição visual dos toasts */}
      <div className="toast-stack" aria-hidden="true">
        {toasts.map((toast) => (
          <div key={toast.id} className={`toast toast-${toast.type}`}>
            {toast.message}
          </div>
        ))}
      </div>
    </>
  );
}
```

### Disclosure Widget Acessível (Alternativa ao Details/Summary)

```tsx
function Disclosure({ summary, children }: { summary: string; children: React.ReactNode }) {
  const [open, setOpen] = useState(false);
  const contentId = useId();

  return (
    <div>
      <button
        type="button"
        aria-expanded={open}
        aria-controls={contentId}
        onClick={() => setOpen((o) => !o)}
      >
        <span aria-hidden="true">{open ? '▾' : '▸'}</span>
        {summary}
      </button>
      <div id={contentId} hidden={!open}>
        {children}
      </div>
    </div>
  );
}
```

### Implementação Completa de Combobox (TypeScript)

```tsx
import { useState, useId, useRef } from 'react';

interface ComboboxProps {
  label: string;
  options: string[];
  onSelect: (value: string) => void;
}

function Combobox({ label, options, onSelect }: ComboboxProps) {
  const [input, setInput] = useState('');
  const [open, setOpen] = useState(false);
  const [activeIndex, setActiveIndex] = useState(-1);
  const listboxId = useId();
  const inputRef = useRef<HTMLInputElement>(null);

  const filtered = options.filter((o) =>
    o.toLowerCase().includes(input.toLowerCase())
  );

  function handleKeyDown(e: React.KeyboardEvent) {
    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault();
        setOpen(true);
        setActiveIndex((i) => Math.min(i + 1, filtered.length - 1));
        break;
      case 'ArrowUp':
        e.preventDefault();
        setActiveIndex((i) => Math.max(i - 1, 0));
        break;
      case 'Enter':
        if (activeIndex >= 0 && filtered[activeIndex]) {
          selectOption(filtered[activeIndex]);
        }
        break;
      case 'Escape':
        setOpen(false);
        setActiveIndex(-1);
        inputRef.current?.focus();
        break;
    }
  }

  function selectOption(value: string) {
    setInput(value);
    setOpen(false);
    setActiveIndex(-1);
    onSelect(value);
  }

  const activeId = activeIndex >= 0 ? `${listboxId}-opt-${activeIndex}` : undefined;

  return (
    <div>
      <label htmlFor={`${listboxId}-input`}>{label}</label>
      <div
        role="combobox"
        aria-expanded={open}
        aria-haspopup="listbox"
        aria-owns={listboxId}
      >
        <input
          id={`${listboxId}-input`}
          ref={inputRef}
          type="text"
          value={input}
          aria-autocomplete="list"
          aria-controls={listboxId}
          aria-activedescendant={activeId}
          autoComplete="off"
          onChange={(e) => {
            setInput(e.target.value);
            setOpen(true);
            setActiveIndex(-1);
          }}
          onKeyDown={handleKeyDown}
          onFocus={() => setOpen(true)}
          onBlur={() => setTimeout(() => setOpen(false), 150)}
        />
      </div>
      {open && filtered.length > 0 && (
        <ul id={listboxId} role="listbox" aria-label={label}>
          {filtered.map((option, i) => (
            <li
              key={option}
              id={`${listboxId}-opt-${i}`}
              role="option"
              aria-selected={i === activeIndex}
              onMouseDown={() => selectOption(option)}
            >
              {option}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

## Padrões e Boas Práticas

### Precedência de Nomeação ARIA

Quando a AT calcula um nome para um elemento, usa esta ordem de prioridade:
1. `aria-labelledby` (referencia texto visível — mais autoritativo)
2. `aria-label` (string explícita)
3. Associação de label nativa (`<label for="...">`)
4. Conteúdo do elemento (para buttons, links)
5. Atributo `title` (último recurso, não confiável)

### Ocultando Conteúdo Corretamente

```html
<!-- Visível para todos -->
<p>Normal content</p>

<!-- Oculto de todos (display:none / atributo hidden) -->
<div hidden>Not visible, not in AT tree</div>

<!-- Visível na tela, oculto da AT -->
<div aria-hidden="true">Decorative icon or redundant text</div>

<!-- Oculto na tela, visível para a AT (classe sr-only) -->
<span class="sr-only">Additional context for screen readers</span>
```

### Melhoria Progressiva com ARIA

Adicione ARIA para enriquecer, não para substituir HTML funcional:

```tsx
// Fase 1: funcional sem ARIA
<select name="country">
  <option>Brazil</option>
  <option>United States</option>
</select>

// Fase 2: combobox customizado com ARIA completo quando você precisa da UX
// (construa customizado apenas quando <select> genuinamente não atender seus requisitos de design)
<Combobox label="Country" options={countries} onSelect={setCountry} />
```

### Boas Práticas para Live Regions

- **Renderize live regions no carregamento da página** — injete conteúdo dinamicamente depois. Os browsers registram live regions na renderização; adicionar `role="alert"` a um elemento existente com conteúdo nem sempre funciona.
- **Evite excesso de anúncios** — não transforme toda mudança de estado em uma live region
- **Use `aria-atomic="true"` para mensagens autocontidas** (toast, status)
- **Use `aria-atomic="false"` para logs** onde apenas novas entradas importam

## Anti-Padrões a Evitar

**Roles ARIA redundantes em elementos semânticos**
```html
<!-- Ruim: <button> já tem role="button" -->
<button role="button">Click</button>

<!-- Ruim: <nav> já tem role="navigation" -->
<nav role="navigation">...</nav>
```

**Suporte de teclado ausente em widgets ARIA**
```html
<!-- Ruim: role="button" sem tabindex ou handler de teclado -->
<div role="button" onclick="doThing()">Click me</div>

<!-- Bom: tabindex torna focável, keydown trata Enter/Space -->
<div
  role="button"
  tabindex="0"
  onclick="doThing()"
  onkeydown="if(e.key==='Enter'||e.key===' ')doThing()"
>
  Click me
</div>
```

**Usar `aria-hidden="true"` em elementos focáveis**
```html
<!-- Ruim: elemento oculto da AT mas ainda recebe focus -->
<a href="/home" aria-hidden="true">Home</a>

<!-- Bom: também remova da tab order -->
<a href="/home" aria-hidden="true" tabindex="-1">Home</a>
<!-- Ou melhor: simplesmente oculte corretamente -->
<a href="/home" hidden>Home</a>
```

**`aria-label` duplicando texto visível**
```html
<!-- Ruim: mesmo texto, redundante -->
<button aria-label="Submit">Submit</button>

<!-- Bom: aria-label apenas quando difere do texto visível -->
<button aria-label="Submit registration form">Submit</button>
```

**Anunciar em cada tecla pressionada**
```tsx
// Ruim: live region dispara a cada caractere digitado
<div role="status" aria-live="polite">{searchResultCount} results</div>

// Bom: aplique debounce na atualização
const [count, setCount] = useState(0);
const debouncedCount = useDebounce(searchResultCount, 500);
useEffect(() => setCount(debouncedCount), [debouncedCount]);
<div role="status" aria-live="polite">{count} results</div>
```

## Depuração e Solução de Problemas

### Inspecionando Propriedades ARIA Computadas

```bash
# Chrome DevTools → Elements → selecione o elemento → aba Accessibility
# Exibe: role, nome, descrição, propriedades, estado e árvore completa

# Ou via Playwright nos testes:
const snapshot = await page.accessibility.snapshot({ root: await page.$('#dialog') });
console.log(JSON.stringify(snapshot, null, 2));
```

### Detectando Problemas ARIA com axe

```bash
npx axe https://localhost:3000 \
  --rules aria-allowed-attr,aria-required-attr,aria-valid-attr-value,\
          aria-roles,aria-hidden-focus,aria-required-children
```

### Testando Live Regions

Live regions são difíceis de testar programaticamente. Combine:
1. **axe-core** — verifica se os valores do atributo `aria-live` são válidos
2. **jest-axe** — detecta `aria-atomic` ausente e roles inválidos
3. **Teste manual com screen reader** — acione a mudança dinâmica e ouça o anúncio

```typescript
// Verificar se a live region existe e tem os atributos corretos
test('status region is configured correctly', async ({ page }) => {
  await page.goto('/dashboard');
  const status = page.locator('[role="status"]');
  await expect(status).toHaveAttribute('aria-live', 'polite');
  await expect(status).toHaveAttribute('aria-atomic', 'true');
});
```

## Cenários do Mundo Real

### Cenário 1: Busca com Contagem de Resultados em Tempo Real

```tsx
function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<Result[]>([]);
  const [searching, setSearching] = useState(false);
  const [announcement, setAnnouncement] = useState('');

  async function handleSearch(q: string) {
    setQuery(q);
    if (!q) { setResults([]); setAnnouncement(''); return; }
    setSearching(true);
    setAnnouncement('Buscando...');
    const data = await search(q);
    setResults(data);
    setSearching(false);
    setAnnouncement(
      data.length === 0
        ? 'Nenhum resultado encontrado.'
        : `${data.length} resultado${data.length !== 1 ? 's' : ''} encontrado${data.length !== 1 ? 's' : ''}.`
    );
  }

  return (
    <div>
      {/* Live region sempre presente */}
      <div role="status" aria-live="polite" aria-atomic="true" className="sr-only">
        {announcement}
      </div>

      <label htmlFor="search">Search</label>
      <input
        id="search"
        type="search"
        value={query}
        onChange={(e) => handleSearch(e.target.value)}
        aria-busy={searching}
      />

      {results.length > 0 && (
        <ul aria-label="Search results">
          {results.map((r) => (
            <li key={r.id}><a href={r.url}>{r.title}</a></li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

### Cenário 2: Modal Dialog Acessível

```tsx
function Modal({ title, onClose, children }: {
  title: string;
  onClose: () => void;
  children: React.ReactNode;
}) {
  const titleId = useId();
  const descId = useId();

  // Gerenciamento de focus tratado em keyboard-navigation.md
  // Aqui o foco é nos atributos ARIA
  return (
    <div
      role="dialog"
      aria-modal="true"
      aria-labelledby={titleId}
      aria-describedby={descId}
    >
      <h2 id={titleId}>{title}</h2>
      <div id={descId}>{children}</div>
      <button type="button" onClick={onClose} aria-label={`Close ${title} dialog`}>
        <svg aria-hidden="true" focusable="false">...</svg>
      </button>
    </div>
  );
}
```

## Leitura Complementar

- [WAI-ARIA Specification 1.2](https://www.w3.org/TR/wai-aria-1.2/) — taxonomia completa de roles
- [WAI-ARIA Authoring Practices Guide](https://www.w3.org/WAI/ARIA/apg/) — padrões de design com exemplos de código
- [ARIA in HTML (W3C)](https://www.w3.org/TR/html-aria/) — uso permitido de ARIA por elemento HTML
- [Deque: ARIA Role Definitions](https://dequeuniversity.com/library/aria/) — referência prática
- [Inclusive Components by Heydon Pickering](https://inclusive-components.design/) — análises aprofundadas de padrões comuns

## Resumo

ARIA adiciona significado a componentes customizados que o HTML não consegue expressar nativamente. Os três atributos são `role` (o que o elemento é), properties (características persistentes como `aria-label`) e states (condições dinâmicas como `aria-expanded`).

As regras de ouro:
- Use HTML nativo primeiro. ARIA é um complemento, não uma substituição.
- Todo widget ARIA precisa de suporte completo a teclado — ARIA comunica, mas não implementa comportamento.
- Live regions (`role="status"`, `role="alert"`) devem estar presentes no DOM antes de você injetar conteúdo.
- Não use `aria-hidden="true"` em elementos focáveis.
- `aria-labelledby` tem prioridade sobre `aria-label`, que tem prioridade sobre o conteúdo.
- Teste com screen readers reais — ferramentas automatizadas não detectam muitos erros de ARIA.
