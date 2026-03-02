# Frontend Challenges

> Practical frontend challenges covering React component design, CSS layout, vanilla JavaScript, performance optimization, and accessibility. Each challenge reflects a real-world scenario you will encounter on the job. Skill level: beginner to advanced. Stack context: React 18+, TypeScript, modern CSS (no framework assumed unless stated).

---

## Exercise 1 — Infinite Scroll Feed (Medium)

**Goal:** Build a post feed that loads more items as the user scrolls to the bottom of the page — no "Load More" button.

**Requirements:**
- Fetch the first page of posts on mount from a mock API (you can use JSONPlaceholder or a local mock).
- When the user scrolls within 200px of the bottom, fetch the next page.
- Show a loading spinner while fetching.
- Stop fetching when the API returns an empty page.
- Each post card shows: title, body (truncated to 2 lines), and author name.

**Acceptance Criteria:**
- [ ] No duplicate posts appear on re-renders.
- [ ] Fetches do not overlap (no concurrent fetches for the same page).
- [ ] The spinner appears only during active fetching.
- [ ] No memory leaks — scroll listener is cleaned up on unmount.
- [ ] Works correctly when the user scrolls very fast.

**Hints:**
1. Use `IntersectionObserver` on a sentinel `<div>` at the bottom of the list — far more reliable than `scroll` event listeners.
2. Track a `isFetching` ref (not state) to prevent duplicate in-flight requests.
3. Store posts in a `useReducer` or append-only state with functional updates: `setPosts(prev => [...prev, ...newItems])`.
4. Separate the data-fetching logic into a `usePaginatedFeed` custom hook.

---

## Exercise 2 — Debounced Search with Autocomplete (Medium)

**Goal:** Build a search input that queries an API as the user types, shows suggestions in a dropdown, and handles keyboard navigation.

**Requirements:**
- Debounce the API call by 300ms after the last keystroke.
- Show up to 8 suggestions in a dropdown list below the input.
- Highlight the matching portion of each suggestion (e.g., bold the typed substring).
- Allow keyboard navigation: `ArrowDown`/`ArrowUp` to move through items, `Enter` to select, `Escape` to close.
- Clicking a suggestion fills the input and closes the dropdown.

**Acceptance Criteria:**
- [ ] No API calls fire for partial keystrokes within the 300ms window.
- [ ] Previous requests are cancelled (AbortController) when a new one starts.
- [ ] The dropdown closes when focus leaves the component.
- [ ] ARIA attributes: `role="combobox"`, `aria-expanded`, `aria-activedescendant`, `role="listbox"`.
- [ ] Works entirely with keyboard — mouse is optional.

**Hints:**
1. Use `useRef` to store the `AbortController` and cancel on each new fetch.
2. Use `useEffect` with the debounce — clear the timer on cleanup.
3. Wrap the entire component in a `<div>` with `onBlur` that checks `e.relatedTarget` is still inside.
4. The highlighted match: split the suggestion on the query string, wrap the match in `<strong>`.

---

## Exercise 3 — Drag-and-Drop Kanban Board (Hard)

**Goal:** Implement a Kanban board with three columns (To Do, In Progress, Done). Cards can be dragged between columns and reordered within a column.

**Requirements:**
- At least 3 columns and 5 initial cards.
- Drag a card to a different column to move it.
- Drag a card within a column to reorder it.
- Show a placeholder where the card will be dropped.
- Persist the board state to `localStorage` so it survives a page refresh.

**Acceptance Criteria:**
- [ ] Cards render in correct order after reorder.
- [ ] No state inconsistency between columns after a drag across columns.
- [ ] Placeholder element has the same height as the dragged card.
- [ ] Works on touch screens (pointer events, not only mouse events).
- [ ] Board state is loaded from `localStorage` on mount.

**Hints:**
1. Use the HTML5 Drag and Drop API (`draggable`, `onDragStart`, `onDragOver`, `onDrop`) for a library-free solution.
2. Alternatively, the `@dnd-kit/core` library handles pointer events, accessibility, and touch automatically.
3. Store state as `{ columns: { [id]: { title, cardIds[] } }, cards: { [id]: { title, body } } }` — a normalized shape.
4. Use `useReducer` for complex board state transitions (MOVE_CARD, REORDER_CARD actions).

---

## Exercise 4 — CSS Holy Grail Layout (Easy)

