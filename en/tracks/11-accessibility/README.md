# Track 11 — Accessibility

Build interfaces that work for everyone, including users who rely on screen readers, keyboard navigation, or assistive technology. Accessibility is a legal requirement in many jurisdictions and a mark of engineering quality.

**Estimated time:** 1–2 weeks

---

## Topics

1. [WCAG Standards](wcag-standards.md) — levels A/AA/AAA, success criteria, the four principles (POUR)
2. [Semantic HTML](semantic-html.md) — landmark elements, heading hierarchy, form labels, native controls
3. [ARIA Roles](aria-roles.md) — when to use ARIA, roles, states, properties, the first rule of ARIA
4. [Keyboard Navigation](keyboard-navigation.md) — focus management, tab order, focus traps, skip links
5. [Testing Accessibility](testing-accessibility.md) — axe-core, screen reader testing, Lighthouse a11y audit

---

## Prerequisites

- Track 02 — Frontend (HTML, CSS, React)
- Basic understanding of the DOM and browser rendering

---

## What You'll Build

- A fully accessible modal dialog with focus trap, Escape key handling, and correct ARIA attributes
- A custom dropdown component that passes axe-core with zero violations
- A keyboard-navigable data table with sortable columns
- An automated accessibility test suite using axe-core integrated into Playwright E2E tests
