# Semantic HTML

## Overview

Semantic HTML means using elements for their intended meaning, not just their visual appearance. A `<button>` communicates interactivity. A `<nav>` communicates navigation. A `<h2>` communicates a subsection heading. When you use the right element, browsers, search engines, and assistive technologies all get accurate information about your content — for free, without any JavaScript or ARIA.

The opposite of semantic HTML is "div soup": wrapping everything in `<div>` and `<span>` with classes that describe visual styling but nothing about meaning. Div soup produces pages that look the same but are invisible to screen readers, un-navigable by keyboard, and harder to maintain.

This chapter covers the semantic elements most critical to accessibility: landmarks, heading hierarchy, lists, forms, tables, and media elements.

## Prerequisites

- HTML basics (block vs inline, attributes)
- A working browser with accessibility DevTools (Chrome or Firefox)
- Optional: a screen reader installed (NVDA for Windows, VoiceOver for macOS/iOS)

## Core Concepts

### Landmark Elements

Landmarks give screen reader users a map of the page. Each landmark is a region they can jump to directly, without reading everything in between.

| Element | Role | Purpose |
|---------|------|---------|
| `<header>` | `banner` | Site-wide header (only once per page, unless inside `<article>`) |
| `<nav>` | `navigation` | Navigation menus |
| `<main>` | `main` | Primary content — only one per page |
| `<aside>` | `complementary` | Supplementary content (sidebars, callouts) |
| `<footer>` | `contentinfo` | Site-wide footer |
| `<section>` | `region` | Named section (needs `aria-label` or `aria-labelledby` to create a landmark) |
| `<article>` | `article` | Self-contained content (blog post, comment, card) |
| `<form>` | `form` | Form region (needs accessible name to become a landmark) |

Screen reader users can list all landmarks on a page and jump directly. If you use `<div>` for everything, this navigation collapses.

### Heading Hierarchy

Headings (`<h1>` through `<h6>`) define the document outline. Screen reader users frequently navigate by headings — "jump to next heading" is one of the most common shortcuts.

Rules:
- **One `<h1>` per page** — it should be the primary topic
- **Don't skip levels** — after an `<h2>`, use `<h3>`, not `<h4>`
- **Heading levels reflect nesting, not size** — use CSS to control visual size
- **Every section of content should be reachable via a heading**

```html
<h1>Developer's Guide to Node.js</h1>

<h2>Getting Started</h2>
  <h3>Installation</h3>
  <h3>Your First Script</h3>

<h2>Core Modules</h2>
  <h3>File System</h3>
    <h4>Reading Files</h4>
    <h4>Writing Files</h4>
  <h3>HTTP</h3>
```

### Lists

Use lists when content is enumerable. Lists communicate grouping and count to screen readers ("list of 5 items").

```html
<!-- Unordered list — order doesn't matter -->
<ul>
  <li>TypeScript</li>
  <li>Node.js</li>
  <li>PostgreSQL</li>
</ul>

<!-- Ordered list — sequence matters -->
<ol>
  <li>Install dependencies</li>
  <li>Configure environment</li>
  <li>Run migrations</li>
  <li>Start the server</li>
</ol>

<!-- Description list — term/definition pairs, key/value data -->
<dl>
  <dt>Timeout</dt>
  <dd>Maximum time in milliseconds before the request fails. Default: 5000.</dd>

  <dt>Retries</dt>
  <dd>Number of times to retry a failed request. Default: 3.</dd>
</dl>
```

Navigation menus are also lists:
```html
<nav aria-label="Main navigation">
  <ul>
    <li><a href="/">Home</a></li>
    <li><a href="/docs">Documentation</a></li>
    <li><a href="/about">About</a></li>
  </ul>
</nav>
```

### Forms

Forms are one of the highest-stakes areas for accessibility. A form that isn't accessible can prevent users from signing up, checking out, or submitting support tickets.

**Label association** is the foundation:

```html
<!-- Method 1: for/id (most reliable) -->
<label for="email">Email address</label>
<input id="email" type="email" autocomplete="email">

<!-- Method 2: wrapping label (no id needed) -->
<label>
  Email address
  <input type="email" autocomplete="email">
</label>

<!-- Bad: no label -->
<input type="email" placeholder="Email">

<!-- Bad: label not programmatically associated -->
<span>Email address</span>
<input type="email">
```

