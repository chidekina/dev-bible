# Testing Accessibility

## Overview

Accessibility testing combines automated scanning, manual keyboard testing, and screen reader testing. Automated tools catch roughly 30-40% of WCAG issues — they're fast, cheap, and run in CI. The remaining 60-70% require human judgment: does the focus order make sense? Does this live region announcement make sense in context? Is this label specific enough?

This chapter covers the full testing stack: axe-core with Playwright for integration tests, jest-axe for component tests, Lighthouse CI for monitoring, and a practical screen reader testing workflow.

## Prerequisites

- Node.js 18+, TypeScript
- Playwright installed (`npm install -D @playwright/test`)
- A working application with a dev or test server

## Core Concepts

### Automated vs Manual Testing

| Type | Coverage | Speed | When |
|------|----------|-------|------|
| axe-core (Playwright) | ~30-40% of WCAG | Fast (CI) | Every build |
| jest-axe | Component level | Fast (unit) | On component change |
| Lighthouse CI | Score regression | Medium | Every PR |
| Keyboard testing | Focus order, traps, controls | Manual | Feature completion |
| Screen reader testing | Comprehension, announcements | Manual | Before release |
| User testing with AT users | Real-world comprehension | Slow | Major releases |

### How axe-core Works

axe-core is a JavaScript accessibility engine by Deque. It inspects the live DOM — including CSS rendering, computed accessibility tree, and ARIA states — against a rule set mapped to WCAG success criteria.

Rules are categorized as:
- **Violations** — definite failures (block release)
- **Incomplete** (needs review) — axe couldn't determine pass/fail (review manually)
- **Passes** — confirmed passing checks
- **Inapplicable** — rule doesn't apply to this page

Each violation includes:
- `id` — rule name
- `impact` — `critical`, `serious`, `moderate`, `minor`
- `description` — what's wrong
- `help` — how to fix
- `helpUrl` — link to dequeuniversity.com
- `nodes` — array of affected elements with HTML snippet and fix suggestions

### axe Tag Sets

Filter which WCAG criteria you test by passing tags:

| Tag | Covers |
|-----|--------|
| `wcag2a` | WCAG 2.0 Level A |
| `wcag2aa` | WCAG 2.0 Level AA |
| `wcag21a` | WCAG 2.1 Level A additions |
| `wcag21aa` | WCAG 2.1 Level AA additions |
| `wcag22aa` | WCAG 2.2 Level AA additions |
| `best-practice` | Not WCAG, but recommended |
| `TTv5` | Trusted Tester v5 (US government) |

Recommended standard: `['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa']`

## Hands-On Examples

### Setting Up axe-core with Playwright

```bash
npm install -D @axe-core/playwright @playwright/test
npx playwright install chromium
```

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './tests/accessibility',
  use: {
    baseURL: 'http://localhost:3000',
    // Accessibility testing works best with a real browser
    channel: 'chrome',
  },
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

```typescript
// tests/accessibility/pages.test.ts
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

// Helper to format violations for readable output
function formatViolations(violations: AxeResults['violations']): string {
  return violations
    .map(
      (v) =>
        `[${v.impact?.toUpperCase()}] ${v.id}: ${v.description}\n` +
        v.nodes
          .slice(0, 2)
          .map((n) => `  → ${n.html.slice(0, 100)}`)
          .join('\n')
    )
    .join('\n\n');
}

const pages = [
  { name: 'Home', path: '/' },
  { name: 'Login', path: '/login' },
  { name: 'Dashboard', path: '/dashboard' },
  { name: 'Settings', path: '/settings' },
];

for (const { name, path } of pages) {
  test(`${name} page: no WCAG 2.1 AA violations`, async ({ page }) => {
    await page.goto(path);

    const results = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
      // Exclude third-party widgets you can't control
      .exclude('#intercom-iframe')
      .analyze();

    expect(results.violations, formatViolations(results.violations)).toEqual([]);
  });
}
```

### Testing Specific Components

