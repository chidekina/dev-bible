# HTML & CSS

> HTML gives the web structure; CSS gives it presentation. Together they form the foundation of every interface. This file covers semantic markup, accessibility, the box model, positioning, stacking contexts, Flexbox, Grid, responsive design, CSS custom properties, and performance-conscious animations.

---

## 1. What & Why

HTML and CSS are the oldest and most widely deployed programming languages in existence. Yet they are also among the most misunderstood — often dismissed as "not real programming" until a layout refuses to work, a screen reader misreads content, or a z-index bug appears on a random browser. Mastering HTML and CSS means:

- Building accessible interfaces that work for everyone including users with disabilities
- Writing maintainable CSS that does not degrade into specificity wars
- Understanding layout systems well enough to implement any design without hacks
- Knowing which CSS properties trigger expensive layout/paint vs cheap composite-only changes
- Writing semantic markup that improves SEO and is correctly interpreted by search engines and assistive technologies

---

## 2. Core Concepts

### Semantic HTML

Semantic elements carry meaning about the content they contain. This meaning is used by browsers, screen readers, search engine crawlers, and developer tools.

| Element | Meaning |
|---------|---------|
| `<header>` | Introductory content for its nearest sectioning ancestor (page-level or within `<article>`) |
| `<nav>` | A set of navigation links |
| `<main>` | The dominant content of the `<body>` — only one per page |
| `<article>` | Self-contained content that makes sense independently (blog post, news article, comment) |
| `<section>` | A thematic grouping of content with a heading — not a generic wrapper (use `<div>` for that) |
| `<aside>` | Content tangentially related to the main content (sidebar, related links) |
| `<footer>` | Footer for its nearest sectioning ancestor (page footer or within `<article>`) |
| `<figure>` + `<figcaption>` | Self-contained visual content with a caption |
| `<time datetime="2024-01-01">` | A specific date/time — `datetime` provides machine-readable format |
| `<address>` | Contact information for the nearest `<article>` or `<body>` |
| `<mark>` | Highlighted text (search results) |
| `<abbr title="...">` | Abbreviation with full expansion on hover |

**Why semantic HTML matters:**
- Screen readers use landmarks (`<nav>`, `<main>`, `<aside>`) to let users jump directly to sections
- Search engines weight content in `<article>` and `<h1>` more heavily
- Developer tooling (VS Code IntelliSense, Lighthouse audits) can flag misused elements
- `<button>` is keyboard-focusable and activatable by Enter/Space; a `<div>` styled as a button is not

---

### Accessibility (a11y)

**ARIA (Accessible Rich Internet Applications):**

ARIA attributes add semantic information when native HTML semantics are insufficient. The first rule of ARIA: use native HTML semantics first. Only add ARIA if there is no native element that does what you need.

```html
<!-- Prefer native button over ARIA role -->
<button type="button">Save</button>  <!-- good -->
<div role="button" tabindex="0">Save</div>  <!-- only if you must -->

<!-- Label a form input -->
<label for="email">Email address</label>
<input id="email" type="email" aria-describedby="email-hint" />
<span id="email-hint">We will never share your email</span>

<!-- Landmark roles (redundant on semantic elements, needed on divs) -->
<div role="navigation" aria-label="Main navigation">...</div>

<!-- Live regions — announce dynamic content to screen readers -->
<div aria-live="polite" aria-atomic="true" id="status-message">
  <!-- Content injected here is announced without focus change -->
</div>

<!-- aria-live values:
     polite   — announces when user is idle (form validation messages)
     assertive — interrupts immediately (critical errors) -->
```

**Keyboard navigation:**
- All interactive elements must be reachable by Tab in logical order
- Focus must be visible (do not `outline: none` without a replacement)
- Modals must trap focus while open; return focus to trigger when closed
- Menus: use arrow keys within the menu, Escape to close

**Alt text:**
```html
<!-- Informative image: describe what it conveys -->
<img src="chart.png" alt="Bar chart showing Q4 revenue up 23% year-over-year" />

<!-- Decorative image: empty alt attribute (screen reader skips it) -->
<img src="divider.png" alt="" role="presentation" />

<!-- Images as links: describe the destination -->
<a href="/home"><img src="logo.png" alt="Company name — home page" /></a>
```

---

