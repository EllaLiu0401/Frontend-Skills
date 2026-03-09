# Custom Dropdown ARIA Patterns

Lessons from building an accessible custom select/combobox component from scratch.

---

## 1. Role Architecture: Two Valid Patterns

When building a custom dropdown, choose the pattern that matches your interaction model.

### Pattern A — Select-only combobox (no search input)

The trigger button becomes the combobox. It manages keyboard focus and announces the active option via `aria-activedescendant`.

```html
<!-- Trigger -->
<button
  role="combobox"
  aria-haspopup="listbox"
  aria-expanded="true"
  aria-controls="listbox-id"
  aria-activedescendant="option-2"
>
  Selected Value
</button>

<!-- Dropdown -->
<ul id="listbox-id" role="listbox">
  <li id="option-1" role="option" aria-selected="false">Apple</li>
  <li id="option-2" role="option" aria-selected="true">Banana</li>
</ul>
```

**Key rule:** `aria-activedescendant` only works when the *owner* element has a composite role like `combobox`, `listbox`, or `grid`. A plain `<button>` without a role cannot use it.

### Pattern B — Editable combobox (with search input)

The `<input>` itself is the combobox. The trigger button becomes a plain button that opens the dropdown.

```html
<!-- Search input = the combobox -->
<input
  type="text"
  role="combobox"
  aria-haspopup="listbox"
  aria-expanded="true"
  aria-autocomplete="list"
  aria-controls="listbox-id"
  aria-activedescendant="option-2"
  aria-label="Search options"
/>

<!-- Dropdown -->
<ul id="listbox-id" role="listbox">
  <li id="option-1" role="option" aria-selected="false">Apple</li>
</ul>
```

**Key rule:** When a visible search input exists, move `role="combobox"` to the `<input>`, not the trigger button.

---

## 2. `aria-activedescendant` — Composite Focus Pattern

Custom dropdowns often need keyboard navigation without moving DOM focus. This is done via the "composite focus" pattern:

- All list items get `tabIndex={-1}` (not tabbable directly)
- Active item is tracked in component state (`activeIndex`)
- The **owner element** (combobox) sets `aria-activedescendant` to the ID of the active item
- The active item's ID must be stable and unique

```tsx
// State-driven active index
const [activeIndex, setActiveIndex] = useState(-1);
const activeOptionId = activeIndex >= 0 ? `listbox-option-${activeIndex}` : undefined;

// Owner element announces active item to screen reader
<button
  role="combobox"
  aria-activedescendant={activeOptionId}
  // ...
/>

// Each option has a stable ID
<li
  id={`listbox-option-${idx}`}
  role="option"
  aria-selected={isSelected}
  tabIndex={-1}
/>
```

**Why `:focus-visible` won't fire on items:** Because DOM focus never moves to the `<li>` elements. The keyboard-active state is purely visual, driven by `isActive` state + conditional CSS class. This is correct and intentional — not a bug.

---

## 3. ARIA Listbox Required Children

`role="listbox"` has strict required-children constraints per the ARIA spec:

| Parent role | Allowed children |
|---|---|
| `listbox` | `option`, `group` |
| `group` (inside listbox) | `option` |

**Common mistake:** Putting a "no results" message inside the `<ul role="listbox">`:

```tsx
// ❌ WRONG — violates ARIA required-children
<ul role="listbox">
  {items.map(...)}
  {items.length === 0 && (
    <li role="status">No results found</li>  // role="status" not allowed here
  )}
</ul>
```

```tsx
// ✅ CORRECT — move empty state outside the listbox
<ul role="listbox">
  {items.map(...)}
</ul>
{items.length === 0 && (
  <div role="status" aria-live="polite">No results found</div>
)}
```

`aria-live="polite"` ensures screen readers announce the empty state when it appears without interrupting ongoing speech.

---

## 4. Always Name Interactive Elements

Every focusable or interactive element must have an accessible name. Common failures:

```tsx
// ❌ Unlabeled search input — screen readers say "edit text"
<input type="text" placeholder="Search…" />

// ✅ Named via aria-label
<input type="text" placeholder="Search…" aria-label="Search options" />

// ✅ Or use a visible <label> with htmlFor
<label htmlFor="search-input">Search</label>
<input id="search-input" type="text" />
```

**Note:** `placeholder` is NOT an accessible name. It disappears when the user types, and screen reader support is inconsistent. Always use `aria-label` or `<label>`.

---

## 5. `onBlur` and Click-Outside: Don't Double-Fire

Closing a dropdown needs both a click-outside listener and a focus-out listener. They can overlap:

```tsx
// mousedown closes the dropdown
useEffect(() => {
  function handleClickOutside(e: MouseEvent) {
    if (!containerRef.current?.contains(e.target as Node)) {
      close();
    }
  }
  document.addEventListener('mousedown', handleClickOutside);
  return () => document.removeEventListener('mousedown', handleClickOutside);
}, [isOpen]);

// onBlur on the container handles tab-away and focus loss
const handleFocusOut = (e: FocusEvent<HTMLDivElement>) => {
  // relatedTarget = where focus is moving TO
  if (containerRef.current?.contains(e.relatedTarget as Node)) return;
  close();
  onBlur?.();
};
```

**Why they don't double-fire:** `mousedown` closes first, then the subsequent focus-change fires `onBlur` on the container — but by that point `isOpen` is already false, so `close()` is a no-op. The `onBlur` callback fires once, correctly.

---

## 6. WCAG Focus Visible (2.4.7) for Programmatic Focus

When keyboard focus is managed programmatically (composite focus pattern), `:focus-visible` won't fire. The visual indicator must be applied manually.

**Minimum requirement (WCAG AA):** A perceivable visual change when an item is keyboard-active. A background color change satisfies this.

```tsx
// Distinct tokens for hover vs keyboard-active states
isActive && 'bg-active-token'   // e.g. rgba(0,0,0,0.08) — pressed/active state
// hover handled by CSS: .item:hover { background: rgba(0,0,0,0.04) } — lighter
```

**Key:** Use different opacity/color tokens. Active state should be visually stronger than hover state.

**WCAG AAA enhancement:** Add a `ring` or `outline` on top of the background change for maximum clarity.

---

## 7. Keyboard Navigation Implementation Checklist

For a fully accessible custom select:

- [ ] **Enter / Space / ArrowDown** on trigger: opens dropdown
- [ ] **ArrowDown / ArrowUp**: moves `activeIndex` through options
- [ ] **Enter / Space** when open: selects the active option
- [ ] **Escape**: closes dropdown, returns focus to trigger
- [ ] **Tab away**: closes dropdown via `onBlur`
- [ ] **Click outside**: closes dropdown
- [ ] Active option scrolled into view when navigating out of visible area
- [ ] `aria-activedescendant` updated as `activeIndex` changes
- [ ] `aria-expanded` reflects open/closed state
- [ ] Focus returns to trigger after selection

---

## Reference

- [ARIA APG — Select-Only Combobox](https://www.w3.org/WAI/ARIA/apg/patterns/combobox/examples/combobox-select-only/)
- [ARIA APG — Editable Combobox](https://www.w3.org/WAI/ARIA/apg/patterns/combobox/examples/combobox-autocomplete-list/)
- [MDN — ARIA listbox role](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Roles/listbox_role)
- [WCAG 2.4.7 — Focus Visible](https://www.w3.org/WAI/WCAG21/Understanding/focus-visible)