```typescript
// tests/accessibility/components.test.ts
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test('modal dialog meets WCAG', async ({ page }) => {
  await page.goto('/');
  await page.click('button:has-text("Open dialog")');

  // Wait for dialog to appear
  await page.waitForSelector('[role="dialog"]');

  // Only scan the dialog component, not the whole page
  const results = await new AxeBuilder({ page })
    .include('[role="dialog"]')
    .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
    .analyze();

  expect(results.violations).toEqual([]);
});

test('form has no label violations', async ({ page }) => {
  await page.goto('/signup');

  const results = await new AxeBuilder({ page })
    .include('form')
    .withRules(['label', 'label-content-name-mismatch', 'form-field-multiple-labels'])
    .analyze();

  expect(results.violations).toEqual([]);
});

test('data table is accessible', async ({ page }) => {
  await page.goto('/reports');

  const results = await new AxeBuilder({ page })
    .include('table')
    .withRules([
      'table-duplicate-name',
      'td-headers-attr',
      'th-has-data-cells',
      'scope-attr-valid',
    ])
    .analyze();

  expect(results.violations).toEqual([]);
});
```

### jest-axe for Component Unit Tests

```bash
npm install -D jest-axe @testing-library/react @testing-library/jest-dom
```

```typescript
// src/components/Button.test.tsx
import { render } from '@testing-library/react';
import { axe, toHaveNoViolations } from 'jest-axe';
import { Button } from './Button';

expect.extend(toHaveNoViolations);

describe('Button accessibility', () => {
  it('has no violations when rendered with text', async () => {
    const { container } = render(<Button onClick={() => {}}>Save changes</Button>);
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });

  it('has no violations when rendered as icon button', async () => {
    const { container } = render(
      <Button onClick={() => {}} aria-label="Delete item">
        <TrashIcon aria-hidden />
      </Button>
    );
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });

  it('has no violations when disabled', async () => {
    const { container } = render(
      <Button disabled>Cannot submit</Button>
    );
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });
});
```

```typescript
// src/components/FormField.test.tsx
import { render } from '@testing-library/react';
import { axe, toHaveNoViolations } from 'jest-axe';
import { FormField } from './FormField';

expect.extend(toHaveNoViolations);

describe('FormField accessibility', () => {
  it('passes with label and no error', async () => {
    const { container } = render(
      <FormField id="email" label="Email address" type="email" />
    );
    expect(await axe(container)).toHaveNoViolations();
  });

  it('passes with error state', async () => {
    const { container } = render(
      <FormField
        id="email"
        label="Email address"
        type="email"
        error="Please enter a valid email address"
      />
    );
    expect(await axe(container)).toHaveNoViolations();
  });

  it('passes with required indicator', async () => {
    const { container } = render(
      <FormField id="name" label="Full name" required />
    );
    expect(await axe(container)).toHaveNoViolations();
  });
});
```

### Custom axe Reporter for CI

```typescript
// scripts/a11y-report.ts
import { chromium } from 'playwright';
import AxeBuilder from '@axe-core/playwright';
import { writeFileSync } from 'fs';

interface Report {
  url: string;
  violations: number;
  serious: number;
  critical: number;
  details: Array<{
    id: string;
    impact: string;
    description: string;
    affectedCount: number;
  }>;
}

async function auditPages(urls: string[]): Promise<Report[]> {
  const browser = await chromium.launch();
  const reports: Report[] = [];

  for (const url of urls) {
    const page = await browser.newPage();
    await page.goto(url, { waitUntil: 'networkidle' });

    const results = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
      .analyze();

    reports.push({
      url,
      violations: results.violations.length,
      serious: results.violations.filter((v) => v.impact === 'serious').length,
      critical: results.violations.filter((v) => v.impact === 'critical').length,
      details: results.violations.map((v) => ({
        id: v.id,
        impact: v.impact ?? 'unknown',
        description: v.description,
        affectedCount: v.nodes.length,
      })),
    });

    await page.close();
  }

  await browser.close();
  return reports;
}

async function main() {
  const urls = [
    'http://localhost:3000/',
    'http://localhost:3000/login',
    'http://localhost:3000/dashboard',
  ];

  const reports = await auditPages(urls);
  writeFileSync('a11y-report.json', JSON.stringify(reports, null, 2));

  let hasErrors = false;
  for (const report of reports) {
    console.log(`\n${report.url}`);
    console.log(`  Violations: ${report.violations} (${report.critical} critical, ${report.serious} serious)`);
    for (const v of report.details) {
      const marker = v.impact === 'critical' || v.impact === 'serious' ? '[FAIL]' : '[WARN]';
      console.log(`  ${marker} ${v.id} — ${v.affectedCount} element(s): ${v.description}`);
      if (v.impact === 'critical' || v.impact === 'serious') hasErrors = true;
    }
  }

  if (hasErrors) {
    console.error('\nCritical or serious violations found. Failing CI.');
    process.exit(1);
  }
}

main();
```

