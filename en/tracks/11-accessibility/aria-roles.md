# ARIA Roles

## Overview

Accessible Rich Internet Applications (ARIA) is a set of HTML attributes that define ways to make web content more accessible to assistive technologies. ARIA adds roles, properties, and states to elements that otherwise have no semantic meaning — or overrides native semantics when necessary.

The first rule of ARIA is: **don't use ARIA**. If a native HTML element or attribute covers your use case, use it. A `<button>` is always better than a `<div role="button">`. But custom interactive components — comboboxes, disclosure widgets, carousels, data grids — often have no native HTML equivalent, and that's where ARIA becomes essential.

This chapter covers when to reach for ARIA, how roles/properties/states work, live regions for dynamic content, and the full implementation of the combobox pattern.

## Prerequisites

- Solid understanding of semantic HTML (see `semantic-html.md`)
- Basic JavaScript / TypeScript
- Familiarity with browser DevTools accessibility panel

## Core Concepts

### The Three ARIA Attributes

**`role`** — what the element is:
```html
<div role="button">Click me</div>
<div role="dialog">...</div>
<div role="alert">Error: file not found</div>
```

**`aria-*` properties** — persistent characteristics:
```html
<input aria-label="Search" aria-required="true">
<nav aria-labelledby="nav-title">
<div role="progressbar" aria-valuemin="0" aria-valuemax="100" aria-valuenow="45">
```

**`aria-*` states** — dynamic conditions that change with user interaction:
```html
<button aria-expanded="false">Menu</button>
<input aria-invalid="true">
<li role="option" aria-selected="true">
```

### The ARIA Hierarchy

ARIA roles are organized in a taxonomy:

```
abstract roles (never use directly)
└── widget roles (interactive UI controls)
    ├── button, checkbox, combobox, dialog, grid, link, listbox ...
└── document structure roles (non-interactive)
    ├── article, figure, heading, list, table ...
└── landmark roles (page regions)
    ├── banner, complementary, contentinfo, form, main, navigation, region, search
└── live region roles
    ├── alert, log, marquee, status, timer
└── window roles
    ├── alertdialog, dialog
```

### When to Use ARIA vs Native HTML

| Use case | Native HTML | ARIA alternative |
|----------|-------------|-----------------|
| Button | `<button>` | `<div role="button" tabindex="0">` |
| Checkbox | `<input type="checkbox">` | `<div role="checkbox" aria-checked="false">` |
| Link | `<a href="...">` | `<div role="link" tabindex="0">` |
| Heading | `<h2>` | `<div role="heading" aria-level="2">` |
| List | `<ul>/<li>` | `<div role="list"><div role="listitem">` |
| Dialog | `<dialog>` | `<div role="dialog" aria-modal="true">` |

Prefer the native HTML column in every case. Use ARIA when:
1. You're building a custom widget with no HTML equivalent (combobox, tree, data grid)
2. You must override native semantics (e.g., a `<div>` in a framework that renders a button)
3. You need to communicate dynamic state changes

### Common ARIA Properties

**Naming and labeling:**
```html
<!-- aria-label: direct string label -->
<button aria-label="Close dialog">
  <svg aria-hidden="true">...</svg>
</button>

<!-- aria-labelledby: reference another element's text -->
<h2 id="dialog-title">Delete file</h2>
<div role="dialog" aria-labelledby="dialog-title">...</div>

<!-- aria-describedby: supplementary description -->
<input id="pw" type="password" aria-describedby="pw-hint">
<p id="pw-hint">Must be 8+ characters with at least one number.</p>
```

**Relationships:**
```html
<!-- aria-controls: this element controls another -->
<button aria-controls="panel1" aria-expanded="false">Section 1</button>
<div id="panel1" hidden>Content</div>

<!-- aria-owns: indicates ownership when DOM order doesn't reflect it -->
<ul role="listbox" aria-owns="extra-option">
  <li role="option">Option 1</li>
</ul>
<li id="extra-option" role="option">Option 2</li>
```

**States:**
```html
<button aria-pressed="true">Bold</button>      <!-- toggle button -->
<button aria-expanded="false">Menu</button>    <!-- disclosure -->
<li role="option" aria-selected="true">       <!-- selection -->
<input aria-invalid="true">                    <!-- validation error -->
<div aria-busy="true">Loading...</div>         <!-- async loading -->
<button aria-disabled="true">Submit</button>   <!-- disabled (focusable) -->
```

### Live Regions

Live regions announce content changes to screen readers without moving focus. This is how you notify users of dynamic updates — toasts, status messages, search results count, error summaries.