### The Box Model

Every element in CSS is a box. The box model defines how the dimensions of that box are calculated:

```
┌─────────────────────────────────┐  ← margin edge
│            margin               │
│  ┌───────────────────────────┐  │  ← border edge
│  │          border           │  │
│  │  ┌─────────────────────┐  │  │  ← padding edge
│  │  │       padding       │  │  │
│  │  │  ┌───────────────┐  │  │  │  ← content edge
│  │  │  │    content    │  │  │  │
│  │  │  └───────────────┘  │  │  │
│  │  └─────────────────────┘  │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

**`box-sizing: content-box` (default, confusing):**
```css
.box {
  width: 200px;
  padding: 20px;
  border: 5px solid;
  /* actual rendered width = 200 + 20*2 + 5*2 = 250px */
}
```

**`box-sizing: border-box` (what you actually want):**
```css
.box {
  width: 200px;
  padding: 20px;
  border: 5px solid;
  /* actual rendered width = 200px (padding and border fit inside) */
}
```

> 💡 Apply `border-box` globally as a reset:
> ```css
> *, *::before, *::after { box-sizing: border-box; }
> ```
> This is included in every modern CSS reset (Normalize, Tailwind base).

**Margin collapsing:** Adjacent vertical margins collapse into the larger of the two. This only happens in normal flow (not in flex or grid containers).
```css
/* h2 has margin-bottom: 16px, p has margin-top: 8px */
/* The gap between them is 16px (larger), NOT 24px */
```

---

### Positioning

```
static    → normal flow (default, top/left/right/bottom have no effect)
relative  → normal flow + offset from its natural position (creates stacking context with z-index)
absolute  → removed from flow, positioned relative to nearest non-static ancestor
fixed     → removed from flow, positioned relative to viewport (stays on scroll)
sticky    → normal flow until threshold, then fixed within its scroll container
```

**ASCII diagrams:**

```
static (default):
┌─────────┐
│  block  │  occupies its natural space in the flow
└─────────┘

relative (offset, still occupies original space):
┌─────────┐          ┌ ─ ─ ─ ─ ─┐
│         │  ──────> │  element  │ (shifted right 20px, ghost space remains)
└─────────┘          └ ─ ─ ─ ─ ─┘

absolute (removed from flow, latches to ancestor):
Parent (position: relative)
┌─────────────────────────┐
│                         │
│              ┌────────┐ │ ← absolute child, top: 10px, right: 10px
│              │ child  │ │
│              └────────┘ │
└─────────────────────────┘

fixed (viewport-relative, ignores scroll):
┌─────────────────────────────────────┐ ← viewport
│  ┌──────────────────────────────┐   │
│  │         sticky navbar        │   │ ← fixed, top: 0
│  └──────────────────────────────┘   │
│                                     │
│  ... scrollable content ...         │
└─────────────────────────────────────┘

sticky (relative until threshold, then fixed within scroll container):
Scrolled past threshold:
┌─────────────────────────────────────┐ ← scroll container
│  ┌──────────────────────────────┐   │
│  │     sticky section header   │   │ ← stuck at top: 0
│  └──────────────────────────────┘   │
│  ... section content scrolling ...  │
└─────────────────────────────────────┘
```

---

### Stacking Context

A stacking context is an independent layer in the Z-axis. Elements within a stacking context are stacked relative to each other, not to the global stack.

**What creates a stacking context:**
- `position: relative/absolute/fixed/sticky` + `z-index` other than `auto`
- `transform` (any value other than `none`)
- `opacity` less than 1
- `filter`, `backdrop-filter`
- `will-change: transform/opacity`
- `isolation: isolate` (explicitly creates one without side effects)
- `clip-path`, `mask`
- Flex/Grid children with `z-index` other than `auto`

> ⚠️ **Why z-index "doesn't work":** If your element's parent has a lower z-index in its own stacking context, your child can never visually escape it. Setting `z-index: 9999` on a child is useless if its parent stacking context is stacked below another element's stacking context.

```html
<!-- z-index trap example -->
<div style="position: relative; z-index: 1;">  <!-- stacking context A -->
  <div style="z-index: 9999;">...</div>         <!-- z-index 9999 within A -->
