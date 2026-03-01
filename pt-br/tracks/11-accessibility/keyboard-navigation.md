# Keyboard Navigation

## Visão Geral

Todo elemento interativo em uma página web deve ser alcançável e operável apenas com teclado. Isso é WCAG 2.1.1 (Nível A) — não é opcional. A acessibilidade por teclado beneficia diretamente usuários que não conseguem usar um dispositivo apontador: pessoas com deficiências motoras, usuários avançados que preferem teclado, softwares de controle por voz (o Dragon NaturallySpeaking navega por elementos focáveis) e usuários de screen reader (que navegam por Tab, setas e atalhos).

Construir uma boa keyboard navigation exige entender a tab order nativa do browser, gerenciar o focus programaticamente quando a interface muda, implementar focus traps em modal dialogs, fornecer skip links para contornar conteúdo repetitivo e construir widgets customizados com roving tabindex.

## Pré-requisitos

- Semantic HTML (veja `semantic-html.md`)
- Conceitos básicos de ARIA (veja `aria-roles.md`)
- JavaScript / TypeScript (tratamento de eventos do DOM)

## Conceitos Fundamentais

### A Tab Order Natural

Por padrão, os browsers criam uma tab order a partir de elementos nativamente focáveis, na ordem do código-fonte do DOM:

- `<a href="...">` — links
- `<button>` — buttons
- `<input>`, `<select>`, `<textarea>` — controles de formulário
- `<details>` — disclosure
- Elementos com `tabindex="0"` — adicionados explicitamente à tab order

`Tab` avança; `Shift+Tab` recua. A posição visual na tela não determina a tab order — a ordem do DOM determina. Reordenar com CSS (`order` do flexbox, `grid-area`, `position: absolute`) sem alterar a ordem do DOM quebra a tab order.

### Atributo `tabindex`

| Valor | Comportamento |
|-------|----------|
| `tabindex="0"` | Adiciona o elemento à tab order natural na sua posição do DOM |
| `tabindex="-1"` | Focável programaticamente (`element.focus()`), excluído do Tab |
| `tabindex="1"` ou maior | tabindex positivo — cria uma ordem de focus separada que roda antes do tabindex="0". **Nunca use.** |

```html
<!-- Widget interativo customizado adicionado à tab order -->
<div role="button" tabindex="0">Click me</div>

<!-- Painel alcançável por script, mas não pelo Tab -->
<div id="modal-content" tabindex="-1">...</div>
```

### Gerenciamento de Focus

Quando a página muda significativamente — abrindo um modal, navegando para uma nova página em um SPA, exibindo um resumo de erros — você deve mover o focus para refletir o novo contexto.

```typescript
// Mover o focus para um elemento programaticamente
const element = document.getElementById('dialog-title');
element?.focus();

// Mover o focus para um painel revelado por ação do usuário
function showModal() {
  dialog.removeAttribute('hidden');
  dialogHeading.focus(); // foca o heading ou o container do dialog
}

// Após fechar um modal, retorna o focus para o elemento que o abriu
let previousFocus: HTMLElement | null = null;

function openModal(triggerButton: HTMLElement) {
  previousFocus = triggerButton;
  modal.show();
  modalTitle.focus();
}

function closeModal() {
  modal.hide();
  previousFocus?.focus();
  previousFocus = null;
}
```

### Focus Traps em Modais

Quando um modal dialog está aberto, o keyboard focus deve ficar preso dentro dele. Usuários que pressionam Tab devem percorrer apenas os elementos focáveis do modal — sem escapar para a página por trás.