| Role / Attribute | Politeness | Use Case |
|-----------------|------------|----------|
| `role="alert"` | Assertive | Errors, critical status (interrupts current reading) |
| `role="status"` | Polite | Success messages, non-critical updates |
| `aria-live="polite"` | Polite | Custom live region (finishes current sentence first) |
| `aria-live="assertive"` | Assertive | Custom live region (interrupts immediately) |
| `role="log"` | Polite | Chat messages, audit logs (additive content) |
| `role="timer"` | Off | Countdown timers (AT won't auto-read) |

```html
<!-- Toast notification -->
<div role="status" aria-live="polite" aria-atomic="true">
  <!-- Injected dynamically: "File saved successfully" -->
</div>

<!-- Error summary -->
<div role="alert">
  <!-- Injected: "3 errors found. Please correct them before continuing." -->
</div>
```

Key attribute: `aria-atomic="true"` — announce the entire region when any part changes (vs only changed parts).

### The Combobox Pattern

A combobox is a combination of a text input and a popup listbox. It's one of the most common ARIA patterns, used for autocomplete search, filtered selects, and multi-select inputs.

Full WAI-ARIA Authoring Practices implementation:

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

## Hands-On Examples

### Building an Accessible Toggle Button

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

// Usage: Bold formatting toggle
<ToggleButton onToggle={(on) => applyFormatting('bold', on)}>
  Bold
</ToggleButton>
```

### Toast Notification System

```tsx
import { useState, useRef, useEffect } from 'react';

interface Toast {
  id: string;
  message: string;
  type: 'success' | 'error';
}

function ToastContainer() {
  const [toasts, setToasts] = useState<Toast[]>([]);
  // Live region ref — must exist in DOM before content is injected
  const liveRef = useRef<HTMLDivElement>(null);

  function addToast(message: string, type: Toast['type']) {
    const id = crypto.randomUUID();
    setToasts((prev) => [...prev, { id, message, type }]);
    // Auto-dismiss after 5s
    setTimeout(() => {
      setToasts((prev) => prev.filter((t) => t.id !== id));
    }, 5000);
  }

  return (
    <>
      {/* Live region — always rendered, content changes announce */}
      <div
        ref={liveRef}
        role={toasts.some((t) => t.type === 'error') ? 'alert' : 'status'}
        aria-live={toasts.some((t) => t.type === 'error') ? 'assertive' : 'polite'}
        aria-atomic="true"
        className="sr-only"
      >
        {toasts[toasts.length - 1]?.message}
      </div>

      {/* Visual toast display */}
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

### Accessible Disclosure Widget (Details/Summary Alternative)

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

### Full Combobox Implementation (TypeScript)

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

## Common Patterns & Best Practices

### ARIA Naming Precedence

When AT computes a name for an element, it uses this priority order:
1. `aria-labelledby` (references visible text — most authoritative)
2. `aria-label` (explicit string)
3. Native label association (`<label for="...">`)
4. Element contents (for buttons, links)
5. `title` attribute (last resort, unreliable)

### Hiding Content Correctly

```html
<!-- Visible to everyone -->
<p>Normal content</p>

<!-- Hidden from everyone (display:none / hidden attr) -->
<div hidden>Not visible, not in AT tree</div>

<!-- Visible on screen, hidden from AT -->
<div aria-hidden="true">Decorative icon or redundant text</div>

<!-- Hidden on screen, visible to AT (sr-only class) -->
<span class="sr-only">Additional context for screen readers</span>
```

### Progressive Enhancement with ARIA

Add ARIA to enhance, not replace, functional HTML:

```tsx
// Phase 1: functional without ARIA
<select name="country">
  <option>Brazil</option>
  <option>United States</option>
</select>

// Phase 2: custom combobox with full ARIA when you need the UX
// (only build custom when <select> genuinely cannot meet your design requirements)
<Combobox label="Country" options={countries} onSelect={setCountry} />
```

### Live Region Best Practices

- **Render live regions at page load** — inject content dynamically after. Browsers register live regions on render; adding `role="alert"` to an existing element with content won't always work.
- **Avoid over-announcing** — don't make every state change a live region
- **Use `aria-atomic="true"` for self-contained messages** (toast, status)
- **Use `aria-atomic="false"` for logs** where only new entries matter

## Anti-Patterns to Avoid

**Redundant ARIA roles on semantic elements**
```html
<!-- Bad: <button> already has role="button" -->
<button role="button">Click</button>

<!-- Bad: <nav> already has role="navigation" -->
<nav role="navigation">...</nav>
```

**Missing keyboard support for ARIA widgets**
```html
<!-- Bad: role="button" with no tabindex or keyboard handler -->
<div role="button" onclick="doThing()">Click me</div>

<!-- Good: tabindex makes it focusable, keydown handles Enter/Space -->
<div
  role="button"
  tabindex="0"
  onclick="doThing()"
  onkeydown="if(e.key==='Enter'||e.key===' ')doThing()"
>
  Click me
</div>
```

**Using `aria-hidden="true"` on focusable elements**
```html
<!-- Bad: element is hidden from AT but still receives focus -->
<a href="/home" aria-hidden="true">Home</a>

<!-- Good: also remove from tab order -->
<a href="/home" aria-hidden="true" tabindex="-1">Home</a>
<!-- Or better: just hide it properly -->
<a href="/home" hidden>Home</a>
```

**`aria-label` duplicating visible text**
```html
<!-- Bad: same text, redundant -->
<button aria-label="Submit">Submit</button>

<!-- Good: aria-label only when it differs from visible text -->
<button aria-label="Submit registration form">Submit</button>
```

**Announcing on every keystroke**
```tsx
// Bad: live region fires on every character typed
<div role="status" aria-live="polite">{searchResultCount} results</div>

// Good: debounce the update
const [count, setCount] = useState(0);
const debouncedCount = useDebounce(searchResultCount, 500);
useEffect(() => setCount(debouncedCount), [debouncedCount]);
<div role="status" aria-live="polite">{count} results</div>
```

## Debugging & Troubleshooting

### Inspecting Computed ARIA Properties

```bash
# Chrome DevTools → Elements → select element → Accessibility tab
# Shows: role, name, description, properties, state, and full tree

# Or via Playwright in tests:
const snapshot = await page.accessibility.snapshot({ root: await page.$('#dialog') });
console.log(JSON.stringify(snapshot, null, 2));
```

### Catching ARIA Issues with axe

```bash
npx axe https://localhost:3000 \
  --rules aria-allowed-attr,aria-required-attr,aria-valid-attr-value,\
          aria-roles,aria-hidden-focus,aria-required-children
```

### Testing Live Regions

Live regions are hard to test programmatically. Combine:
1. **axe-core** — checks `aria-live` attribute values are valid
2. **jest-axe** — catches missing `aria-atomic` and invalid roles
3. **Manual screen reader test** — trigger the dynamic change and listen for announcement

```typescript
// Verify the live region exists and has correct attributes
test('status region is configured correctly', async ({ page }) => {
  await page.goto('/dashboard');
  const status = page.locator('[role="status"]');
  await expect(status).toHaveAttribute('aria-live', 'polite');
  await expect(status).toHaveAttribute('aria-atomic', 'true');
});
```

## Real-World Scenarios

### Scenario 1: Search with Live Result Count

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
    setAnnouncement('Searching...');
    const data = await search(q);
    setResults(data);
    setSearching(false);
    setAnnouncement(
      data.length === 0
        ? 'No results found.'
        : `${data.length} result${data.length !== 1 ? 's' : ''} found.`
    );
  }

  return (
    <div>
      {/* Always-present live region */}
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

### Scenario 2: Accessible Modal Dialog

```tsx
function Modal({ title, onClose, children }: {
  title: string;
  onClose: () => void;
  children: React.ReactNode;
}) {
  const titleId = useId();
  const descId = useId();

  // Focus management handled in keyboard-navigation.md
  // Here we focus on ARIA attributes
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

## Further Reading

- [WAI-ARIA Specification 1.2](https://www.w3.org/TR/wai-aria-1.2/) — complete role taxonomy
- [WAI-ARIA Authoring Practices Guide](https://www.w3.org/WAI/ARIA/apg/) — design patterns with code examples
- [ARIA in HTML (W3C)](https://www.w3.org/TR/html-aria/) — allowed ARIA usage per HTML element
- [Deque: ARIA Role Definitions](https://dequeuniversity.com/library/aria/) — practical reference
- [Inclusive Components by Heydon Pickering](https://inclusive-components.design/) — deep dives into common patterns

## Summary

ARIA adds meaning to custom components that HTML can't express natively. The three attributes are `role` (what the element is), properties (persistent characteristics like `aria-label`), and states (dynamic conditions like `aria-expanded`).

The golden rules:
- Use native HTML first. ARIA is a fallback, not a replacement.
- Every ARIA widget needs full keyboard support — ARIA communicates but doesn't implement behavior.
- Live regions (`role="status"`, `role="alert"`) must be present in the DOM before you inject content.
- Don't use `aria-hidden="true"` on focusable elements.
- `aria-labelledby` takes priority over `aria-label` takes priority over content.
- Test with actual screen readers — automated tools miss many ARIA errors.