</div>
<div style="position: relative; z-index: 2;">  <!-- stacking context B, above A -->
  <div style="z-index: 1;">...</div>            <!-- still above all of A -->
</div>
```

---

## 3. How It Works

### Flexbox — Complete Guide

Flexbox is a one-dimensional layout system (row or column).

```css
/* Container properties */
.container {
  display: flex;

  flex-direction: row;           /* row | row-reverse | column | column-reverse */
  justify-content: flex-start;   /* main axis: flex-start | center | flex-end | space-between | space-around | space-evenly */
  align-items: stretch;          /* cross axis: stretch | flex-start | center | flex-end | baseline */
  flex-wrap: nowrap;             /* nowrap | wrap | wrap-reverse */
  gap: 16px;                     /* shorthand for row-gap + column-gap */
  align-content: normal;         /* multi-line cross axis: same values as justify-content */
}

/* Item properties */
.item {
  flex-grow: 0;      /* how much extra space this item takes (0 = don't grow) */
  flex-shrink: 1;    /* how much this item shrinks when space is tight (0 = don't shrink) */
  flex-basis: auto;  /* initial size before grow/shrink (can be px, %, auto) */
  flex: 1;           /* shorthand: flex-grow flex-shrink flex-basis = 1 1 0% */

  align-self: auto;  /* override align-items for this item */
  order: 0;          /* change visual order (default 0, lower = earlier) */
}
```

**Common flex patterns:**
```css
/* Center anything horizontally and vertically */
.center { display: flex; justify-content: center; align-items: center; }

/* Sticky footer pattern */
.page { display: flex; flex-direction: column; min-height: 100vh; }
.content { flex: 1; }  /* takes all remaining space */

/* Equal-width columns that shrink gracefully */
.cols { display: flex; gap: 16px; }
.col { flex: 1; min-width: 0; }  /* min-width: 0 prevents overflow in flex items */
```

---

### CSS Grid — Complete Guide

Grid is a two-dimensional layout system (rows and columns simultaneously).

```css
/* Container */
.grid {
  display: grid;

  /* Define columns */
  grid-template-columns: 1fr 2fr 1fr;          /* 3 columns, middle is 2x wider */
  grid-template-columns: repeat(3, 1fr);        /* same */
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));  /* responsive columns */
  grid-template-columns: 250px 1fr;             /* fixed sidebar + fluid content */

  /* Define rows (optional — rows auto-create by default) */
  grid-template-rows: auto 1fr auto;            /* header, content, footer */

  /* Named areas */
  grid-template-areas:
    "header  header  header"
    "sidebar content content"
    "footer  footer  footer";

  gap: 16px;              /* or row-gap + column-gap separately */
  align-items: stretch;   /* align items on block axis */
  justify-items: stretch; /* align items on inline axis */
}

/* Items */
.header { grid-area: header; }        /* use named area */
.sidebar { grid-area: sidebar; }
.content { grid-area: content; }

/* Or use line numbers */
.span-two {
  grid-column: 1 / 3;    /* start line 1, end line 3 (spans 2 columns) */
  grid-row: 2 / 4;       /* start row 2, end row 4 */
}
.full-width {
  grid-column: 1 / -1;   /* -1 = last line, spans all columns */
}
```

**`auto-fill` vs `auto-fit`:**
```css
/* auto-fill: creates as many columns as possible, leaves empty ones */
grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));

/* auto-fit: creates columns, collapses empty ones (items stretch to fill) */
grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));

/* With few items: auto-fit stretches them, auto-fill leaves empty space */
```

> 💡 **Flexbox vs Grid:** Use Flexbox when laying out items in a single direction (nav links, button groups, cards in a row) and the layout should adapt to content size. Use Grid when you need control over both rows and columns simultaneously (page layouts, image galleries, complex forms).

---

### Responsive Design

**Mobile-first approach** — write base styles for small screens, then add `min-width` media queries for larger screens:

```css
/* Mobile base — no media query needed */
.card { padding: 16px; }
.grid { grid-template-columns: 1fr; }

/* Tablet */
@media (min-width: 768px) {
  .grid { grid-template-columns: repeat(2, 1fr); }
}

