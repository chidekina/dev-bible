# WCAG Standards

## Overview

The Web Content Accessibility Guidelines (WCAG) are the international standard for making web content accessible to people with disabilities. Published by the W3C's Web Accessibility Initiative (WAI), WCAG defines measurable success criteria organized around four principles: Perceivable, Operable, Understandable, and Robust — often remembered as **POUR**.

WCAG 2.1 (published 2018) added 17 new criteria focused on mobile, low vision, and cognitive disabilities. WCAG 2.2 (published 2023) added 9 more, with new focus-visible and accessible authentication criteria. Understanding these standards is essential for shipping software that works for everyone and for meeting legal requirements (ADA in the US, EN 301 549 in Europe, LGPD-adjacent requirements in Brazil).

## Prerequisites

- Basic HTML knowledge (forms, headings, links, images)
- Familiarity with browser DevTools
- Understanding of assistive technologies at a conceptual level (screen readers, keyboard-only navigation)

## Core Concepts

### Conformance Levels

WCAG defines three conformance levels:

| Level | Description | Typical Requirement |
|-------|-------------|---------------------|
| **A** | Minimum accessibility — must be met | Legal baseline in most jurisdictions |
| **AA** | Standard accessibility — should be met | Required by most laws (ADA, EN 301 549) |
| **AAA** | Enhanced accessibility — aspirational | Recommended for specialized audiences |

Most organizations target **AA conformance**. AAA is not required as a blanket target but individual AAA criteria should be met where feasible.

### The POUR Principles

#### 1. Perceivable

Information and UI components must be presentable in ways users can perceive.

Key success criteria:
- **1.1.1 Non-text Content (A)** — All images, icons, and non-text elements have a text alternative
- **1.3.1 Info and Relationships (A)** — Structure conveyed visually is also conveyed programmatically (headings, lists, labels)
- **1.3.3 Sensory Characteristics (A)** — Instructions don't rely solely on shape, color, size, or position ("click the red button" fails)
- **1.4.1 Use of Color (A)** — Color is not the only visual means of conveying information
- **1.4.3 Contrast (Minimum) (AA)** — Text contrast ratio ≥ 4.5:1 (normal), ≥ 3:1 (large text 18pt+ or 14pt+ bold)
- **1.4.4 Resize Text (AA)** — Text can be resized to 200% without loss of content or functionality
- **1.4.10 Reflow (AA, 2.1)** — Content reflows at 320px width without horizontal scrolling
- **1.4.11 Non-text Contrast (AA, 2.1)** — UI components and graphical objects have ≥ 3:1 contrast
- **1.4.12 Text Spacing (AA, 2.1)** — No content or functionality is lost when letter/word/line spacing is increased

#### 2. Operable

UI components and navigation must be operable.

Key success criteria:
- **2.1.1 Keyboard (A)** — All functionality is available from a keyboard
- **2.1.2 No Keyboard Trap (A)** — Users can navigate away from any component using keyboard alone
- **2.4.1 Bypass Blocks (A)** — Mechanism to skip repeated navigation blocks (skip links)
- **2.4.3 Focus Order (A)** — Focus order preserves meaning and operability
- **2.4.4 Link Purpose (A)** — Link purpose is determinable from link text alone (or context)
- **2.4.7 Focus Visible (AA)** — Keyboard focus indicator is visible
- **2.4.11 Focus Appearance (AA, 2.2)** — Focus indicator has sufficient size and contrast
- **2.5.3 Label in Name (A, 2.1)** — Visible label text is contained in the accessible name

#### 3. Understandable

Information and UI operation must be understandable.

Key success criteria:
- **3.1.1 Language of Page (A)** — Default human language of the page is programmatically determined (`lang` attribute)
- **3.2.1 On Focus (A)** — No context change on focus alone
- **3.2.2 On Input (A)** — No context change on input unless user is advised
- **3.3.1 Error Identification (A)** — Input errors are described to the user in text
- **3.3.2 Labels or Instructions (A)** — Labels or instructions provided for user input
- **3.3.7 Redundant Entry (A, 2.2)** — Information already entered is auto-populated or available
- **3.3.8 Accessible Authentication (AA, 2.2)** — No cognitive function test required for authentication (no "type the characters you see")