```typescript
function getFocusableElements(container: HTMLElement): HTMLElement[] {
  const selectors = [
    'a[href]',
    'button:not([disabled])',
    'input:not([disabled])',
    'select:not([disabled])',
    'textarea:not([disabled])',
    '[tabindex]:not([tabindex="-1"])',
  ].join(',');
  return Array.from(container.querySelectorAll<HTMLElement>(selectors));
}

function trapFocus(container: HTMLElement) {
  function handleKeydown(e: KeyboardEvent) {
    if (e.key !== 'Tab') return;

    const focusable = getFocusableElements(container);
    if (focusable.length === 0) return;

    const first = focusable[0];
    const last = focusable[focusable.length - 1];

    if (e.shiftKey) {
      // Shift+Tab: se estiver no primeiro elemento, vai para o último
      if (document.activeElement === first) {
        e.preventDefault();
        last.focus();
      }
    } else {
      // Tab: se estiver no último elemento, vai para o primeiro
      if (document.activeElement === last) {
        e.preventDefault();
        first.focus();
      }
    }
  }

  container.addEventListener('keydown', handleKeydown);
  return () => container.removeEventListener('keydown', handleKeydown);
}
```

### Skip Links

Skip links permitem que usuários de teclado pulem blocos de navegação repetidos e vão diretamente para o conteúdo principal. Exigido pela WCAG 2.4.1 (Nível A).

```html
<!-- Primeiro elemento focável em todas as páginas -->
<a href="#main-content" class="skip-link">Ir para o conteúdo principal</a>

<header>
  <!-- 50+ links de navegação -->
</header>

<main id="main-content" tabindex="-1">
  <!-- tabindex="-1" para que possamos focar <main> programaticamente,
       mesmo não sendo nativamente focável -->
  <h1>Page title</h1>
</main>
```

```css
.skip-link {
  position: absolute;
  top: -999px;
  left: 0;
  padding: 8px 16px;
  background: #000;
  color: #fff;
  text-decoration: none;
  z-index: 9999;
  border-radius: 0 0 4px 0;
}

.skip-link:focus {
  top: 0;
}
```

Múltiplos skip links para páginas complexas:
```html
<a href="#main-content">Ir para o conteúdo</a>
<a href="#site-search">Ir para a busca</a>
<a href="#main-nav">Ir para a navegação</a>
```

### Roving Tabindex

Para widgets compostos (toolbars, tab lists, radio groups, menus, tree views), apenas um item deve estar na tab order de cada vez. O usuário navega entre os itens com as teclas de seta. Isso é chamado de "roving tabindex".

Padrão:
1. Defina `tabindex="0"` no item atualmente ativo
2. Defina `tabindex="-1"` em todos os outros itens
3. Ao pressionar uma tecla de seta, mova o focus e troque os valores de tabindex

```typescript
class RovingTabindex {
  private items: HTMLElement[];
  private currentIndex: number;

  constructor(private container: HTMLElement) {
    this.items = Array.from(
      container.querySelectorAll<HTMLElement>('[role="tab"]')
    );
    this.currentIndex = 0;
    this.init();
  }

  private init() {
    this.items.forEach((item, i) => {
      item.setAttribute('tabindex', i === 0 ? '0' : '-1');
      item.addEventListener('keydown', (e) => this.handleKeydown(e, i));
      item.addEventListener('click', () => this.moveTo(i));
    });
  }

  private handleKeydown(e: KeyboardEvent, index: number) {
    let next = index;

    switch (e.key) {
      case 'ArrowRight':
      case 'ArrowDown':
        e.preventDefault();
        next = (index + 1) % this.items.length;
        break;
      case 'ArrowLeft':
      case 'ArrowUp':
        e.preventDefault();
        next = (index - 1 + this.items.length) % this.items.length;
        break;
      case 'Home':
        e.preventDefault();
        next = 0;
        break;
      case 'End':
        e.preventDefault();
        next = this.items.length - 1;
        break;
      default:
        return;
    }

    this.moveTo(next);
  }

  moveTo(index: number) {
    this.items[this.currentIndex].setAttribute('tabindex', '-1');
    this.items[index].setAttribute('tabindex', '0');
    this.items[index].focus();
    this.currentIndex = index;
  }
}
```

### Contratos de Teclado para Componentes Customizados

As WAI-ARIA Authoring Practices definem as interações de teclado esperadas para cada tipo de widget:

| Widget | Teclas |
|--------|------|
| Button | Enter, Space para ativar |
| Checkbox | Space para alternar |
| Radio group | Teclas de seta para mover entre opções |
| Tab list | Teclas de seta para trocar tabs, Enter para ativar |
| Menu / Menubar | Teclas de seta, Escape para fechar, Enter para selecionar |
| Listbox | Teclas de seta, Space/Enter para selecionar, Type-ahead |
| Tree | Teclas de seta, Enter para expandir/recolher |
| Dialog | Escape para fechar, Tab percorre internamente |
| Date picker | Teclas de seta para dias, Page Up/Down para meses |

## Exemplos Práticos

### Modal React com Gerenciamento Completo de Focus

```tsx
import { useEffect, useRef, useCallback } from 'react';

interface ModalProps {
  isOpen: boolean;
  title: string;
  onClose: () => void;
  children: React.ReactNode;
}

function Modal({ isOpen, title, onClose, children }: ModalProps) {
  const modalRef = useRef<HTMLDivElement>(null);
  const titleId = `modal-title-${useId()}`;
  // Armazena o elemento que abriu o modal
  const previousFocusRef = useRef<HTMLElement | null>(null);

  useEffect(() => {
    if (isOpen) {
      // Salva o focus atual
      previousFocusRef.current = document.activeElement as HTMLElement;
      // Move o focus para dentro do dialog
      modalRef.current?.focus();
    } else {
      // Retorna o focus para o elemento que abriu o modal
      previousFocusRef.current?.focus();
    }
  }, [isOpen]);

  const handleKeydown = useCallback(
    (e: React.KeyboardEvent) => {
      if (e.key === 'Escape') {
        onClose();
        return;
      }

      if (e.key !== 'Tab') return;

      const container = modalRef.current;
      if (!container) return;

      const focusable = Array.from(
        container.querySelectorAll<HTMLElement>(
          'a[href], button:not([disabled]), input:not([disabled]), [tabindex="0"]'
        )
      );

      const first = focusable[0];
      const last = focusable[focusable.length - 1];

      if (e.shiftKey && document.activeElement === first) {
        e.preventDefault();
        last?.focus();
      } else if (!e.shiftKey && document.activeElement === last) {
        e.preventDefault();
        first?.focus();
      }
    },
    [onClose]
  );

  if (!isOpen) return null;

  return (
    <>
      {/* Backdrop */}
      <div className="modal-backdrop" onClick={onClose} aria-hidden="true" />

      <div
        ref={modalRef}
        role="dialog"
        aria-modal="true"
        aria-labelledby={titleId}
        tabIndex={-1}   // Torna o container focável programaticamente
        onKeyDown={handleKeydown}
        className="modal"
      >
        <div className="modal-header">
          <h2 id={titleId}>{title}</h2>
          <button
            type="button"
            onClick={onClose}
            aria-label={`Close ${title}`}
          >
            &times;
          </button>
        </div>
        <div className="modal-body">{children}</div>
      </div>
    </>
  );
}
```

### Tab List Acessível com Roving Tabindex

```tsx
import { useState, useRef, useId } from 'react';

interface Tab {
  id: string;
  label: string;
  content: React.ReactNode;
}

function TabList({ tabs }: { tabs: Tab[] }) {
  const [activeIndex, setActiveIndex] = useState(0);
  const baseId = useId();
  const tabRefs = useRef<(HTMLButtonElement | null)[]>([]);

  function handleKeydown(e: React.KeyboardEvent, index: number) {
    let next = index;
    if (e.key === 'ArrowRight') next = (index + 1) % tabs.length;
    else if (e.key === 'ArrowLeft') next = (index - 1 + tabs.length) % tabs.length;
    else if (e.key === 'Home') next = 0;
    else if (e.key === 'End') next = tabs.length - 1;
    else return;

    e.preventDefault();
    setActiveIndex(next);
    tabRefs.current[next]?.focus();
  }

  return (
    <div>
      <div role="tablist" aria-label="Content sections">
        {tabs.map((tab, i) => (
          <button
            key={tab.id}
            ref={(el) => { tabRefs.current[i] = el; }}
            role="tab"
            id={`${baseId}-tab-${tab.id}`}
            aria-controls={`${baseId}-panel-${tab.id}`}
            aria-selected={i === activeIndex}
            tabIndex={i === activeIndex ? 0 : -1}
            onClick={() => setActiveIndex(i)}
            onKeyDown={(e) => handleKeydown(e, i)}
          >
            {tab.label}
          </button>
        ))}
      </div>

      {tabs.map((tab, i) => (
        <div
          key={tab.id}
          role="tabpanel"
          id={`${baseId}-panel-${tab.id}`}
          aria-labelledby={`${baseId}-tab-${tab.id}`}
          hidden={i !== activeIndex}
          tabIndex={0}
        >
          {tab.content}
        </div>
      ))}
    </div>
  );
}
```