### Lighthouse CI Integration

```bash
npm install -D @lhci/cli
```

```javascript
// lighthouserc.js
module.exports = {
  ci: {
    collect: {
      url: ['http://localhost:3000/', 'http://localhost:3000/login'],
      numberOfRuns: 1,
      settings: {
        onlyCategories: ['accessibility'],
      },
    },
    assert: {
      assertions: {
        'categories:accessibility': ['error', { minScore: 0.9 }],
        // Fail on specific rules
        'audits[color-contrast].score': ['error', { minScore: 1 }],
        'audits[document-title].score': ['error', { minScore: 1 }],
        'audits[html-has-lang].score': ['error', { minScore: 1 }],
        'audits[image-alt].score': ['error', { minScore: 1 }],
        'audits[label].score': ['error', { minScore: 1 }],
      },
    },
    upload: {
      target: 'temporary-public-storage',
    },
  },
};
```

```yaml
# .github/workflows/a11y.yml
name: Accessibility

on: [push, pull_request]

jobs:
  a11y:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npm run build
      - name: Start server
        run: npm start &
      - name: Wait for server
        run: npx wait-on http://localhost:3000
      - name: Run axe tests
        run: npx playwright test tests/accessibility/
      - name: Lighthouse CI
        run: npx lhci autorun
```

## Common Patterns & Best Practices

### Testing Dynamic Content

```typescript
test('error summary announces to screen readers', async ({ page }) => {
  await page.goto('/signup');

  // Submit with empty required fields
  await page.click('button[type="submit"]');

  // Wait for error state
  await page.waitForSelector('[role="alert"]');

  // Check the live region has content
  const alertText = await page.textContent('[role="alert"]');
  expect(alertText).toBeTruthy();
  expect(alertText).toContain('error');

  // Re-run axe after dynamic state change
  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
    .analyze();

  expect(results.violations).toEqual([]);
});
```

### Testing at Different Viewport Sizes

```typescript
test('navigation is accessible on mobile viewport', async ({ page }) => {
  await page.setViewportSize({ width: 375, height: 812 });
  await page.goto('/');

  // Mobile nav often has hamburger menus — test the toggle
  const menuBtn = page.getByRole('button', { name: /menu/i });
  await menuBtn.click();
  await page.waitForSelector('[role="navigation"][aria-expanded="true"]');

  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
    .analyze();

  expect(results.violations).toEqual([]);
});
```

### Snapshot Testing for Accessibility Tree

```typescript
test('accessibility tree snapshot for card component', async ({ page }) => {
  await page.goto('/');
  const card = page.locator('.project-card').first();
  const snapshot = await card.evaluateHandle((el) =>
    (window as any).axe.utils.getNodeFromTree(el)
  );
  // Use Playwright's accessibility snapshot
  const a11ySnapshot = await page.accessibility.snapshot({
    root: await card.elementHandle() ?? undefined,
  });
  expect(a11ySnapshot).toMatchSnapshot();
});
```

## Anti-Patterns to Avoid

**Running axe on only the homepage**
Accessibility issues often appear on specific pages (forms, dialogs, data tables). Test every unique page type and every dynamic state (error, loading, empty, filled).

**Ignoring `incomplete` results**
axe marks some checks as `incomplete` when it can't determine pass/fail automatically (color contrast over images, dynamic content). Review these manually — they often contain real issues.

**Only running automated tests**
```typescript
// Not enough — axe misses:
// - focus order logic issues
// - live region announcement content
// - screen reader reading order
// - cognitive accessibility

// Supplement with manual checklist on every feature
```