/* Desktop */
@media (min-width: 1024px) {
  .card { padding: 24px; }
  .grid { grid-template-columns: repeat(3, 1fr); }
}
```

**Viewport meta tag** — required for responsive design on mobile:
```html
<meta name="viewport" content="width=device-width, initial-scale=1" />
<!-- Without this, mobile browsers render at desktop width and scale down -->
```

**Fluid typography with `clamp()`:**
```css
/* Font scales from 16px (at 320px viewport) to 24px (at 1200px viewport) */
.heading {
  font-size: clamp(1rem, 0.5rem + 2.5vw, 1.5rem);
  /* min, preferred (viewport-relative), max */
}
```

**Container queries** (modern browsers, no IE):
```css
/* Style based on the container's width, not the viewport */
.card-container { container-type: inline-size; }

@container (min-width: 400px) {
  .card { display: flex; gap: 16px; }
}
```

---

### CSS Custom Properties (Variables)

```css
/* Declaration — in :root for global scope */
:root {
  --color-primary: #3b82f6;
  --color-text: #1f2937;
  --spacing-sm: 8px;
  --spacing-md: 16px;
  --radius: 6px;
}

/* Usage */
.button {
  background: var(--color-primary);
  padding: var(--spacing-sm) var(--spacing-md);
  border-radius: var(--radius);
}