### Gerenciamento de Focus em Navegação SPA

```typescript
// Em um roteador client-side, após a navegação:
function handleRouteChange(newPath: string) {
  // Aguarda o DOM atualizar
  requestAnimationFrame(() => {
    // Opção 1: focar o heading principal
    const h1 = document.querySelector<HTMLElement>('h1');
    if (h1) {
      h1.setAttribute('tabindex', '-1');
      h1.focus();
      // Remove o tabindex para que o h1 não ganhe focus ring no clique do mouse
      h1.addEventListener('blur', () => h1.removeAttribute('tabindex'), { once: true });
    }

    // Opção 2: anunciar a mudança de rota via live region
    const announcer = document.getElementById('route-announcer');
    if (announcer) {
      announcer.textContent = '';
      requestAnimationFrame(() => {
        announcer.textContent = `Navegou para ${document.title}`;
      });
    }
  });
}
```

## Padrões e Boas Práticas

### O Hook `useId` para IDs Estáveis

No React 18+, use `useId` para gerar IDs estáveis para associações aria — nunca use Math.random() ou contadores para atributos de acessibilidade.

```tsx
function FormField({ label, error }: { label: string; error?: string }) {
  const id = useId();
  const errorId = `${id}-error`;
  return (
    <div>
      <label htmlFor={id}>{label}</label>
      <input
        id={id}
        aria-describedby={error ? errorId : undefined}
        aria-invalid={error ? 'true' : undefined}
      />
      {error && <span id={errorId}>{error}</span>}
    </div>
  );
}
```

### Estilos de Focus Visível

Todo elemento focável precisa de um indicador de focus visível que atenda à WCAG 2.4.11 (focus appearance):
- Área do outline ≥ perímetro do componente × 2px
- Razão de contraste ≥ 3:1 entre os estados com e sem focus

```css
/* Estilo de focus global — cobre todos os elementos */
:focus-visible {
  outline: 3px solid #1a56db;
  outline-offset: 2px;
  border-radius: 2px;
}

/* Remove o outline para cliques de mouse (browsers tratam :focus-visible automaticamente) */
:focus:not(:focus-visible) {
  outline: none;
}

/* Suporte ao modo de alto contraste */
@media (forced-colors: active) {
  :focus-visible {
    outline: 3px solid ButtonText;
  }
}
```

### O Atributo `inert` para Conteúdo em Segundo Plano

Quando um modal está aberto, use `inert` para tornar o conteúdo ao fundo não interativo sem precisar gerenciar o tabindex em cada elemento:

```html
<main id="main-content" inert><!-- segundo plano --></main>
<div role="dialog" aria-modal="true"><!-- primeiro plano --></div>
```

```typescript
function openModal() {
  document.getElementById('main-content')?.setAttribute('inert', '');
  document.getElementById('main-nav')?.setAttribute('inert', '');
  dialog.removeAttribute('hidden');
  dialogTitle.focus();
}

function closeModal() {
  dialog.setAttribute('hidden', '');
  document.getElementById('main-content')?.removeAttribute('inert');
  document.getElementById('main-nav')?.removeAttribute('inert');
  triggerButton.focus();
}
```

## Anti-Padrões a Evitar

**Valores positivos de tabindex**
```html
<!-- Ruim: tabindex="2" cria uma ordem de focus paralela, confunde a todos -->
<input tabindex="2" name="username">
<input tabindex="1" name="password">

<!-- Bom: deixe a ordem do DOM determinar a tab order -->
<input name="username">
<input name="password">
```