#### 4. Robust

Content must be robust enough for reliable interpretation by assistive technologies.

Key success criteria:
- **4.1.1 Parsing (A)** — No duplicate IDs, properly nested elements (deprecated in WCAG 2.2 — browsers handle this)
- **4.1.2 Name, Role, Value (A)** — All UI components have name, role, and state determinable by AT
- **4.1.3 Status Messages (AA, 2.1)** — Status messages can be programmatically determined without receiving focus (ARIA live regions)

### Understanding Success Criteria Structure

Each criterion has:
- **Intent** — what problem it solves
- **Benefits** — who benefits
- **Examples** — passing and failing cases
- **Sufficient techniques** — how to meet it (not prescriptive, technique-agnostic)
- **Advisory techniques** — optional enhancements
- **Failures** — documented ways to fail

## Hands-On Examples

### Checking Contrast Ratios

```typescript
// Utility to calculate WCAG contrast ratio between two hex colors
function hexToRgb(hex: string): [number, number, number] {
  const result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex);
  if (!result) throw new Error(`Invalid hex color: ${hex}`);
  return [
    parseInt(result[1], 16),
    parseInt(result[2], 16),
    parseInt(result[3], 16),
  ];
}

function relativeLuminance(r: number, g: number, b: number): number {
  const [rs, gs, bs] = [r, g, b].map((c) => {
    const s = c / 255;
    return s <= 0.03928 ? s / 12.92 : Math.pow((s + 0.055) / 1.055, 2.4);
  });
  return 0.2126 * rs + 0.7152 * gs + 0.0722 * bs;
}

function contrastRatio(hex1: string, hex2: string): number {
  const [r1, g1, b1] = hexToRgb(hex1);
  const [r2, g2, b2] = hexToRgb(hex2);
  const l1 = relativeLuminance(r1, g1, b1);
  const l2 = relativeLuminance(r2, g2, b2);
  const lighter = Math.max(l1, l2);
  const darker = Math.min(l1, l2);
  return (lighter + 0.05) / (darker + 0.05);
}

function meetsWCAG(
  foreground: string,
  background: string,
  level: 'AA' | 'AAA' = 'AA',
  large = false
): boolean {
  const ratio = contrastRatio(foreground, background);
  if (level === 'AAA') return large ? ratio >= 4.5 : ratio >= 7;
  return large ? ratio >= 3 : ratio >= 4.5;
}

// Usage
console.log(contrastRatio('#ffffff', '#767676').toFixed(2)); // ~4.54 — passes AA normal text
console.log(meetsWCAG('#ffffff', '#767676')); // true
console.log(meetsWCAG('#ffffff', '#aaaaaa')); // false — only 2.32:1
```

### Automating Contrast Checks in CI

```typescript
// scripts/check-contrast.ts
import { contrastRatio, meetsWCAG } from './color-utils';

const brandColors = {
  primary: '#1a56db',
  primaryText: '#ffffff',
  secondary: '#6b7280',
  secondaryText: '#ffffff',
  danger: '#dc2626',
  dangerText: '#ffffff',
  warning: '#d97706',
  warningText: '#000000',
};

const pairs: Array<{ name: string; fg: string; bg: string; large?: boolean }> = [
  { name: 'Primary button', fg: brandColors.primaryText, bg: brandColors.primary },
  { name: 'Secondary button', fg: brandColors.secondaryText, bg: brandColors.secondary },
  { name: 'Danger badge', fg: brandColors.dangerText, bg: brandColors.danger },
  { name: 'Warning badge (large)', fg: brandColors.warningText, bg: brandColors.warning, large: true },
];

let failed = false;
for (const { name, fg, bg, large } of pairs) {
  const ratio = contrastRatio(fg, bg).toFixed(2);
  const passes = meetsWCAG(fg, bg, 'AA', large);
  const status = passes ? 'PASS' : 'FAIL';
  console.log(`[${status}] ${name}: ${ratio}:1`);
  if (!passes) failed = true;
}

if (failed) process.exit(1);
```