**Goal:** Implement the classic "holy grail" layout using only CSS — no JavaScript, no frameworks, no grid library.

**Requirements:**
- Full-page layout: header (fixed height), three-column middle section (left sidebar, main content, right sidebar), footer.
- The main content area takes all remaining horizontal space.
- Both sidebars have fixed widths (200px each).
- The middle section fills all vertical space between header and footer.
- Responsive: on screens below 768px, stack the three columns vertically (sidebar → content → sidebar).

**Acceptance Criteria:**
- [ ] Layout achieved with CSS Grid or Flexbox (your choice — solve with both, separately).
- [ ] No JavaScript used.
- [ ] The footer sticks to the bottom even when content is short.
- [ ] Responsive breakpoint collapses columns correctly.
- [ ] Works in Chrome, Firefox, and Safari (no vendor prefixes needed for modern CSS).

**Hints:**
1. Grid solution: `grid-template-areas` with `"header header header"`, `"sidebar main aside"`, `"footer footer footer"`.
2. Flexbox solution: wrap the three columns in a flex row; give main `flex: 1`; make the outer container a column flex with `min-height: 100vh`.
3. For sticky footer without JS: `body { display: flex; flex-direction: column; min-height: 100vh }` and `footer { margin-top: auto }`.

---

## Exercise 5 — Accessible Modal Dialog (Medium)

**Goal:** Build a reusable modal dialog component that is fully accessible and follows the ARIA dialog pattern.

**Requirements:**
- Triggered by a "Open Modal" button.
- Contains a title, body content, and Close button.
- Pressing `Escape` closes the modal.
- When the modal opens, focus moves to the first focusable element inside it.
- When the modal closes, focus returns to the element that opened it.
- Focus is trapped inside the modal while it is open — Tab cannot leave.
- Background content is inert while the modal is open.

**Acceptance Criteria:**
- [ ] `role="dialog"`, `aria-modal="true"`, `aria-labelledby` pointing to the title.
- [ ] Focus trap works for all focusable elements: buttons, inputs, links, select, textarea.
- [ ] Screen reader announces the modal title when it opens.
- [ ] `Shift+Tab` wraps focus in reverse.
- [ ] `document.body` gets `aria-hidden="true"` (or `inert` attribute) while modal is open.

**Hints:**
1. Collect focusable elements with `querySelectorAll('button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])')`.
2. Attach a `keydown` listener on the modal: intercept `Tab` and `Shift+Tab`, cycle through the focusable list manually.
3. Use `useRef` to store the trigger element (`document.activeElement`) before opening, so you can restore focus on close.
4. The modern approach: the HTML `<dialog>` element with `showModal()` handles most of this natively — implement with `<dialog>` and then without it for the learning experience.

---

## Exercise 6 — Virtualized Long List (Hard)

**Goal:** Render a list of 100,000 items without freezing the browser, using windowed/virtual rendering.

**Requirements:**
- The list has 100,000 rows, each 50px tall.
- Only the rows visible in the viewport (plus a small overscan buffer) are in the DOM.
- Scrolling must feel smooth — no jank.
- Each row displays: index, a random color swatch, and a label.
- Total DOM node count must stay below 100 at all times.

**Acceptance Criteria:**
- [ ] Initial render takes less than 100ms (measure with Lighthouse or React DevTools).
- [ ] Scroll performance: no visible dropped frames on a mid-range device.
- [ ] The scrollbar thumb reflects the full 100,000-item height.
- [ ] Rows are recycled (not unmounted/remounted on every scroll event).

**Hints:**
1. Use `react-virtual` or `@tanstack/react-virtual` for the windowing logic — this is the industry standard for custom lists.
2. DIY approach: place a tall outer container (`height = count * rowHeight`). An inner container is positioned `absolute` at `top = startIndex * rowHeight`. Only render `startIndex` to `endIndex` rows.
3. Listen to `onScroll` on the outer container to update `scrollTop`, derive `startIndex = Math.floor(scrollTop / rowHeight)`.
4. Add an overscan of 3-5 rows above and below the visible window to prevent blank flashes.

---

## Exercise 7 — Form with Complex Validation (Medium)

**Goal:** Build a multi-step registration form (3 steps) with real-time validation and error messages.