**Remover o outline de focus sem substituição**
```css
/* Ruim: usuários pressionando Tab não veem focus visível */
* { outline: none; }
button:focus { outline: none; }

/* Bom: substitua por um indicador personalizado com boa aparência */
button:focus-visible {
  outline: 3px solid #1a56db;
  outline-offset: 2px;
}
```

**Usar `<div>` para buttons sem tratamento de teclado**
```html
<!-- Ruim: o clique funciona, mas Enter/Space não -->
<div class="btn" onclick="submit()">Submit</div>

<!-- Bom -->
<button type="submit">Submit</button>
```

**Abrir dialogs sem mover o focus**
```typescript
// Ruim: o dialog aparece, mas o focus fica no elemento que o abriu
function showDialog() {
  dialog.style.display = 'block';
}

// Bom: mova o focus para dentro do dialog
function showDialog() {
  dialog.style.display = 'block';
  dialog.querySelector('h2')?.focus();
}
```

**Scroll infinito sem acesso por teclado ao novo conteúdo**
```typescript
// Ruim: novos itens são adicionados, mas o usuário de teclado não consegue alcançá-los
container.appendChild(newItems);

// Bom: anuncie e opcionalmente mova o focus
container.appendChild(newItems);
announcer.textContent = `${newItems.childElementCount} itens adicionais carregados.`;
// Se o usuário acionou explicitamente o carregamento: newItems.firstElementChild?.focus()
```

## Depuração e Solução de Problemas

### Checklist de Testes Manuais de Teclado

1. Desconecte o mouse
2. Pressione `Tab` repetidamente — todo elemento interativo deve ser alcançável em ordem lógica
3. Pressione `Shift+Tab` — a ordem reversa também deve ser lógica
4. Ative cada elemento com `Enter` ou `Space`
5. Abra modais — verifique se o focus entra neles; pressionar `Escape` deve fechar e retornar o focus ao elemento de origem
6. Navegue dentro de widgets compostos (tabs, dropdowns) usando teclas de seta
7. Verifique se o focus está sempre visível (sem focus invisível)

### Diagnosticando Problemas de Tab Order

```typescript
// Registrar a tab order de todos os elementos focáveis na página
const focusable = document.querySelectorAll(
  'a[href], button:not([disabled]), input:not([disabled]), [tabindex]:not([tabindex="-1"])'
);
console.log('Tab order:');
focusable.forEach((el, i) => {
  console.log(
    `${i + 1}. ${el.tagName} "${el.textContent?.trim() || el.getAttribute('aria-label') || el.getAttribute('id')}"`
  );
});
```

### Testando Gerenciamento de Focus no Playwright

```typescript
import { test, expect } from '@playwright/test';

test('modal traps focus correctly', async ({ page }) => {
  await page.goto('/');
  await page.click('button:has-text("Open dialog")');

  // O focus deve estar dentro do dialog
  const focused = await page.evaluate(() => document.activeElement?.getAttribute('role'));
  expect(['dialog', 'heading', 'button']).toContain(focused);

  // Tab deve percorrer apenas dentro do dialog
  const dialogFocusables = await page.$$('[role="dialog"] button, [role="dialog"] input');
  for (let i = 0; i < dialogFocusables.length + 2; i++) {
    await page.keyboard.press('Tab');
    const isInsideDialog = await page.evaluate(
      () => !!document.activeElement?.closest('[role="dialog"]')
    );
    expect(isInsideDialog).toBe(true);
  }

  // Escape fecha e restaura o focus
  await page.keyboard.press('Escape');
  await expect(page.locator('button:has-text("Open dialog")')).toBeFocused();
});
```

## Cenários do Mundo Real

### Cenário 1: Command Palette (Ctrl+K)