**Grouping related inputs:**
```html
<fieldset>
  <legend>Shipping address</legend>

  <label for="street">Street</label>
  <input id="street" type="text" autocomplete="street-address">

  <label for="city">City</label>
  <input id="city" type="text" autocomplete="address-level2">

  <label for="postcode">Postcode</label>
  <input id="postcode" type="text" autocomplete="postal-code">
</fieldset>

<fieldset>
  <legend>Payment method</legend>
  <label>
    <input type="radio" name="payment" value="card"> Credit card
  </label>
  <label>
    <input type="radio" name="payment" value="pix"> PIX
  </label>
</fieldset>
```

**Required fields and error states:**
```html
<label for="username">
  Username
  <span aria-hidden="true"> *</span>
  <span class="sr-only">(required)</span>
</label>
<input
  id="username"
  type="text"
  required
  aria-required="true"
  aria-invalid="true"
  aria-describedby="username-error"
>
<span id="username-error" role="alert">
  Username is required. Use only letters, numbers, and underscores.
</span>
```

### Tables

Tables convey relationships between data across rows and columns. They must be used for tabular data only — not for layout.

```html
<table>
  <caption>Monthly server costs by region (USD)</caption>
  <thead>
    <tr>
      <th scope="col">Region</th>
      <th scope="col">Compute</th>
      <th scope="col">Storage</th>
      <th scope="col">Total</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th scope="row">us-east-1</th>
      <td>$320</td>
      <td>$48</td>
      <td>$368</td>
    </tr>
    <tr>
      <th scope="row">eu-west-1</th>
      <td>$290</td>
      <td>$41</td>
      <td>$331</td>
    </tr>
  </tbody>
  <tfoot>
    <tr>
      <th scope="row">Total</th>
      <td>$610</td>
      <td>$89</td>
      <td>$699</td>
    </tr>
  </tfoot>
</table>
```

Key attributes:
- `<caption>` — table title, read first by screen readers
- `scope="col"` — header applies to the column below
- `scope="row"` — header applies to the row to the right
- `<thead>`, `<tbody>`, `<tfoot>` — structural grouping

### Figure and Figcaption

Use `<figure>` with `<figcaption>` for images, diagrams, code samples, or charts that have a caption.

```html
<!-- Image with caption -->
<figure>
  <img
    src="architecture-diagram.png"
    alt="Three-tier architecture showing web layer, application layer, and database layer"
  >
  <figcaption>Figure 1: High-level architecture of the API service</figcaption>
</figure>

<!-- Code block with caption -->
<figure>
  <figcaption>Listing 1: Basic Fastify server setup</figcaption>
  <pre><code class="language-typescript">
import Fastify from 'fastify';
const app = Fastify({ logger: true });
app.get('/', async () => ({ status: 'ok' }));
await app.listen({ port: 3000 });
  </code></pre>
</figure>

<!-- Chart where alt is empty, caption carries the data summary -->
<figure>
  <img src="revenue-chart.png" alt="">
  <figcaption>
    Revenue grew from $1.2M in Q1 to $1.9M in Q4, a 58% increase.
    <a href="/data/revenue-2024.csv">Download raw data (CSV)</a>
  </figcaption>
</figure>
```

## Hands-On Examples

### Auditing Heading Structure with Playwright

```typescript
import { chromium } from 'playwright';

async function auditHeadings(url: string) {
  const browser = await chromium.launch();
  const page = await browser.newPage();
  await page.goto(url);

  const headings = await page.$$eval(
    'h1, h2, h3, h4, h5, h6',
    (elements) =>
      elements.map((el) => ({
        level: parseInt(el.tagName.slice(1)),
        text: el.textContent?.trim() ?? '',
      }))
  );

  console.log('Heading structure:');
  for (const h of headings) {
    console.log(`${'  '.repeat(h.level - 1)}H${h.level}: ${h.text}`);
  }

  // Detect skipped levels
  for (let i = 1; i < headings.length; i++) {
    const prev = headings[i - 1].level;
    const curr = headings[i].level;
    if (curr > prev + 1) {
      console.warn(`Skipped H${prev} → H${curr} at: "${headings[i].text}"`);
    }
  }

  await browser.close();
}

auditHeadings('http://localhost:3000');
```

### Validating Landmark Presence

```typescript
async function auditLandmarks(url: string) {
  const browser = await chromium.launch();
  const page = await browser.newPage();
  await page.goto(url);

  const landmarks = await page.$$eval(
    'main, nav, header, footer, aside, section[aria-label], section[aria-labelledby]',
    (elements) =>
      elements.map((el) => ({
        tag: el.tagName.toLowerCase(),
        label: el.getAttribute('aria-label') ?? null,
      }))
  );

  console.log('Landmarks found:');
  landmarks.forEach(({ tag, label }) =>
    console.log(`  <${tag}>${label ? ` "${label}"` : ''}`)
  );

  if (!landmarks.some((l) => l.tag === 'main')) {
    console.error('ERROR: no <main> element found');
  }

  const navs = landmarks.filter((l) => l.tag === 'nav');
  if (navs.length > 1 && navs.some((n) => !n.label)) {
    console.warn('WARNING: multiple <nav> elements without aria-label');
  }

  await browser.close();
}
```