/* Fallback value */
.box { color: var(--color-brand, #000); }

/* Scoped override */
.dark-theme {
  --color-text: #f9fafb;
  --color-primary: #60a5fa;
}

/* JavaScript access */
const root = document.documentElement;
root.style.setProperty('--color-primary', '#ef4444');
const value = getComputedStyle(root).getPropertyValue('--color-primary');
```

---

### Animations and Transitions

**Transitions** — smooth change when a CSS property changes:
```css
.button {
  background: blue;
  transform: scale(1);
  transition: background 200ms ease, transform 150ms ease;
  /* property duration timing-function */
}
.button:hover {
  background: darkblue;
  transform: scale(1.05);
}
```

**@keyframes animations:**
```css
@keyframes fadeIn {
  from { opacity: 0; transform: translateY(-10px); }
  to   { opacity: 1; transform: translateY(0); }
}

.modal {
  animation: fadeIn 300ms ease forwards;
  /* name duration timing fill-mode */
}

/* Pause on reduced motion preference */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

**Performance — composite-only properties:**

Changing most CSS properties triggers layout → paint → composite (expensive). Only two properties are composited on the GPU without triggering layout or paint:
- `transform` (translate, scale, rotate)
- `opacity`

```css
/* Bad — triggers layout reflow on every frame */
.bad-animation { animation: move-bad 1s; }
@keyframes move-bad { to { left: 100px; } }

/* Good — GPU-composited, no layout, 60fps */
.good-animation { animation: move-good 1s; }
@keyframes move-good { to { transform: translateX(100px); } }

/* Hint browser to prepare a compositing layer */
.will-animate { will-change: transform; }
/* Remove after animation if many elements — will-change uses GPU memory */
```

---

## 4. Code Examples

### Semantic page structure
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Article Title — Site Name</title>
  <meta name="description" content="A 150-character description for SEO." />
</head>
<body>
  <header>
    <nav aria-label="Main navigation">
      <a href="/">Home</a>
      <a href="/about" aria-current="page">About</a>
    </nav>
  </header>

  <main>
    <article>
      <header>
        <h1>Article Title</h1>
        <time datetime="2024-01-15">January 15, 2024</time>
        <address rel="author">By <a href="/authors/alice">Alice</a></address>
      </header>

      <section aria-labelledby="intro-heading">
        <h2 id="intro-heading">Introduction</h2>
        <p>...</p>
      </section>
    </article>

    <aside aria-label="Related articles">
      <h2>Related</h2>
      <ul>...</ul>
    </aside>
  </main>

  <footer>
    <p><small>&copy; 2024 Site Name</small></p>
  </footer>
</body>
</html>
```

### CSS specificity and the cascade
```css
/* Specificity: (inline, IDs, classes/attrs/pseudos, elements) */

p { color: black; }                        /* (0,0,0,1) */
.text { color: blue; }                     /* (0,0,1,0) wins over p */
#content { color: green; }                 /* (0,1,0,0) wins over .text */
style="color: red"                         /* (1,0,0,0) wins over everything */

/* Calculating */
nav ul li a.active:hover { ... }
/* 0 IDs, 2 classes/pseudos (active, hover), 3 elements (nav, ul, li, a) = (0,0,2,4) */

/* !important overrides everything (except another !important with higher specificity) */
/* Avoid it — it makes debugging a nightmare */
```

### Centering patterns
```css
/* 1. Flexbox (recommended for most cases) */
.container {
  display: flex;
  justify-content: center;
  align-items: center;
}

/* 2. Grid */
.container {
  display: grid;
  place-items: center;  /* shorthand for align-items + justify-items */
}

/* 3. Absolute positioning (when you know dimensions or use transform) */
.child {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}

/* 4. Margin auto (horizontal only, block elements with known width) */
.child {
  width: 300px;
  margin: 0 auto;
}
```

---

## 5. Common Mistakes & Pitfalls

> ⚠️ **Specificity wars**: When multiple developers add `!important` to override each other, the cascade breaks down. Solution: use a consistent naming system (BEM, CSS Modules, utility classes) that avoids specificity conflicts by design. The universal selector `*` has specificity 0; elements have 1; classes have 10; IDs have 100.

> ⚠️ **z-index without a positioning context**: `z-index` has no effect on elements with `position: static` (the default). You must set `position: relative` (or any non-static value) for z-index to apply. This is the most common z-index bug.

> ⚠️ **Using `!important`**: It breaks the cascade and makes future overrides nearly impossible. The only valid use case is utility classes that must always win (like `display: none` in `.hidden { display: none !important }`).

> ⚠️ **Margin collapsing surprise**: Vertical margins between adjacent block siblings collapse to the largest value. Margins also collapse between a parent and its first/last child if there is no border, padding, or block formatting context between them. Flexbox and Grid containers do not collapse margins.

> ⚠️ **Not using `min-width: 0` on flex children**: Flex items default to `min-width: auto`, which prevents them from shrinking below their content size. Long text or images can overflow the container. Set `min-width: 0` on flex children that contain potentially overflowing content.

> ⚠️ **Animating layout-triggering properties**: Animating `width`, `height`, `top`, `left`, `margin`, or `padding` forces the browser to recalculate layout on every frame, which causes jank at < 60fps. Use `transform` and `opacity` instead.

> ⚠️ **Missing `lang` attribute on `<html>`**: Screen readers use this to determine which language pronunciation rules to apply. Missing it means non-English content may be mispronounced.

---

## 6. When to Use / Not Use

**Use Flexbox when:**
- Laying out items in a single direction (horizontal nav, vertical stack)
- Item sizes should be determined by their content
- You need easy vertical centering

**Use Grid when:**
- You need to control rows and columns simultaneously
- Building page-level layouts (header/sidebar/content/footer)
- Creating image galleries or card grids
- Items need to align across rows and columns

**Use semantic elements when:**
- Always — there is almost always a semantic element that matches the content

**Use ARIA when:**
- A custom widget has no native HTML equivalent (e.g., a custom combobox, tabs, tree)
- Native semantics need to be corrected (e.g., `<div role="main">` as a polyfill)
- Dynamic content needs to be announced (`aria-live`)

**Use CSS custom properties when:**
- Building a design system with tokens (colors, spacing, typography)
- Theming (light/dark mode toggle via class swap on `:root`)
- Values needed in JavaScript

---

## 7. Real-World Scenario

### Building a responsive navigation bar with a hamburger menu

```html
<header class="header">
  <a href="/" class="logo">Brand</a>

  <button class="nav-toggle" aria-controls="main-nav" aria-expanded="false">
    <span class="sr-only">Toggle navigation</span>
    <span class="hamburger" aria-hidden="true"></span>
  </button>

  <nav id="main-nav" class="nav" aria-label="Main">
    <ul class="nav__list">
      <li><a href="/" class="nav__link">Home</a></li>
      <li><a href="/about" class="nav__link">About</a></li>
      <li><a href="/contact" class="nav__link">Contact</a></li>
    </ul>
  </nav>
</header>
```

```css
/* Base (mobile) */
.header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 16px;
  height: 60px;
  background: var(--color-surface);
}

.nav {
  display: none;  /* hidden on mobile */
  position: absolute;
  top: 60px;
  left: 0;
  right: 0;
  background: var(--color-surface);
  padding: 16px;
}
.nav.is-open { display: block; }

.nav__list { list-style: none; margin: 0; padding: 0; }
.nav__link { display: block; padding: 12px 0; text-decoration: none; }

/* Hamburger icon */
.hamburger,
.hamburger::before,
.hamburger::after {
  display: block;
  width: 24px;
  height: 2px;
  background: currentColor;
  transition: transform 300ms ease;
}
.hamburger { position: relative; }
.hamburger::before { content: ''; position: absolute; top: -7px; }
.hamburger::after  { content: ''; position: absolute; bottom: -7px; }

/* Open state — X icon */
.nav-toggle[aria-expanded="true"] .hamburger { background: transparent; }
.nav-toggle[aria-expanded="true"] .hamburger::before { transform: rotate(45deg) translate(5px, 5px); }
.nav-toggle[aria-expanded="true"] .hamburger::after  { transform: rotate(-45deg) translate(5px, -5px); }

/* Screen-reader-only utility */
.sr-only {
  position: absolute;
  width: 1px; height: 1px;
  padding: 0; margin: -1px;
  overflow: hidden;
  clip: rect(0 0 0 0);
  white-space: nowrap;
  border: 0;
}

/* Desktop — horizontal nav */
@media (min-width: 768px) {
  .nav-toggle { display: none; }
  .nav { display: block; position: static; padding: 0; }
  .nav__list { display: flex; gap: 24px; }
  .nav__link { padding: 0; }
}
```

```javascript
// Toggle with keyboard support
const toggle = document.querySelector('.nav-toggle');
const nav = document.querySelector('#main-nav');

toggle.addEventListener('click', () => {
  const isOpen = toggle.getAttribute('aria-expanded') === 'true';
  toggle.setAttribute('aria-expanded', String(!isOpen));
  nav.classList.toggle('is-open', !isOpen);
});

// Close on Escape
document.addEventListener('keydown', (e) => {
  if (e.key === 'Escape' && nav.classList.contains('is-open')) {
    nav.classList.remove('is-open');
    toggle.setAttribute('aria-expanded', 'false');
    toggle.focus();
  }
});
```

---

## 8. Interview Questions

**Q1: Explain the CSS box model with `box-sizing: border-box`.**

A: Every CSS element is composed of content, padding, border, and margin layers. With the default `box-sizing: content-box`, `width` applies only to the content area — padding and border are added on top, making the rendered element wider than declared. With `border-box`, `width` includes padding and border, so a `width: 200px` element is exactly 200px wide regardless of padding or border. `border-box` is almost always what you want and should be applied globally.

---

**Q2: How does z-index work and what is a stacking context?**

A: `z-index` controls the stacking order of elements along the Z-axis. It only works on positioned elements (`position` is not `static`). A stacking context is an isolated 3D layer. Elements inside a stacking context are stacked relative to each other and cannot be interleaved with elements in sibling stacking contexts. Stacking contexts are created by: positioned elements with a non-auto z-index, transforms, opacity < 1, filters, will-change, and others. Common bug: a child with `z-index: 9999` still cannot appear above a sibling stacking context if its parent context has a lower z-index than that sibling.

---

**Q3: When would you use Flexbox vs Grid?**

A: Flexbox is one-dimensional — optimal for single-row or single-column layouts where item size is content-driven (nav links, button groups, card rows). Grid is two-dimensional — optimal for layouts that require control over both axes simultaneously (page structure, image galleries, form layouts where labels and inputs must align across rows). In practice, you often use Grid for the page frame and Flexbox within each component.

---

**Q4: What is CSS specificity?**

A: Specificity determines which CSS rule wins when multiple rules target the same element and property. It is calculated as a tuple (inline, IDs, classes/attributes/pseudos, elements). Inline styles win over everything except `!important`. IDs beat classes. Classes beat elements. When specificities are equal, the last declared rule wins (cascade). Common pitfalls: ID selectors make it very hard to override styles; `!important` breaks the cascade and should be avoided except for utility classes.

---

**Q5: How do you center a div vertically and horizontally?**

A: Most modern approach: on the parent, `display: flex; justify-content: center; align-items: center;`. Or with Grid: `display: grid; place-items: center;`. The old approach for absolute-positioned elements: `position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%)` — works without knowing the element's dimensions.

---

**Q6: What is a stacking context and why does z-index sometimes not work?**

A: Answered in Q2. The common symptom: developer sets `z-index: 9999` on a modal backdrop and it still appears behind a tooltip from a different component. The root cause is that the modal's parent element created a stacking context with a lower z-index. Solution: either move the modal higher in the DOM (outside the offending parent), or use `isolation: isolate` strategically, or flatten the stacking context of the parent.

---

**Q7: What is mobile-first CSS and why is it preferred?**

A: Mobile-first means writing base styles for the smallest viewport, then using `min-width` media queries to progressively enhance for larger screens. It is preferred because: base styles are simpler (mobile layouts are usually simpler), CSS specificity is easier to manage (you add complexity, not override it), and it aligns with Progressive Enhancement — the site works on any device by default and gets better with more capability.

---

**Q8: How do CSS animations perform? Which properties are "free"?**

A: The browser rendering pipeline is: JavaScript → Style → Layout → Paint → Composite. Changing properties that affect layout (width, height, margin, top, left) triggers the entire pipeline — expensive. Changing paint-only properties (color, background, box-shadow) skips layout but still triggers paint — moderate cost. Changing only `transform` and `opacity` skips both layout and paint and goes straight to composite (GPU-accelerated) — essentially free at 60fps. Always animate transform/opacity when possible, and use `will-change: transform` to hint the browser to pre-promote an element to its own GPU layer.

---

## 9. Exercises

**Exercise 1 — Responsive nav bar with hamburger menu:**

Build the nav described in section 7 from scratch. Requirements:
- Mobile: hidden nav, visible hamburger button
- Desktop (min-width: 768px): horizontal nav, no hamburger
- Hamburger icon must be drawn with CSS only (no SVG, no image)
- Must use `aria-expanded` on the toggle button
- Must close on Escape key and return focus to the button
- Must be keyboard navigable (Tab through links, no trap on mobile)

---

**Exercise 2 — CSS Grid magazine layout:**

Build a 12-column grid layout with:
- A full-width header
- A 3-column feature section where the first item spans 2 columns and 2 rows
- A 4-column card grid below that wraps responsively (min 200px per card)
- A full-width footer

Use named grid areas for the page-level layout and `auto-fill` + `minmax` for the card grid. No Flexbox allowed (except inside individual cards).

---

**Exercise 3 — Smooth CSS accordion without JavaScript:**

Build an FAQ accordion using only HTML and CSS:
- Each item has a question (clickable) and an answer (initially hidden)
- Clicking a question smoothly reveals the answer
- Only one answer visible at a time
- Uses `<details>` and `<summary>` elements (semantic, keyboard accessible)
- Animate the reveal with `transition` on `max-height` or via the `@starting-style` rule

Hint: `<details>` has a built-in open/closed state managed by the browser. Style `details[open] .answer` for the open state. For smooth animation, CSS `transition` on `max-height` from 0 to a large value works; alternatively, modern browsers support `transition: height allow-discrete` with `@starting-style`.

---

## 10. Further Reading

- **MDN — HTML elements reference**: https://developer.mozilla.org/en-US/docs/Web/HTML/Element
- **MDN — CSS reference**: https://developer.mozilla.org/en-US/docs/Web/CSS/Reference
- **A Complete Guide to Flexbox** (CSS-Tricks): https://css-tricks.com/snippets/css/a-guide-to-flexbox/
- **A Complete Guide to CSS Grid** (CSS-Tricks): https://css-tricks.com/snippets/css/complete-guide-grid/
- **WCAG 2.2** (Web Content Accessibility Guidelines): https://www.w3.org/TR/WCAG22/
- **Inclusive Components** by Heydon Pickering: https://inclusive-components.design/ — accessible UI patterns
- **Every Layout** (Heydon Pickering & Andy Bell): https://every-layout.dev/ — algorithmic CSS layout patterns
- **web.dev — Learn CSS**: https://web.dev/learn/css/ — interactive course covering every aspect of CSS
- **Smashing Magazine — CSS specificity**: https://www.smashingmagazine.com/2007/07/css-specificity-things-you-should-know/
- **Chrome DevTools — Rendering tab**: Enable "Paint flashing" and "Layer borders" to visualize compositing layers and paint triggers