```tsx
function CommandPalette() {
  const [open, setOpen] = useState(false);
  const inputRef = useRef<HTMLInputElement>(null);
  const previousFocusRef = useRef<HTMLElement | null>(null);

  useEffect(() => {
    function handleGlobalKey(e: KeyboardEvent) {
      if ((e.ctrlKey || e.metaKey) && e.key === 'k') {
        e.preventDefault();
        setOpen((o) => {
          if (!o) {
            previousFocusRef.current = document.activeElement as HTMLElement;
          }
          return !o;
        });
      }
    }
    window.addEventListener('keydown', handleGlobalKey);
    return () => window.removeEventListener('keydown', handleGlobalKey);
  }, []);

  useEffect(() => {
    if (open) {
      inputRef.current?.focus();
    } else {
      previousFocusRef.current?.focus();
    }
  }, [open]);

  if (!open) return null;

  return (
    <div
      role="dialog"
      aria-modal="true"
      aria-label="Command palette"
      onKeyDown={(e) => e.key === 'Escape' && setOpen(false)}
    >
      <input
        ref={inputRef}
        type="search"
        aria-label="Search commands"
        placeholder="Type a command..."
      />
      {/* resultados como um listbox */}
    </div>
  );
}
```

### Cenário 2: Scroll Infinito com "Carregar Mais" Acessível por Teclado

```tsx
function ArticleList() {
  const [articles, setArticles] = useState<Article[]>(initialArticles);
  const [loading, setLoading] = useState(false);
  const loadMoreRef = useRef<HTMLButtonElement>(null);
  const firstNewRef = useRef<HTMLLIElement>(null);
  const announceRef = useRef<HTMLDivElement>(null);

  async function loadMore() {
    setLoading(true);
    const newArticles = await fetchArticles({ offset: articles.length });
    const previousLength = articles.length;
    setArticles((prev) => [...prev, ...newArticles]);
    setLoading(false);

    // Anuncia a atualização
    if (announceRef.current) {
      announceRef.current.textContent = `${newArticles.length} artigos adicionais carregados.`;
    }

    // Move o focus para o primeiro item novo (opcional — apenas no clique explícito do button)
    requestAnimationFrame(() => firstNewRef.current?.focus());
  }

  return (
    <div>
      <div role="status" ref={announceRef} aria-live="polite" className="sr-only" />

      <ul>
        {articles.map((article, i) => (
          <li
            key={article.id}
            ref={i === articles.length - (articles.length - /* previousLength */ 0) ? firstNewRef : null}
            tabIndex={-1}
          >
            <a href={`/articles/${article.slug}`}>{article.title}</a>
          </li>
        ))}
      </ul>

      <button
        ref={loadMoreRef}
        type="button"
        onClick={loadMore}
        disabled={loading}
        aria-busy={loading}
      >
        {loading ? 'Carregando...' : 'Carregar mais artigos'}
      </button>
    </div>
  );
}
```

## Leitura Complementar

- [WAI-ARIA Authoring Practices — Keyboard Navigation](https://www.w3.org/WAI/ARIA/apg/practices/keyboard-interface/)
- [WCAG 2.1 — Understanding Focus Order (2.4.3)](https://www.w3.org/WAI/WCAG21/Understanding/focus-order.html)
- [WebAIM: Keyboard Accessibility](https://webaim.org/techniques/keyboard/)
- [CSS-Tricks: A Guide to Keyboard Accessibility](https://css-tricks.com/a-guide-to-keyboard-accessibility-html-and-css-part-1/)
- [The `inert` attribute — MDN](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/inert)

## Resumo

Acessibilidade por teclado é fundamental. Os conceitos-chave:

- **Tab order** segue a ordem do código-fonte do DOM; nunca use valores positivos de `tabindex`
- **`tabindex="-1"`** torna elementos focáveis programaticamente sem entrar na tab order
- **Gerenciamento de focus** — sempre mova o focus quando a interface mudar significativamente (modais, transições de página, erros)
- **Focus traps** — obrigatório em modais; percorra o focus dentro do dialog, libere no Escape
- **Skip links** — primeiro elemento focável em todas as páginas; sempre obrigatório
- **Roving tabindex** — para widgets compostos (tabs, toolbars, menus): um item na tab order, teclas de seta navegam internamente
- **Focus visible** — CSS com `:focus-visible`; nunca remova o outline sem uma substituição
- **Atributo `inert`** — forma moderna e confiável de excluir conteúdo do segundo plano durante sessões com modal

Teste manualmente: desconecte o mouse e complete todos os fluxos de usuário apenas com o teclado. Se não conseguir, é uma violação da WCAG.