### Checking Form Label Association

```typescript
import { test, expect } from '@playwright/test';

test('all form inputs have associated labels', async ({ page }) => {
  await page.goto('/signup');

  // Find inputs that have no label association
  const unlabeled = await page.$$eval(
    'input:not([type="hidden"]):not([type="submit"]):not([type="button"]), select, textarea',
    (inputs) =>
      inputs
        .filter((input) => {
          const id = input.getAttribute('id');
          const ariaLabel = input.getAttribute('aria-label');
          const ariaLabelledby = input.getAttribute('aria-labelledby');
          const wrappingLabel = input.closest('label');
          const labelFor = id ? document.querySelector(`label[for="${id}"]`) : null;
          return !ariaLabel && !ariaLabelledby && !wrappingLabel && !labelFor;
        })
        .map((input) => ({
          type: input.getAttribute('type') ?? input.tagName,
          id: input.getAttribute('id') ?? '(no id)',
          name: input.getAttribute('name') ?? '(no name)',
        }))
  );

  expect(unlabeled).toEqual([]);
});
```

## Common Patterns & Best Practices

### Full Page Template

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Dashboard — Acme App</title>
</head>
<body>

  <a href="#main-content" class="skip-link">Skip to main content</a>

  <header>
    <a href="/"><img src="/logo.svg" alt="Acme App home"></a>
    <nav aria-label="Main navigation">
      <ul>
        <li><a href="/dashboard" aria-current="page">Dashboard</a></li>
        <li><a href="/projects">Projects</a></li>
        <li><a href="/settings">Settings</a></li>
      </ul>
    </nav>
  </header>

  <main id="main-content">
    <h1>Dashboard</h1>

    <section aria-labelledby="recent-heading">
      <h2 id="recent-heading">Recent Projects</h2>
      <!-- project cards as <article> elements -->
    </section>

    <section aria-labelledby="activity-heading">
      <h2 id="activity-heading">Recent Activity</h2>
      <!-- activity list -->
    </section>
  </main>

  <aside aria-labelledby="tips-heading">
    <h2 id="tips-heading">Quick Tips</h2>
    <!-- sidebar tips -->
  </aside>

  <footer>
    <nav aria-label="Footer navigation">
      <ul>
        <li><a href="/privacy">Privacy Policy</a></li>
        <li><a href="/terms">Terms of Service</a></li>
      </ul>
    </nav>
    <p><small>&copy; 2024 Acme Corp.</small></p>
  </footer>

</body>
</html>
```

### Skip Link (CSS)

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

### Visually Hidden Class

```css
/* Readable by AT, invisible on screen */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
```

## Anti-Patterns to Avoid

**`<div>` for interactive elements**
```html
<!-- Bad -->
<div class="btn" onclick="submit()">Submit</div>

<!-- Good -->
<button type="submit">Submit</button>
```

**Tables for layout**
```html
<!-- Bad -->
<table><tr><td><nav>...</nav></td><td><main>...</main></td></tr></table>

<!-- Good: CSS grid or flexbox -->
<div class="layout-grid">
  <nav>...</nav>
  <main>...</main>
</div>
```

**Heading levels chosen for visual size, not hierarchy**
```html
<!-- Bad -->
<h1>Settings</h1>
<h4>Email Preferences</h4>  <!-- skipped h2, h3 -->

<!-- Good: use CSS for font size, HTML for structure -->
<h1>Settings</h1>
<h2 class="subsection-heading">Email Preferences</h2>
```

**Placeholder as the only label**
```html
<!-- Bad: label disappears when user types -->
<input type="email" placeholder="Email address">

<!-- Good: persistent label + helpful placeholder -->
<label for="email">Email address</label>
<input id="email" type="email" placeholder="you@example.com">
```

**Multiple `<main>` elements**
```html
<!-- Bad: only one <main> per page -->
<main>Dashboard content</main>
<main>Sidebar content</main>

