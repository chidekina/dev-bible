# Keyboard Navigation

## Overview

Every interactive element on a web page must be reachable and operable with a keyboard alone. This is WCAG 2.1.1 (Level A) — not optional. Keyboard accessibility directly benefits users who cannot use a pointer device: people with motor disabilities, power users who prefer keyboard, voice control software users (Dragon NaturallySpeaking moves through focusable elements), and screen reader users (who navigate by Tab, arrows, and shortcuts).

Building good keyboard navigation requires understanding the native browser tab order, managing focus programmatically when UI changes, implementing focus traps in modal dialogs, providing skip links to bypass repetitive content, and building custom widgets with roving tabindex.

## Prerequisites

- Semantic HTML (see `semantic-html.md`)
- Basic ARIA concepts (see `aria-roles.md`)
- JavaScript / TypeScript (DOM event handling)

## Core Concepts

### The Natural Tab Order

By default, browsers create a tab order from elements that are natively focusable in DOM source order:

- `<a href="...">` — links
- `<button>` — buttons
- `<input>`, `<select>`, `<textarea>` — form controls
- `<details>` — disclosure
- Elements with `tabindex="0"` — explicitly added to tab order

`Tab` moves forward; `Shift+Tab` moves backward. The visual position on screen doesn't determine tab order — DOM order does. Reordering with CSS (flexbox `order`, CSS `grid-area`, `position: absolute`) while leaving DOM order unchanged breaks tab order.

### `tabindex` Attribute

| Value | Behavior |
|-------|----------|
| `tabindex="0"` | Add element to natural tab order at its DOM position |
| `tabindex="-1"` | Focusable programmatically (`element.focus()`), excluded from Tab |
| `tabindex="1"` or higher | Positive tabindex — creates separate focus order that runs before tabindex="0". **Never use.** |

```html
<!-- Custom interactive widget added to tab order -->
<div role="button" tabindex="0">Click me</div>

<!-- Panel reachable by script but not Tab -->
<div id="modal-content" tabindex="-1">...</div>
```

### Focus Management

When the page changes significantly — opening a modal, navigating to a new page in an SPA, showing an error summary — you must move focus to reflect the new context.

```typescript
// Move focus to an element programmatically
const element = document.getElementById('dialog-title');
element?.focus();

// Move focus to a panel revealed by user action
function showModal() {
  dialog.removeAttribute('hidden');
  dialogHeading.focus(); // focus the dialog's heading or container
}

// After closing a modal, return focus to the trigger
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

### Focus Traps in Modals

When a modal dialog is open, keyboard focus must be trapped inside it. Users pressing Tab should cycle through the modal's focusable elements — not escape into the page behind.

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
      // Shift+Tab: if at first element, wrap to last
      if (document.activeElement === first) {
        e.preventDefault();
        last.focus();
      }
    } else {
      // Tab: if at last element, wrap to first
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

Skip links allow keyboard users to jump past repeated navigation blocks directly to main content. Required by WCAG 2.4.1 (Level A).

```html
<!-- First focusable element on every page -->
<a href="#main-content" class="skip-link">Skip to main content</a>

<header>
  <!-- 50+ navigation links -->
</header>

<main id="main-content" tabindex="-1">
  <!-- tabindex="-1" so we can focus <main> programmatically,
       even though it's not natively focusable -->
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

Multiple skip links for complex pages:
```html
<a href="#main-content">Skip to content</a>
<a href="#site-search">Skip to search</a>
<a href="#main-nav">Skip to navigation</a>
```

### Roving Tabindex

For composite widgets (toolbars, tab lists, radio groups, menus, tree views), only one item should be in the tab order at a time. The user navigates between items with arrow keys. This is called "roving tabindex."

Pattern:
1. Set `tabindex="0"` on the currently active item
2. Set `tabindex="-1"` on all other items
3. On arrow key press, move focus and swap tabindex values

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

### Custom Component Keyboard Contracts

The WAI-ARIA Authoring Practices define expected keyboard interactions for each widget type:

| Widget | Keys |
|--------|------|
| Button | Enter, Space to activate |
| Checkbox | Space to toggle |
| Radio group | Arrow keys to move between options |
| Tab list | Arrow keys to switch tabs, Enter to activate |
| Menu / Menubar | Arrow keys, Escape to close, Enter to select |
| Listbox | Arrow keys, Space/Enter to select, Type-ahead |
| Tree | Arrow keys, Enter to expand/collapse |
| Dialog | Escape to close, Tab cycles within |
| Date picker | Arrow keys for days, Page Up/Down for months |

## Hands-On Examples

### React Modal with Full Focus Management

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
  // Store the element that triggered the modal
  const previousFocusRef = useRef<HTMLElement | null>(null);

  useEffect(() => {
    if (isOpen) {
      // Save current focus
      previousFocusRef.current = document.activeElement as HTMLElement;
      // Move focus into dialog
      modalRef.current?.focus();
    } else {
      // Return focus to trigger
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
        tabIndex={-1}   // Makes the container programmatically focusable
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

### Accessible Tab List with Roving Tabindex

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

### SPA Navigation Focus Management

```typescript
// In a client-side router, after navigation:
function handleRouteChange(newPath: string) {
  // Give the DOM time to update
  requestAnimationFrame(() => {
    // Option 1: Focus the main heading
    const h1 = document.querySelector<HTMLElement>('h1');
    if (h1) {
      h1.setAttribute('tabindex', '-1');
      h1.focus();
      // Clean up tabindex so h1 doesn't get a focus ring on mouse click
      h1.addEventListener('blur', () => h1.removeAttribute('tabindex'), { once: true });
    }

    // Option 2: Announce route change via live region
    const announcer = document.getElementById('route-announcer');
    if (announcer) {
      announcer.textContent = '';
      requestAnimationFrame(() => {
        announcer.textContent = `Navigated to ${document.title}`;
      });
    }
  });
}
```

## Common Patterns & Best Practices

### The `useId` Hook for Stable IDs

In React 18+, use `useId` to generate stable IDs for aria associations — never use Math.random() or counters for accessibility attributes.

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

### Focus Visible Styles

Every focusable element needs a visible focus indicator that meets WCAG 2.4.11 (focus appearance):
- Outline area ≥ perimeter of the component × 2px
- Contrast ratio ≥ 3:1 between focused and unfocused states

```css
/* Global focus style — covers all elements */
:focus-visible {
  outline: 3px solid #1a56db;
  outline-offset: 2px;
  border-radius: 2px;
}

/* Remove outline for mouse clicks (browsers handle :focus-visible automatically) */
:focus:not(:focus-visible) {
  outline: none;
}

/* High contrast mode support */
@media (forced-colors: active) {
  :focus-visible {
    outline: 3px solid ButtonText;
  }
}
```

### The `inert` Attribute for Background Content

When a modal is open, use `inert` to make the background content non-interactive without managing tabindex on every element:

```html
<main id="main-content" inert><!-- background --></main>
<div role="dialog" aria-modal="true"><!-- foreground --></div>
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

## Anti-Patterns to Avoid

**Positive tabindex values**
```html
<!-- Bad: tabindex="2" creates a parallel focus order, confusing for everyone -->
<input tabindex="2" name="username">
<input tabindex="1" name="password">

<!-- Good: let DOM order determine tab order -->
<input name="username">
<input name="password">
```

**Removing focus outline without replacement**
```css
/* Bad: users pressing Tab see no visible focus */
* { outline: none; }
button:focus { outline: none; }

/* Good: replace with a better-looking custom indicator */
button:focus-visible {
  outline: 3px solid #1a56db;
  outline-offset: 2px;
}
```

**Using `<div>` for buttons without keyboard handling**
```html
<!-- Bad: click works but Enter/Space don't -->
<div class="btn" onclick="submit()">Submit</div>

<!-- Good -->
<button type="submit">Submit</button>
```

**Opening dialogs without moving focus**
```typescript
// Bad: dialog appears but focus stays on the trigger
function showDialog() {
  dialog.style.display = 'block';
}

// Good: move focus into the dialog
function showDialog() {
  dialog.style.display = 'block';
  dialog.querySelector('h2')?.focus();
}
```

**Infinite scroll without keyboard access to new content**
```typescript
// Bad: new items appended but keyboard user can't reach them
container.appendChild(newItems);

// Good: announce and optionally move focus
container.appendChild(newItems);
announcer.textContent = `${newItems.childElementCount} more items loaded.`;
// If user explicitly triggered load: newItems.firstElementChild?.focus()
```

## Debugging & Troubleshooting

### Manual Keyboard Testing Checklist

1. Disconnect mouse
2. Press `Tab` repeatedly — every interactive element should be reachable in a logical order
3. Press `Shift+Tab` — reverse order should be logical too
4. Activate each element with `Enter` or `Space`
5. Open modals — verify focus moves in; pressing `Escape` closes and returns focus to trigger
6. Navigate within composite widgets (tabs, dropdowns) using arrow keys
7. Check that focus is always visible (no invisible focus)

### Diagnosing Tab Order Issues

```typescript
// Log tab order of all focusable elements on the page
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

### Testing Focus Management in Playwright

```typescript
import { test, expect } from '@playwright/test';

test('modal traps focus correctly', async ({ page }) => {
  await page.goto('/');
  await page.click('button:has-text("Open dialog")');

  // Focus should be inside the dialog
  const focused = await page.evaluate(() => document.activeElement?.getAttribute('role'));
  expect(['dialog', 'heading', 'button']).toContain(focused);

  // Tab should cycle within the dialog
  const dialogFocusables = await page.$$('[role="dialog"] button, [role="dialog"] input');
  for (let i = 0; i < dialogFocusables.length + 2; i++) {
    await page.keyboard.press('Tab');
    const isInsideDialog = await page.evaluate(
      () => !!document.activeElement?.closest('[role="dialog"]')
    );
    expect(isInsideDialog).toBe(true);
  }

  // Escape closes and restores focus
  await page.keyboard.press('Escape');
  await expect(page.locator('button:has-text("Open dialog")')).toBeFocused();
});
```

## Real-World Scenarios

### Scenario 1: Command Palette (Ctrl+K)

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
      {/* results as a listbox */}
    </div>
  );
}
```

### Scenario 2: Infinite Scroll with Keyboard-Accessible Load More

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

    // Announce the update
    if (announceRef.current) {
      announceRef.current.textContent = `${newArticles.length} more articles loaded.`;
    }

    // Move focus to the first new item (optional — only on explicit button click)
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
        {loading ? 'Loading...' : 'Load more articles'}
      </button>
    </div>
  );
}
```

## Further Reading

- [WAI-ARIA Authoring Practices — Keyboard Navigation](https://www.w3.org/WAI/ARIA/apg/practices/keyboard-interface/)
- [WCAG 2.1 — Understanding Focus Order (2.4.3)](https://www.w3.org/WAI/WCAG21/Understanding/focus-order.html)
- [WebAIM: Keyboard Accessibility](https://webaim.org/techniques/keyboard/)
- [CSS-Tricks: A Guide to Keyboard Accessibility](https://css-tricks.com/a-guide-to-keyboard-accessibility-html-and-css-part-1/)
- [The `inert` attribute — MDN](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/inert)

## Summary

Keyboard accessibility is foundational. The key concepts:

- **Tab order** follows DOM source order; never use positive `tabindex` values
- **`tabindex="-1"`** makes elements programmatically focusable without entering the tab order
- **Focus management** — always move focus when UI changes significantly (modals, page transitions, errors)
- **Focus traps** — required in modals; cycle focus within the dialog, release on Escape
- **Skip links** — first focusable element on every page; always required
- **Roving tabindex** — for composite widgets (tabs, toolbars, menus): one item in tab order, arrow keys navigate within
- **Focus visible** — `:focus-visible` CSS; never remove the outline without a replacement
- **`inert` attribute** — modern, reliable way to exclude background content during modal sessions

Test manually: disconnect the mouse and complete every user flow by keyboard alone. If you can't, it's a WCAG violation.