**Disabling rules that are inconvenient**
```typescript
// Bad: silencing a real issue
const results = await new AxeBuilder({ page })
  .disableRules(['color-contrast'])  // "we'll fix this later"
  .analyze();

// Good: fix the issue, or document why it's excluded with a time-boxed plan
```

**Testing only in one browser**
Accessibility tree behavior varies across browsers. Test in Chromium (axe default) AND Firefox:

```typescript
// playwright.config.ts
projects: [
  { name: 'chromium', use: { browserName: 'chromium' } },
  { name: 'firefox',  use: { browserName: 'firefox' } },
],
```

## Debugging & Troubleshooting

### Interpreting axe Violations

```typescript
// Log full violation details for debugging
const results = await new AxeBuilder({ page }).analyze();

for (const violation of results.violations) {
  console.log(`\n--- ${violation.id} (${violation.impact}) ---`);
  console.log('Issue:', violation.description);
  console.log('Fix:', violation.help);
  console.log('WCAG:', violation.tags.filter((t) => t.startsWith('wcag')).join(', '));
  console.log('More info:', violation.helpUrl);
  console.log('Affected elements:');
  for (const node of violation.nodes) {
    console.log('  HTML:', node.html);
    if (node.failureSummary) console.log('  Failure:', node.failureSummary);
    if (node.any.length) {
      console.log('  To fix, satisfy one of:');
      node.any.forEach((c) => console.log(`    - ${c.message}`));
    }
    if (node.all.length) {
      console.log('  To fix, satisfy all of:');
      node.all.forEach((c) => console.log(`    - ${c.message}`));
    }
  }
}
```

### False Positives and Exclusions

```typescript
// Documented exclusion with reason and tracking issue
const results = await new AxeBuilder({ page })
  .exclude('#stripe-card-element')  // third-party iframe, no control — tracked in JIRA-123
  .analyze();
```

### When axe Passes but Screen Reader Fails

Scenarios axe cannot catch:
1. **Reading order** — elements in correct DOM order but visually reordered with CSS
2. **Live region content quality** — region exists but announces confusing text
3. **Focus management** — modal opens but focus doesn't move (focus is not a DOM attribute)
4. **Cognitive load** — too many options, unclear instructions, complex language

For these, do manual testing.

## Real-World Scenarios

### Scenario 1: Screen Reader Testing Workflow (NVDA + Chrome on Windows)

**Setup:**
1. Install NVDA (free, nvaccess.org)
2. Install Chrome
3. Set NVDA to use "Browse mode" for web content (default)

**Navigation shortcuts:**
| Key | Action |
|-----|--------|
| `H` | Next heading |
| `Shift+H` | Previous heading |
| `B` | Next button |
| `F` | Next form field |
| `T` | Next table |
| `L` | Next list |
| `D` | Next landmark |
| `Insert+F7` | List all links/headings/landmarks |
| `Insert+F5` | List all form fields |
| `Tab` | Next interactive element |
| `Enter` | Activate link/button |
| `Space` | Activate button/checkbox |
| `Arrow keys` | Navigate within widget or read line by line |

**Test checklist:**
- [ ] Page title is read on load
- [ ] Landmark navigation makes sense (list landmarks with `Insert+F7`)
- [ ] Heading outline is logical (list headings)
- [ ] All images have meaningful text alternatives
- [ ] Form labels are read when input receives focus
- [ ] Error messages are announced (either via live region or focus move)
- [ ] Dialog traps focus and can be dismissed
- [ ] Dynamic content changes are announced

**macOS VoiceOver (Safari):**
- `Cmd+F5` — toggle VoiceOver
- `VO+U` — open rotor (choose headings, links, landmarks, form controls)
- `VO+Space` — activate element
- `VO+Shift+M` — shortcut menu for element

### Scenario 2: Integrating into PR Review Process

