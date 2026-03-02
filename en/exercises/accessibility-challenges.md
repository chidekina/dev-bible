# Accessibility Challenges

> Practical accessibility exercises covering ARIA, keyboard navigation, screen readers, color contrast, and automated testing. Each exercise addresses a real-world accessibility barrier. Stack assumed: HTML5, React, TypeScript, jest-axe, Playwright. Target skill level: beginner to intermediate.

---

## Exercise 1 — Audit a Page with axe-core (Easy)

**Scenario:** Run an automated accessibility audit on an existing page and produce a prioritized remediation report.

**Requirements:**
- Install `axe-core` and run it against a target HTML page (your app's `/dashboard` or a provided static page).
- Capture all violations with their WCAG criteria, impact level, and affected elements.
- Produce a Markdown report listing violations grouped by impact: `critical`, `serious`, `moderate`, `minor`.
- Fix all `critical` violations before moving on.
- Re-run the audit after fixes to verify zero critical violations remain.

**Acceptance Criteria:**
- [ ] The audit script runs with `node scripts/audit-a11y.js <url>` and outputs a Markdown file.
- [ ] The report includes: violation id, WCAG criterion, impact, number of affected nodes, and a fix suggestion.
- [ ] All `critical` violations (e.g., missing alt text, form labels, color contrast failures) are resolved.
- [ ] Re-running after fixes produces zero `critical` or `serious` violations.
- [ ] The audit script is added to `package.json` as `"a11y:audit"` script.

**Hints:**
1. axe-core with Playwright: `const { AxeBuilder } = require('@axe-core/playwright'); const results = await new AxeBuilder({ page }).analyze();`.
2. Filter by impact: `results.violations.filter(v => v.impact === 'critical')`.
3. The most common critical violations: missing `alt` on `<img>`, form inputs without `<label>`, buttons with no accessible name, insufficient color contrast.
4. `axe-core` only catches ~30–40% of accessibility issues — automated audits are a starting point, not a complete audit.

---

## Exercise 2 — Fix Heading Hierarchy (Easy)

**Scenario:** A given HTML page has a broken heading hierarchy — headings skip levels and are used for visual styling rather than document structure. Fix it to produce a logical outline.

**Given HTML (broken hierarchy):**
```html
<h1>Dashboard</h1>
<h3>Recent Orders</h3>    <!-- skips h2 -->
<h5>Order #1042</h5>      <!-- skips h4 -->
<h2>Account Settings</h2>
<h4>Email Preferences</h4> <!-- skips h3 -->
<h2>Help</h2>
<h4>Contact Support</h4>   <!-- skips h3 -->
```

**Requirements:**
- Fix the heading hierarchy so that it follows a logical outline (no skipped levels).
- If a heading was chosen for visual styling (font size), replace the heading tag with a semantic element + CSS class.
- Verify the fixed structure with the axe-core `heading-order` rule.
- Write a brief document outline (bulleted list) that represents the intended page structure.

**Acceptance Criteria:**
- [ ] No `axe` violations for the `heading-order` rule after fixes.
- [ ] Headings do not skip levels (e.g., `h1 → h2 → h3` is valid; `h1 → h3` is not).
- [ ] If a `<h5>` was used for visual sizing only, it is replaced with a `<p class="order-label">` or similar.
- [ ] The page's document outline (visible in browser DevTools → Accessibility → Headings) is logical.
- [ ] Fixing headings does not change the visual appearance (CSS takes care of font sizes independently).

**Hints:**
1. A heading's level should reflect its position in the document hierarchy, not its visual size.
2. To check the outline: open Chrome DevTools → Elements → Accessibility pane. Or use the headingsMap browser extension.
3. If a `<h3>` is used just for a bold subheading style: change it to `<p class="subheading">` and add `font-weight: 600; font-size: 1.1rem` in CSS.
4. The rule: each level should appear only under its parent level. `h1` → `h2` → `h3` is correct. `h1` → `h3` is not.

---

## Exercise 3 — Add ARIA Roles to a Custom Dropdown (Medium)

**Scenario:** A custom-built dropdown menu uses `<div>` and `<span>` elements with no ARIA attributes. Screen readers cannot identify it as a listbox. Add the correct ARIA roles and attributes.

**Given HTML (inaccessible):**
```html
<div class="dropdown" onclick="toggleDropdown()">
  <span class="dropdown-trigger">Select a country</span>
  <div class="dropdown-menu" style="display: none">
    <div class="dropdown-item" onclick="select('us')">United States</div>
    <div class="dropdown-item" onclick="select('br')">Brazil</div>
    <div class="dropdown-item" onclick="select('de')">Germany</div>
  </div>
</div>
```

**Requirements:**
- Add ARIA roles: `combobox`, `listbox`, `option`.
- The trigger element must have `aria-haspopup="listbox"`, `aria-expanded` (toggled on open/close), and `aria-controls` pointing to the listbox id.
- Each option must have `role="option"` and `aria-selected`.
- The selected option must update `aria-selected="true"`; all others `false`.
- Keyboard: `Enter`/`Space` opens the dropdown; `ArrowDown`/`ArrowUp` moves focus; `Enter` selects; `Escape` closes.

**Acceptance Criteria:**
- [ ] `axe-core` reports zero violations on the dropdown after fixes.
- [ ] A screen reader (NVDA/JAWS/VoiceOver) announces the component as "Select a country, combobox, collapsed" on focus.
- [ ] Keyboard navigation works as specified without a mouse.
- [ ] `aria-expanded` is `"true"` when open, `"false"` when closed.
- [ ] The currently focused option has `aria-selected="true"` and is announced by screen readers.

**Hints:**
1. ARIA pattern: follow the WAI-ARIA Authoring Practices Guide (APG) combobox pattern — `role="combobox"` on the trigger, `role="listbox"` on the menu, `role="option"` on items.
2. `aria-controls="my-listbox"` on the trigger + `id="my-listbox"` on the dropdown menu.
3. Focus management: use `tabindex="0"` on the trigger and `tabindex="-1"` on options. Move focus programmatically with `element.focus()` on arrow key events.
4. `aria-activedescendant` on the combobox can point to the currently focused option id — avoids moving DOM focus into the listbox.

---

## Exercise 4 — Build a Keyboard-Navigable Tab List (Medium)

**Scenario:** Build a tab interface that is fully keyboard navigable according to the WAI-ARIA Tabs pattern.

**Requirements:**
- Three tabs: "Profile", "Settings", "Notifications".
- Keyboard behavior: `Tab` moves focus to the tab list; `ArrowRight`/`ArrowLeft` cycles between tabs and activates them; `Home` goes to first tab; `End` goes to last tab.
- Active tab has `aria-selected="true"` and `tabindex="0"`; inactive tabs have `aria-selected="false"` and `tabindex="-1"`.
- Each tab panel is associated with its tab via `aria-controls` / `aria-labelledby`.
- Inactive tab panels have `hidden` attribute (not just `display: none`).

**Acceptance Criteria:**
- [ ] `axe-core` reports zero violations.
- [ ] Pressing `ArrowRight` on the "Profile" tab moves to "Settings" and shows its panel.
- [ ] Pressing `End` moves to "Notifications" regardless of current position.
- [ ] Each `tabpanel` has `role="tabpanel"`, `id`, and `aria-labelledby` pointing to its tab's id.
- [ ] Screen reader announces: "Profile, tab, 1 of 3, selected" on focus.

**Hints:**
1. Tab `role` attributes: `role="tablist"` on the container, `role="tab"` on each tab button, `role="tabpanel"` on each panel.
2. Use actual `<button>` elements for tabs — they have built-in keyboard support and semantics.
3. `tabindex` roving: only the active tab has `tabindex="0"`; all others have `tabindex="-1"`. Update on arrow key press.
4. `aria-labelledby` on the panel: `<div role="tabpanel" id="panel-profile" aria-labelledby="tab-profile">`.

---

## Exercise 5 — Implement a Focus Trap in a Modal (Medium)

**Scenario:** Your modal dialog allows keyboard focus to escape outside the modal. Implement a focus trap so Tab and Shift+Tab cycle only within the modal while it is open.

**Requirements:**
- When the modal opens, focus moves to the first focusable element inside it.
- `Tab` cycles through focusable elements within the modal and wraps from last to first.
- `Shift+Tab` cycles in reverse and wraps from first to last.
- When the modal closes, focus returns to the element that triggered the modal.
- `Escape` closes the modal.

**Acceptance Criteria:**
- [ ] Tabbing past the last focusable element inside the modal moves focus to the first (not to the document body).
- [ ] The trigger button receives focus when the modal is dismissed.
- [ ] Focusable elements are discovered dynamically: `a[href], button:not(:disabled), input, select, textarea, [tabindex]:not([tabindex="-1"])`.
- [ ] The focus trap activates on open and deactivates on close — it must not affect other parts of the page.
- [ ] `axe-core` rule `dialog-name` passes (modal has `aria-labelledby` pointing to its title).

**Hints:**
1. Collect focusable elements: `const focusable = modal.querySelectorAll('a[href], button:not(:disabled), input, ...')`.
2. Listen for `keydown` on the modal: if `Tab` pressed and `document.activeElement === lastFocusable`, call `event.preventDefault(); firstFocusable.focus()`.
3. Store trigger: `const trigger = document.activeElement as HTMLElement` before opening. Restore: `trigger.focus()` on close.
4. Use `aria-modal="true"` on the dialog — this hints to some screen readers that content outside the modal is inert.

---

## Exercise 6 — Add Skip Navigation Links (Easy)

**Scenario:** A page with a long navigation bar requires keyboard users to Tab through all nav links before reaching the main content. Add skip navigation links.

**Requirements:**
- Add a "Skip to main content" link as the very first focusable element on the page.
- The link is visually hidden until it receives focus (shown as a visible overlay when focused).
- Clicking/activating the link moves focus to `<main id="main-content">`.
- Add additional skip links for: "Skip to navigation", "Skip to footer" (for long-scrolling pages).
- All skip links appear in the correct tab order and are announced by screen readers.

**Acceptance Criteria:**
- [ ] "Skip to main content" is the first Tab stop on every page.
- [ ] The link is visually invisible when not focused and becomes visible on focus (no display:none — that removes it from tab order).
- [ ] Activating the link moves keyboard focus to `<main>` (not just scrolls to it).
- [ ] `<main>` has `tabindex="-1"` so it can receive programmatic focus.
- [ ] Screen reader announces "Skip to main content, link" on the first Tab press.

**Hints:**
1. CSS for visually hidden but focusable:
   ```css
   .skip-link { position: absolute; top: -40px; left: 0; }
   .skip-link:focus { top: 0; }
   ```
2. `<main id="main-content" tabindex="-1">` — `tabindex="-1"` allows programmatic focus but excludes it from the tab sequence.
3. The link: `<a href="#main-content" class="skip-link">Skip to main content</a>`.
4. For SPAs: re-implement skip links as React components that manage focus on route changes — the default browser anchor behavior may not work with client-side routing.

---

## Exercise 7 — Fix Color Contrast Issues (Easy)

**Scenario:** A given CSS file contains several color combinations that fail WCAG 2.1 AA contrast ratios. Find and fix all failing combinations.

**Given CSS (contains contrast failures):**
```css
.primary-button { background: #4A90D9; color: #FFFFFF; } /* check */
.secondary-button { background: #F5F5F5; color: #AAAAAA; } /* check */
.error-text { color: #FF6B6B; background: #FFFFFF; } /* check */
.badge { background: #FFD700; color: #FFFFFF; } /* check */
.muted-label { color: #CCCCCC; background: #FFFFFF; } /* check */
```

**Requirements:**
- Calculate the contrast ratio for each color pair (use the WCAG formula or an online tool).
- WCAG 2.1 AA requires: ≥ 4.5:1 for normal text, ≥ 3:1 for large text (18px+ or 14px bold+).
- Fix all failing pairs by adjusting the foreground or background color.
- Ensure the fixed colors still match the design intent (do not change a blue button to red).
- Document the before/after contrast ratios in a comment above each rule.

**Acceptance Criteria:**
- [ ] All color pairs meet WCAG 2.1 AA contrast ratio requirements after fixes.
- [ ] Comments show: `/* contrast: 3.2:1 → FAIL | fixed: 7.1:1 → PASS */` format.
- [ ] Fixed colors are perceptually similar to the originals (hue preserved, just darker/lighter).
- [ ] The fix does not introduce new contrast failures in related components.
- [ ] Use the WebAIM Contrast Checker or `color-contrast` npm package to verify ratios.

**Hints:**
1. Failing pairs: `.secondary-button` (#AAAAAA on #F5F5F5 ≈ 2.3:1 — FAIL), `.badge` (white on gold ≈ 1.9:1 — FAIL), `.muted-label` (#CCC on white ≈ 1.6:1 — FAIL).
2. Fix `.secondary-button`: darken the text to `#767676` (minimum for 4.5:1 on white). On #F5F5F5, you need even darker — try `#595959`.
3. Fix `.badge`: change text from white to dark — `#5C4A00` (dark brown) on gold achieves ~7:1.
4. `color-contrast` npm: `const { hex } = require('color-contrast'); hex('#AAAAAA', '#F5F5F5')` returns the ratio.

---

## Exercise 8 — Write jest-axe Tests for a React Component (Medium)

**Scenario:** Write automated accessibility tests for a `LoginForm` React component using `jest-axe` (or `vitest-axe`).

**Requirements:**
- The `LoginForm` renders: email input, password input, "Remember me" checkbox, and a submit button.
- Write jest-axe tests for: default state, validation error state (email field shows an error), loading state (button disabled).
- Each test renders the component and calls `axe(container)` — assert `toHaveNoViolations()`.
- Beyond axe: write a separate test that manually verifies label associations (each input has a correctly associated `<label>`).

**Acceptance Criteria:**
- [ ] `expect(await axe(container)).toHaveNoViolations()` passes for all three states.
- [ ] The test for the error state renders an error message and verifies `aria-describedby` on the email input points to the error element's id.
- [ ] The `<label>` association test uses `getByLabelText('Email address')` — it passes only if the label is correctly associated.
- [ ] Tests use `@testing-library/react` for rendering — no direct DOM manipulation.
- [ ] All three test cases run in under 1 second combined.

**Hints:**
1. Setup: `import { axe, toHaveNoViolations } from 'jest-axe'; expect.extend(toHaveNoViolations);`.
2. Render and test: `const { container } = render(<LoginForm />); expect(await axe(container)).toHaveNoViolations();`.
3. Error state: render with `<LoginForm errors={{ email: 'Please enter a valid email' }} />`. The input should have `aria-describedby="email-error"` and the error span should have `id="email-error"`.
4. Label association: `getByLabelText('Email address')` uses the `for`/`id` relationship or `aria-label`. If it throws, the label is not correctly associated.

---

## Exercise 9 — Make a Form Fully Accessible (Medium)

**Scenario:** An existing registration form has several accessibility issues: unlabeled inputs, no error messages, missing required indicators, and no live region for async validation feedback.

**Requirements:**
- Every input must have a visible `<label>` (not just placeholder text).
- Required fields must be indicated with `aria-required="true"` and a visible `*` with a legend explaining `* = required`.
- Validation errors must be associated with their input via `aria-describedby`.
- Async validation (e.g., "email already taken") must use `aria-live="polite"` to announce the result to screen readers.
- On form submit with errors, focus must move to the first field with an error.

**Acceptance Criteria:**
- [ ] Every input has an associated `<label>` — `getByLabelText('First name')` succeeds in tests.
- [ ] `aria-required="true"` is on all required inputs; `aria-describedby` links inputs to their error messages.
- [ ] Submitting with errors announces "Please fix 2 errors" via an `aria-live` region.
- [ ] After async email validation, "Email address is already taken" is announced without a page reload.
- [ ] The form passes all axe-core rules in the error state (the hardest state to get right).

**Hints:**
1. Visible label: `<label for="email">Email address <span aria-hidden="true">*</span></label><input id="email" aria-required="true" />`.
2. Error association: `<input id="email" aria-describedby="email-error" aria-invalid="true" />` + `<span id="email-error" role="alert">Invalid email</span>`.
3. Live region for async: `<div aria-live="polite" aria-atomic="true" id="async-feedback"></div>`. Update its `textContent` from JS — screen readers announce the change.
4. Focus on error: `document.querySelector('[aria-invalid="true"]')?.focus()` after failed submit.

---

## Exercise 10 — Screen Reader Walkthrough Checklist (Easy)

**Scenario:** Manually test a page with a screen reader to discover issues that automated tools cannot catch. Document findings and fixes.

**Requirements:**
- Choose a screen reader: NVDA (Windows, free), VoiceOver (macOS/iOS, built-in), or TalkBack (Android).
- Walk through the target page using only the keyboard and screen reader — no mouse.
- Complete the checklist below, marking Pass/Fail for each item with observations.
- Fix at least 3 issues found during the walkthrough.
- Re-test after fixes to confirm resolution.

**Checklist:**
```
[ ] Page title is meaningful and unique (announced on load)
[ ] Landmark regions: header, nav, main, footer are all present
[ ] All images have meaningful alt text (decorative images have alt="")
[ ] Links have descriptive text (no "click here" or "read more" without context)
[ ] Buttons are distinguishable from links (buttons perform actions, links navigate)
[ ] Form inputs all have associated labels
[ ] Error messages are announced automatically (not just visually shown)
[ ] Custom widgets (dropdowns, modals, tabs) are operable by keyboard
[ ] No content is only conveyed by color alone
[ ] Reading order in screen reader matches visual order
```

**Acceptance Criteria:**
- [ ] All 10 checklist items are evaluated (Pass, Fail, or N/A with reasoning).
- [ ] At least 3 Fail items have a documented fix with before/after code snippets.
- [ ] After fixes, those 3 items pass when re-tested with the screen reader.
- [ ] A brief summary (5–10 sentences) describes the overall experience from a screen reader user's perspective.
- [ ] The walkthrough is done on a page you built — not a third-party site.

**Hints:**
1. NVDA shortcuts: `H` to jump between headings, `F` for form fields, `B` for buttons, `L` for lists, `D` for landmarks, `Insert+F7` for a full list of links.
2. VoiceOver on macOS: `Ctrl+Option+U` opens the Web Rotor — navigate by headings, links, form controls.
3. "No content conveyed by color alone": verify error states use an icon or text, not just a red border.
4. Reading order: screen readers follow DOM order, not visual order. CSS `order` and `position: absolute` can cause DOM order vs. visual order mismatches.