**Requirements:**
- Step 1: Name (required, min 2 chars), Email (valid format), Password (min 8 chars, must contain a number and a special character).
- Step 2: Date of Birth (must be 18+), Country (select from list), Phone (optional, but if provided must match `+[country code] [number]` format).
- Step 3: Review screen showing all entered values before final submit.
- Errors show inline, below each field, on blur.
- The "Next" button is disabled until all required fields on the current step are valid.

**Acceptance Criteria:**
- [ ] Validation triggers on blur, not on every keystroke.
- [ ] Password strength indicator (weak/medium/strong) updates on every keystroke.
- [ ] Back navigation preserves values entered in previous steps.
- [ ] `aria-describedby` links each input to its error message.
- [ ] Submit shows a success state (no backend needed — mock it with a 1-second delay).

**Hints:**
1. Use `react-hook-form` for form state management — it minimizes re-renders and integrates with Zod or Yup for schema validation.
2. Store all step values in a shared context or lifted state so Back navigation restores them.
3. Password strength: score based on length, uppercase, number, special character — sum up and map to weak/medium/strong.
4. Country phone prefix map: keep a small JSON object `{ US: "+1", BR: "+55", ... }` to validate the prefix.

---

## Exercise 8 — Dark Mode Toggle with System Preference (Easy)

**Goal:** Implement a dark/light mode toggle that respects the user's OS preference and persists the choice across sessions.

**Requirements:**
- On first visit, read `prefers-color-scheme` media query to set the initial mode.
- A toggle button switches between light and dark.
- The chosen mode is saved to `localStorage` and restored on reload.
- Mode applies via a CSS class on `<body>` (e.g., `.dark`). All colors are CSS custom properties.
- The toggle button icon changes (sun/moon) and has a proper accessible label.

**Acceptance Criteria:**
- [ ] No flash of wrong theme on page load (hint: set the class before React hydrates).
- [ ] `localStorage` value takes precedence over the media query on subsequent visits.
- [ ] `aria-label` on the toggle button reflects the current action ("Switch to dark mode" or "Switch to light mode").
- [ ] At least 5 custom properties (`--bg`, `--text`, `--accent`, `--border`, `--card-bg`) defined for both themes.

**Hints:**
1. Add an inline `<script>` in the `<head>` (before React bundle) to read `localStorage` and set the class synchronously — this prevents the theme flash.
2. In React: use `useEffect` to sync the class and `localStorage` whenever the theme state changes.
3. `window.matchMedia('(prefers-color-scheme: dark)').addEventListener('change', ...)` lets you react to OS changes in real time.

---

## Exercise 9 — Optimizing a Slow React Dashboard (Medium)

**Goal:** You inherit a dashboard component that is extremely slow to interact with. Profile it and apply the correct optimizations.

**Starting code (intentionally broken):**
```tsx
// Dashboard.tsx — do NOT keep this code as-is
export function Dashboard({ users }: { users: User[] }) {
  const [filter, setFilter] = useState('');
  const [sortKey, setSortKey] = useState<keyof User>('name');

  const filtered = users
    .filter(u => u.name.toLowerCase().includes(filter.toLowerCase()))
    .sort((a, b) => a[sortKey] > b[sortKey] ? 1 : -1);

  return (
    <>
      <input value={filter} onChange={e => setFilter(e.target.value)} />
      <select value={sortKey} onChange={e => setSortKey(e.target.value as keyof User)}>
        <option value="name">Name</option>
        <option value="email">Email</option>
      </select>
      {filtered.map(u => <UserCard key={u.id} user={u} onDelete={() => deleteUser(u.id)} />)}
    </>
  );
}
```

**Requirements:**
- `users` array contains 10,000 items.
- `UserCard` is a complex component (assume it does heavy rendering).
- Typing in the filter input causes every `UserCard` to re-render on every keystroke.
- Fix the performance without changing observable behavior.

**Acceptance Criteria:**
- [ ] `UserCard` only re-renders when its own `user` prop changes.
- [ ] The filter + sort computation does not run on every render unrelated to filter/sort changes.
- [ ] The `onDelete` callback does not cause all cards to re-render when state changes.
- [ ] Profile with React DevTools before and after — show at least 5× improvement in render time.

**Hints:**
1. `useMemo` the `filtered` array — only recompute when `users`, `filter`, or `sortKey` change.
2. Wrap `UserCard` in `React.memo` to skip re-renders when props haven't changed.
3. `useCallback` the `onDelete` handler — inline arrow functions create a new reference every render, defeating `React.memo`.
4. Consider whether the sort should happen inside or outside of `useMemo` and what its dependency array should be.