```markdown
## Accessibility checklist for PR reviewers

### Automated (blocked by CI)
- [ ] axe-core Playwright tests pass
- [ ] jest-axe component tests pass
- [ ] Lighthouse accessibility score ≥ 90

### Manual (reviewer responsibility)
- [ ] All new interactive elements are keyboard accessible
- [ ] New forms have associated labels
- [ ] Dynamic content changes use live regions or focus management
- [ ] Modal/dialog components trap focus and restore on close
- [ ] No new color-only information conveyance
- [ ] New images have appropriate alt text
```

### Scenario 3: Full Test File for a Feature Page

```typescript
// tests/accessibility/signup.test.ts
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test.describe('Signup page accessibility', () => {
  test('initial page state passes WCAG 2.1 AA', async ({ page }) => {
    await page.goto('/signup');
    const results = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
      .analyze();
    expect(results.violations).toEqual([]);
  });

  test('form error state passes WCAG 2.1 AA', async ({ page }) => {
    await page.goto('/signup');
    await page.click('button[type="submit"]');
    await page.waitForSelector('[aria-invalid="true"]');

    const results = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
      .analyze();
    expect(results.violations).toEqual([]);
  });

  test('all form inputs have labels', async ({ page }) => {
    await page.goto('/signup');
    const results = await new AxeBuilder({ page })
      .include('form')
      .withRules(['label', 'label-content-name-mismatch'])
      .analyze();
    expect(results.violations).toEqual([]);
  });

  test('submit button is keyboard accessible', async ({ page }) => {
    await page.goto('/signup');
    await page.keyboard.press('Tab');
    // Tab through to submit button
    let focused = '';
    for (let i = 0; i < 20; i++) {
      focused = await page.evaluate(() => document.activeElement?.textContent?.trim() ?? '');
      if (focused === 'Create account') break;
      await page.keyboard.press('Tab');
    }
    expect(focused).toBe('Create account');
    // Activate with keyboard
    await page.keyboard.press('Enter');
    // Verify form submission attempted
    await page.waitForSelector('[aria-invalid="true"]'); // validation ran
  });

  test('error messages are associated with inputs', async ({ page }) => {
    await page.goto('/signup');
    await page.click('button[type="submit"]');
    await page.waitForSelector('[aria-invalid="true"]');

    // Each invalid input should have aria-describedby pointing to error
    const invalids = await page.$$('[aria-invalid="true"]');
    for (const input of invalids) {
      const describedby = await input.getAttribute('aria-describedby');
      expect(describedby).toBeTruthy();
      const errorEl = page.locator(`#${describedby}`);
      await expect(errorEl).toBeVisible();
      const text = await errorEl.textContent();
      expect(text?.length).toBeGreaterThan(0);
    }
  });
});
```

## Further Reading

- [axe-core GitHub](https://github.com/dequelabs/axe-core) — rules documentation and contributing
- [@axe-core/playwright](https://github.com/dequelabs/axe-core-npm/tree/develop/packages/playwright) — Playwright integration
- [jest-axe](https://github.com/nickcolley/jest-axe) — Jest/Vitest integration
- [Lighthouse Accessibility Audits](https://developer.chrome.com/docs/lighthouse/accessibility/) — what each Lighthouse check covers
- [Deque University](https://dequeuniversity.com/) — screen reader training
- [NVDA User Guide](https://www.nvaccess.org/files/nvda/documentation/userGuide.html)

## Summary

An accessibility testing strategy needs all three layers:

1. **Automated (CI gate)** — axe-core with Playwright on every page and dynamic state; jest-axe on components; Lighthouse CI for regression tracking. Catches ~35% of WCAG issues but runs on every PR.

2. **Keyboard testing** — Tab through every user flow without a mouse. Verify focus management for modals, SPAs, and dynamic content. Do this on every new feature.

3. **Screen reader testing** — NVDA+Chrome on Windows or VoiceOver+Safari on macOS. Test the scenarios that matter: form submission with errors, modal dialogs, dynamic search results, live region announcements. Do this before release.

Key setup reminders:
- Install `@axe-core/playwright` and run with `withTags(['wcag2a','wcag2aa','wcag21a','wcag21aa'])`
- In CI: fail on `critical` and `serious` violations; warn on `moderate`
- Never disable rules to make tests pass — fix the underlying issue
- Document exclusions (third-party iframes) with the reason and a tracking issue