<!-- Good: use <aside> for supplementary content -->
<main>Dashboard content</main>
<aside>Sidebar content</aside>
```

## Debugging & Troubleshooting

### Chrome Accessibility Panel

1. Open DevTools → Elements
2. Select any element
3. Click the "Accessibility" tab (right side)
4. See: role, accessible name, accessible description, properties, and the accessibility tree

### Firefox Accessibility Inspector

1. DevTools → Accessibility
2. Toggle "Turn on accessibility features" if needed
3. Hover elements to see their role, name, and state
4. "Check for issues" button runs a quick audit

### Validating Table Headers

```bash
# Check for tables without captions and th without scope
npx axe https://localhost:3000 --rules table-duplicate-name,th-has-data-cells,td-headers-attr
```

### Common axe Rules for Semantics

```bash
npx axe https://localhost:3000 \
  --rules landmark-one-main,page-has-heading-one,region,\
          label,list,listitem,definition-list,dlitem
```

## Real-World Scenarios

### Scenario 1: Card Component

Cards are common in dashboards. A card linking to a detail view needs to be a proper link, not a `<div>` with an onClick.

```tsx
interface ProjectCard {
  id: string;
  name: string;
  status: 'active' | 'paused' | 'completed';
  lastUpdated: string;
}

function ProjectCard({ id, name, status, lastUpdated }: ProjectCard) {
  return (
    <article>
      <h3>
        {/* The heading contains the link — entire heading is clickable */}
        <a href={`/projects/${id}`}>{name}</a>
      </h3>
      <dl>
        <dt>Status</dt>
        <dd>
          {/* Combine icon + text — don't rely on color alone */}
          <span aria-hidden="true">{status === 'active' ? '✓' : '○'}</span>
          {' '}{status}
        </dd>
        <dt>Last updated</dt>
        <dd><time dateTime={lastUpdated}>{lastUpdated}</time></dd>
      </dl>
    </article>
  );
}
```

### Scenario 2: Multi-Step Registration Form

```tsx
function RegistrationForm() {
  const [step, setStep] = useState(1);
  const totalSteps = 3;

  return (
    <form aria-label="Account registration" noValidate>
      <nav aria-label="Registration progress">
        <ol>
          {['Account', 'Profile', 'Review'].map((label, i) => (
            <li
              key={label}
              aria-current={step === i + 1 ? 'step' : undefined}
            >
              {label}
            </li>
          ))}
        </ol>
      </nav>

      <p>
        <span className="sr-only">Step {step} of {totalSteps}: </span>
        {step === 1 && 'Create your account'}
        {step === 2 && 'Set up your profile'}
        {step === 3 && 'Review and confirm'}
      </p>

      {step === 1 && (
        <fieldset>
          <legend>Account credentials</legend>
          <label htmlFor="reg-email">Email</label>
          <input id="reg-email" type="email" autoComplete="email" required />
          <label htmlFor="reg-password">Password</label>
          <input id="reg-password" type="password" autoComplete="new-password" required />
        </fieldset>
      )}

      {/* ... other steps ... */}

      <div>
        {step > 1 && (
          <button type="button" onClick={() => setStep((s) => s - 1)}>
            Back
          </button>
        )}
        {step < totalSteps ? (
          <button type="button" onClick={() => setStep((s) => s + 1)}>
            Next
          </button>
        ) : (
          <button type="submit">Create account</button>
        )}
      </div>
    </form>
  );
}
```

## Further Reading

- [HTML Living Standard — Semantics](https://html.spec.whatwg.org/multipage/semantics.html)
- [WAI-ARIA Authoring Practices — Landmark Regions](https://www.w3.org/WAI/ARIA/apg/practices/landmark-regions/)
- [WebAIM: Semantic Structure](https://webaim.org/techniques/semanticstructure/)
- [W3C Accessible Forms Tutorial](https://www.w3.org/WAI/tutorials/forms/)
- [W3C Accessible Tables Tutorial](https://www.w3.org/WAI/tutorials/tables/)

## Summary

Semantic HTML gives you accessibility for free. The key elements:

- **Landmarks** (`<header>`, `<nav>`, `<main>`, `<aside>`, `<footer>`) — page navigation map for screen reader users; multiple `<nav>` elements need `aria-label` to distinguish them
- **Headings** — document outline; one `<h1>`, never skip levels, use CSS for visual size
- **Lists** — `<ul>`, `<ol>`, `<dl>` communicate grouping and count; navigation menus are lists
- **Forms** — every input needs a label via `for`/`id` or wrapping `<label>`; group related inputs with `<fieldset>`/`<legend>`; mark errors with `aria-invalid` and `aria-describedby`
- **Tables** — for tabular data only; always use `<caption>`, `<thead>`, and `scope` on `<th>` elements
- **Figure/Figcaption** — captions for images, diagrams, and code examples

The single highest-impact change you can make: replace `<div>` wrappers with meaningful landmark elements and ensure every form input has a programmatically associated `<label>`.