---

## Exercise 10 — Canvas Particle Animation (Hard)

**Goal:** Render 500 animated particles on a `<canvas>` element that move and bounce off the walls, using `requestAnimationFrame`.

**Requirements:**
- Each particle has a random position, velocity, size (2–8px), and color.
- Particles bounce off all four canvas edges.
- The canvas resizes to fill the window; particles re-scale accordingly.
- A "Pause/Resume" button stops and starts the animation without resetting particle positions.
- Show the current FPS in the top-left corner.

**Acceptance Criteria:**
- [ ] Consistent 60 FPS on a modern laptop (measure with the FPS counter or Chrome DevTools).
- [ ] No particles escape the canvas boundaries.
- [ ] Resize handler does not create additional animation loops.
- [ ] All Canvas API operations use the 2D context correctly (no WebGL required).

**Hints:**
1. Store particles in a plain array of objects `{ x, y, vx, vy, r, color }` — keep them out of React state.
2. Use a `useRef` for the canvas and the `animationFrameId` to avoid triggering re-renders.
3. FPS calculation: `fps = 1000 / (currentTimestamp - lastTimestamp)`. Smooth it with a rolling average of the last 60 frames.
4. On resize: use a `ResizeObserver` on the canvas container, update canvas `width`/`height` attributes (not CSS dimensions), and clamp particle positions to the new bounds.

---

## Exercise 11 — Custom useLocalStorage Hook (Easy)

**Goal:** Write a `useLocalStorage<T>` hook that works like `useState` but persists the value in `localStorage`.

**Requirements:**
- Signature: `function useLocalStorage<T>(key: string, initialValue: T): [T, (value: T) => void]`
- On mount, reads the stored value. Falls back to `initialValue` if nothing is stored or JSON parsing fails.
- Updates `localStorage` whenever the value is set.
- Multiple tabs should stay in sync via the `storage` event.

**Acceptance Criteria:**
- [ ] Works with any JSON-serializable type `T` (string, number, object, array).
- [ ] Handles `localStorage.getItem` returning `null` gracefully.
- [ ] Handles corrupted JSON (catch parse errors, fall back to `initialValue`).
- [ ] The cross-tab sync updates the state without a page reload.
- [ ] The hook is fully typed — no `any` in the implementation.

**Hints:**
1. Initialize state with a lazy initializer function in `useState` to avoid running `localStorage.getItem` on every render.
2. In `setValue`, call both `setStoredValue` and `localStorage.setItem`.
3. `window.addEventListener('storage', handler)` fires when another tab sets the same key. Use `useEffect` to add and remove this listener.
4. The `storage` event does NOT fire in the same tab that made the change.

---

## Exercise 12 — Responsive CSS Grid Image Gallery (Easy)

**Goal:** Build an image gallery using CSS Grid that creates a Pinterest-style masonry layout without JavaScript.

**Requirements:**
- Display 20 images of varying heights.
- Layout: 4 columns on desktop, 2 on tablet (< 768px), 1 on mobile (< 480px).
- Images fill the column width, maintain aspect ratio, and have a consistent gap.
- On hover, each image shows an overlay with the image title and a "View" button.
- The hover overlay transitions smoothly (no jump).

**Acceptance Criteria:**
- [ ] Layout uses `display: grid` — no flexbox for the primary layout.
- [ ] True masonry: items fill vertical gaps (CSS `grid-template-rows: masonry` or a `column-count` fallback).
- [ ] Hover overlay: opacity or transform transition, no layout shift.
- [ ] Images are lazy-loaded (`loading="lazy"` on `<img>`).
- [ ] Gallery is navigable by keyboard — Tab through images, Enter activates the overlay.

**Hints:**
1. CSS masonry (`grid-template-rows: masonry`) is supported in Firefox behind a flag; use `columns: 4` as a wider-supported fallback.
2. The hover overlay: position the image wrapper `relative`, the overlay `absolute inset-0`. Use `opacity: 0` → `opacity: 1` on `:hover` with `transition: opacity 200ms`.
3. For keyboard: add `tabindex="0"` to each gallery item and handle `:focus-within` the same as `:hover` in CSS.