### Language Declaration

```html
<!-- Root language — always set this -->
<html lang="en">

<!-- Inline language switch for multilingual content -->
<p>
  The French phrase
  <span lang="fr">bonjour le monde</span>
  means "hello world."
</p>
```

### Auditing a Page with Axe

```typescript
// playwright test — see testing-accessibility.md for full setup
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test('homepage meets WCAG 2.1 AA', async ({ page }) => {
  await page.goto('/');
  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
    .analyze();
  expect(results.violations).toEqual([]);
});
```

## Common Patterns & Best Practices

### Building an Accessibility Audit Checklist

Run through these categories on every new page:

**Perceivable**
- [ ] All images have meaningful `alt` text (decorative images use `alt=""`)
- [ ] Color contrast ≥ 4.5:1 for body text, ≥ 3:1 for large text and UI elements
- [ ] No information conveyed by color alone (add icons, patterns, or text labels)
- [ ] Page works when font size is zoomed to 200%
- [ ] Page reflows correctly at 320px wide (no horizontal scroll)

**Operable**
- [ ] All interactive elements reachable and operable with keyboard
- [ ] Visible focus indicator on all focusable elements
- [ ] Skip navigation link present
- [ ] No keyboard traps
- [ ] Links have descriptive text (not "click here" or "read more")

**Understandable**
- [ ] `<html lang="...">` set correctly
- [ ] Form inputs have visible labels (not just placeholder)
- [ ] Error messages identify the field and describe what's wrong
- [ ] Instructions don't rely on sensory cues alone

**Robust**
- [ ] No duplicate IDs on the page
- [ ] All form controls have associated `<label>` elements
- [ ] Interactive elements have correct ARIA roles if not using native HTML
- [ ] Dynamic changes announced via live regions where appropriate

### Prioritizing Issues

Use severity levels when triaging:

| Severity | Example | Action |
|----------|---------|--------|
| Critical | No keyboard access to core flow | Block release |
| Serious | Missing form labels | Fix in current sprint |
| Moderate | Insufficient contrast on secondary text | Fix in next sprint |
| Minor | Missing `lang` on inline phrase | Track in backlog |

## Anti-Patterns to Avoid

**Using color alone to convey state**
```html
<!-- Bad: only color differentiates required from optional -->
<label style="color: red;">Email</label>

<!-- Good: text indicator + color -->
<label>Email <span aria-hidden="true">*</span><span class="sr-only">(required)</span></label>
```

**Suppressing the focus outline without a replacement**
```css
/* Bad: removes focus indicator entirely */
* { outline: none; }

/* Good: replace with a custom visible indicator */
:focus-visible {
  outline: 3px solid #1a56db;
  outline-offset: 2px;
}
```

**Vague link text**
```html
<!-- Bad -->
<a href="/report.pdf">Click here</a>

<!-- Good -->
<a href="/report.pdf">Download annual report (PDF, 2.4 MB)</a>
```

**Placeholder as sole label**
```html
<!-- Bad: when user starts typing, label disappears -->
<input type="email" placeholder="Email address">

<!-- Good -->
<label for="email">Email address</label>
<input id="email" type="email" placeholder="you@example.com">
```

**Images with redundant or missing alt text**
```html
<!-- Bad: redundant -->
<img src="logo.png" alt="Image of our company logo">

<!-- Bad: missing for meaningful image -->
<img src="chart.png" alt="">

<!-- Good: meaningful alt -->
<img src="chart.png" alt="Bar chart showing 40% increase in revenue from Q1 to Q4 2024">

<!-- Good: decorative image -->
<img src="divider.png" alt="" role="presentation">
```

## Debugging & Troubleshooting

### Running Automated Audits

```bash
# Install axe CLI
npm install -g @axe-core/cli

# Audit a URL
axe https://example.com --tags wcag2a,wcag2aa

# Audit with specific rules
axe https://example.com --rules color-contrast,label,image-alt
```

### Using Browser DevTools

**Chrome Accessibility panel:**
1. Open DevTools → Elements
2. Select an element
3. Click the "Accessibility" tab in the panel
4. Inspect the accessibility tree, computed name, role, and properties

**Lighthouse accessibility audit:**
1. DevTools → Lighthouse
2. Check "Accessibility"
3. Generate report — scores each criterion and links to failing elements

### Testing with Keyboard Only

Disconnect your mouse and navigate the entire flow:
1. `Tab` — move to next focusable element
2. `Shift+Tab` — move to previous
3. `Enter` / `Space` — activate buttons and links
4. Arrow keys — navigate within components (menus, tabs, sliders)
5. `Escape` — close dialogs, dismiss overlays

If you can't complete the flow, there's a WCAG 2.1.1 violation.

### Checking the Accessibility Tree

```typescript
// Playwright: inspect computed accessible name and role
test('button has accessible name', async ({ page }) => {
  await page.goto('/dashboard');
  const btn = page.getByRole('button', { name: 'Save changes' });
  await expect(btn).toBeVisible();
  // Playwright uses the accessibility tree, so this already validates AT name
});
```

## Real-World Scenarios

### Scenario 1: Form Validation Errors

When a form submission fails, screen reader users need to know what went wrong. A common mistake is displaying errors visually but not programmatically.

```tsx
// React component with accessible error handling
interface FieldProps {
  id: string;
  label: string;
  error?: string;
  type?: string;
}

function FormField({ id, label, error, type = 'text' }: FieldProps) {
  const errorId = `${id}-error`;
  return (
    <div>
      <label htmlFor={id}>{label}</label>
      <input
        id={id}
        type={type}
        aria-describedby={error ? errorId : undefined}
        aria-invalid={error ? 'true' : undefined}
      />
      {error && (
        <span id={errorId} role="alert" className="error-message">
          {error}
        </span>
      )}
    </div>
  );
}
```

### Scenario 2: Dynamic Content Updates

A dashboard that shows a notification when new data loads:

```tsx
function Dashboard() {
  const [status, setStatus] = useState('');

  async function refreshData() {
    setStatus('Loading data...');
    await fetchData();
    setStatus('Data updated successfully');
  }

  return (
    <div>
      {/* Live region announces dynamic changes to screen readers */}
      <div role="status" aria-live="polite" className="sr-only">
        {status}
      </div>
      <button onClick={refreshData}>Refresh</button>
      {/* ... data table ... */}
    </div>
  );
}
```

### Scenario 3: Responsive Layout for Reflow (1.4.10)

```css
/* Bad: fixed width causes horizontal scroll at 320px */
.card {
  width: 600px;
  padding: 24px;
}

/* Good: fluid layout with no horizontal scroll */
.card {
  width: 100%;
  max-width: 600px;
  padding: clamp(12px, 4vw, 24px);
  box-sizing: border-box;
}
```

## Further Reading

- [WCAG 2.2 Quick Reference](https://www.w3.org/WAI/WCAG22/quickref/) — filterable list of all success criteria
- [Understanding WCAG 2.2](https://www.w3.org/WAI/WCAG22/Understanding/) — intent and techniques for each criterion
- [WebAIM Contrast Checker](https://webaim.org/resources/contrastchecker/) — fast online tool
- [Deque University](https://dequeuniversity.com/) — structured accessibility training
- [A11y Project Checklist](https://www.a11yproject.com/checklist/) — practical implementation checklist
- [axe-core rules](https://github.com/dequelabs/axe-core/blob/develop/doc/rule-descriptions.md) — full list of automated checks

## Summary

WCAG organizes accessibility requirements around four principles — Perceivable, Operable, Understandable, Robust — across three conformance levels (A, AA, AAA). Most organizations and laws require AA conformance. WCAG 2.1 added mobile and cognitive criteria; WCAG 2.2 added better focus and authentication criteria.

Key things to internalize:
- AA contrast ratio is 4.5:1 for normal text, 3:1 for large text and UI components
- Every interactive element must be keyboard accessible
- Color alone cannot convey information
- All form inputs need programmatic labels
- Dynamic content changes need live regions so screen readers announce them
- Automate what you can (axe-core), but also test manually with keyboard and screen readers
