# Custom Combobox — Accessibility Patterns and Common Pitfalls

## Context

These learnings come from building and reviewing a custom combobox (searchable select) component from scratch using HTML, React, and a utility CSS framework. The component replaces the native `<select>` element with a fully custom dropdown that supports icons, search filtering, keyboard navigation, and a clear action.

---

## 1. ARIA Roles for the Combobox Pattern

A custom select must replicate what the browser provides natively. The correct ARIA structure depends on whether the component is searchable.

### Non-searchable select (basic dropdown)

```tsx
<button
  role="combobox"
  aria-expanded={isOpen}
  aria-haspopup="listbox"
  aria-controls="listbox-id"
  aria-activedescendant={activeOptionId}
  aria-label="Choose a fruit"
>
  Selected value
</button>

<ul id="listbox-id" role="listbox">
  <li role="option" aria-selected={isSelected} id="option-0">Apple</li>
  <li role="option" aria-selected={isSelected} id="option-1">Banana</li>
</ul>
```

### Searchable select (combobox with text input)

When a search input is present, the **input** carries `role="combobox"`, not the trigger button.

```tsx
{/* Trigger button — opens the panel, no combobox role */}
<button aria-label="Choose a fruit" aria-expanded={isOpen}>
  Selected value
</button>

{/* Search input — this is the combobox */}
<input
  role="combobox"
  aria-expanded={true}
  aria-haspopup="listbox"
  aria-autocomplete="list"
  aria-controls="listbox-id"
  aria-activedescendant={activeOptionId}
  aria-label="Choose a fruit"   {/* use component label, not placeholder */}
  placeholder="Search..."
/>

<ul id="listbox-id" role="listbox">
  <li role="option" aria-selected={false} id="option-0">Apple</li>
</ul>
```

### Key ARIA attributes

| Attribute | Where | Purpose |
|---|---|---|
| `role="combobox"` | trigger or search input | Identifies the control to screen readers |
| `aria-expanded` | combobox element | Communicates open/closed state |
| `aria-haspopup="listbox"` | combobox element | Indicates a listbox will appear |
| `aria-controls` | combobox element | Links to the listbox by ID |
| `aria-activedescendant` | combobox element | Points to the currently highlighted option |
| `role="listbox"` | `<ul>` container | Identifies the options container |
| `role="option"` | each `<li>` | Identifies each selectable item |
| `aria-selected` | each option | Whether this option is the current selection |
| `aria-label` | combobox | Accessible name — must never be `undefined` |

---

## 2. WCAG Violations Easy to Miss in Custom Selects

### 2a. `aria-label` silently becomes `undefined`

**The trap**: You pass an optional prop directly as `aria-label`.

```tsx
// ❌ Wrong — if clearLabel is not provided, aria-label={undefined}
// Screen readers get no accessible name → WCAG 4.1.2 failure
<button aria-label={clearLabel}>
  <IconX />
</button>

// ✅ Correct — always provide a fallback
<button aria-label={clearLabel ?? 'Clear'}>
  <IconX />
</button>
```

Same applies to search input accessible names:

```tsx
// ❌ Wrong — searchPlaceholder may not be provided
<input aria-label={searchPlaceholder} placeholder={searchPlaceholder} />

// ✅ Correct — derive accessible name from the component's label for richer context
<input
  aria-label={label ?? searchPlaceholder ?? 'Search'}
  placeholder={searchPlaceholder}
/>
```

**Rule**: Every interactive element must have a non-empty accessible name at runtime. Never use an optional prop as the sole source of `aria-label` without a fallback.

### 2b. Icon-only buttons require an accessible name

Any button that contains only an icon and no visible text must have `aria-label`. This includes:
- Clear / reset buttons (× icon)
- Chevron-only triggers
- Search icon buttons

```tsx
// ❌ Wrong — no accessible name
<button onClick={handleClear}>
  <IconX size={14} />
</button>

// ✅ Correct
<button aria-label={clearLabel ?? 'Clear'} onClick={handleClear}>
  <IconX size={14} aria-hidden="true" />
</button>
```

Note: add `aria-hidden="true"` on decorative icons inside labelled buttons to avoid screen readers announcing both the label and the icon name.

### 2c. Touch target minimum: 44×44px

WCAG 2.5.5 (AAA) recommends 44×44px. WCAG 2.5.8 (AA) sets a lower bound of 24×24px. In practice, aim for 44px.

```tsx
// ❌ Too small
<button className="w-6 h-6">×</button>

// ✅ Minimum hit area
<button className="min-w-11 min-h-11 flex items-center justify-center">×</button>
// min-w-11 = min-h-11 = 2.75rem = 44px in Tailwind
```

When the button is inside a flex container with a fixed height smaller than 44px, use `self-stretch` for height and `min-w-11` for width:

```tsx
// Container is h-8 (32px) for "sm" size — min-h-11 would overflow
// ✅ self-stretch fills the container height; min-w-11 keeps 44px width
<button className="min-w-11 self-stretch flex items-center justify-center">×</button>
```

### 2d. `role="status"` inside `role="listbox"` violates ARIA rules

A `<ul role="listbox">` only allows `role="option"` children. Placing an empty-state message inside it breaks ARIA required-children rules.

```tsx
// ❌ Wrong — role="status" is not a valid child of role="listbox"
<ul role="listbox">
  {options.length === 0 && (
    <div role="status" aria-live="polite">No results</div>
  )}
</ul>

// ✅ Correct option A — place it outside the listbox
<ul role="listbox">
  {options.map(...)}
</ul>
{options.length === 0 && (
  <div role="status" aria-live="polite">No results</div>
)}

// ✅ Correct option B — use a disabled option inside the list
<ul role="listbox">
  {options.length === 0 && (
    <li role="option" aria-disabled="true" aria-selected={false}>No results</li>
  )}
</ul>
```

---

## 3. CSS Class Location Determines `:focus` Scope — A Regression Trap

### The problem

When you move a styling class from a **focusable** element to a **non-focusable** wrapper, any CSS rules using `:focus` on that class silently stop working.

**Scenario**: A CSS framework has `.select:focus { outline: 2px solid blue; }`. Originally `.select` is on a `<button>`. You restructure the layout and move `.select` to an outer `<div>` to fix a visual issue.

```tsx
// Before — .select on a button (focusable): focus ring works ✅
<button className="select select-primary">...</button>

// After — .select on a div (not focusable): focus ring never fires ❌
<div className="select select-primary">
  <button className="bg-transparent">...</button>
</div>
```

`<div>` does not receive focus naturally (it has no `tabIndex`). The `:focus` pseudo-class never matches, so `.select:focus` never triggers.

### The fix: `focus-within`

`:focus-within` matches an element when **any descendant** has focus. Apply it on the wrapper instead of relying on `:focus`.

```tsx
// ✅ Correct — wrapper shows focus ring when inner button is focused
<div className="select select-primary focus-within:ring-2 focus-within:ring-primary focus-within:ring-offset-2">
  <button className="bg-transparent focus:outline-none">...</button>
</div>
```

Key points:
- `focus-within:ring-2` on the wrapper — fires when any child receives focus
- `focus:outline-none` on the inner button — suppresses the browser's own ring to avoid double rings
- Color variants: change `ring-primary` to `ring-error` for error state

### Tailwind v4 shorthand

```tsx
// Tailwind v4 — CSS custom property shorthand
'focus-within:ring-2 focus-within:ring-(--color-primary) focus-within:ring-offset-2'
```

---

## 4. `scrollIntoView` — Keyboard vs Mouse Distinction

### The bug

A common pattern for keeping the highlighted option visible is:

```tsx
useEffect(() => {
  if (activeOptionId) {
    document.getElementById(activeOptionId)?.scrollIntoView({ block: 'nearest' });
  }
}, [activeOptionId]);
```

This fires on **every** `activeIndex` change — including changes triggered by `onMouseEnter`. When a user hovers over a partially visible item, the list jumps to scroll it fully into view, which is jarring and unexpected.

### The fix: track navigation source

```tsx
const lastNavSourceRef = useRef<'keyboard' | 'mouse'>('mouse');

// In keyboard handlers (ArrowUp, ArrowDown)
lastNavSourceRef.current = 'keyboard';
setActiveIndex(prev => ...);

// In mouse handlers (onMouseEnter)
lastNavSourceRef.current = 'mouse';
setActiveIndex(idx);

// In the scroll effect — only scroll on keyboard
useEffect(() => {
  if (activeOptionId && lastNavSourceRef.current === 'keyboard') {
    document.getElementById(activeOptionId)?.scrollIntoView({ block: 'nearest' });
  }
}, [activeOptionId]);
```

`useRef` is appropriate here (not `useState`) because changing it should not trigger a re-render — it's just a signal for the side effect.

---

## 5. Default Prop Values Must Match Documentation

When documenting that a prop defaults to a specific value, that default must be explicitly set in the destructuring. TypeScript's optional (`?`) typing does NOT provide runtime defaults.

```tsx
// ❌ Wrong — prop is optional in type, but no runtime default
// If consumer omits it, the rendered output is blank
interface SelectProps {
  placeholder?: string;  // documented default: 'Select…'
}
function Select({ placeholder }: SelectProps) {
  return <button>{selectedLabel ?? placeholder}</button>; // renders nothing if omitted
}

// ✅ Correct — default in destructuring matches the docs
function Select({ placeholder = 'Select…' }: SelectProps) {
  return <button>{selectedLabel ?? placeholder}</button>;
}
```

**Checklist**: If your PR description or docs say "defaults to X", confirm the destructuring actually has `= X`.

---

## 6. Refactoring Regression Checklist

When restructuring a component's DOM (e.g. moving visual classes from one element to another to fix a layout issue), always verify these don't regress:

| Behavior | What to check |
|---|---|
| Focus ring | Does `:focus` still fire on the element carrying the visual class? |
| Touch target size | Did any button's `min-w-*` or `min-h-*` get removed? |
| `aria-*` propagation | Are all ARIA attributes still on the correct elements? |
| Event handlers | Did any `onClick`, `onKeyDown`, `onBlur` move to a non-interactive element? |
| CSS pseudo-classes | Do `:hover`, `:active`, `:focus` still work after moving class names? |

---

## 7. PR Review Response Format

When responding to code review comments on a component, classify each comment clearly:

### Fixed

Use Before/After with one sentence explaining the root cause.

```
Fixed!

Before: `aria-label={clearLabel}` — undefined when clearLabel is omitted,
leaving this icon-only button with no accessible name (WCAG 4.1.2 failure).

After: `aria-label={clearLabel ?? 'Clear'}` — safe fallback ensures the
button always has an accessible name.
```

### Acknowledged (not fixing now)

Explain why and whether it is tracked as a follow-up.

```
Acknowledged — keeping as-is for now. [Brief reason]. Happy to revisit
if [specific condition]. Tracked as a known gap / follow-up.
```

### Intentional design decision

Explain the architectural reason and provide a migration path for consumers.

```
Intentional design decision. [Feature X] was a native HTML feature with
no direct equivalent in a custom [pattern]. Adding it back is a valid
follow-up — tracked as a known gap. Consumers currently using [X] can
[workaround] in the meantime.
```

---

## 8. Keyboard Navigation Patterns for Custom Listboxes

```
Key         | Action
------------|-----------------------------------------------------
ArrowDown   | Move highlight to next option (no wrap)
ArrowUp     | Move highlight to previous option (no wrap)
Enter       | Select highlighted option, close, return focus to trigger
Space       | Open dropdown (when trigger is focused and closed)
Escape      | Close dropdown, return focus to trigger
Tab         | Close dropdown (via onBlur), move to next focusable element
```

Focus management rules:
- On open: focus moves to search input (searchable) or stays on trigger (non-searchable)
- On select: focus returns to trigger
- On Escape: focus returns to trigger
- On click-outside: close via document `mousedown` listener + `onBlur` on the container

---

## Quick Reference

### Accessible combobox checklist

- [ ] `role="combobox"` with `aria-expanded`, `aria-haspopup="listbox"`, `aria-controls`
- [ ] `aria-label` on combobox — non-empty at all times (use fallback)
- [ ] `role="listbox"` on options container
- [ ] `role="option"` + `aria-selected` on each item
- [ ] `aria-activedescendant` links combobox to highlighted option by ID
- [ ] Icon-only buttons have `aria-label` + icons have `aria-hidden="true"`
- [ ] Touch targets ≥44px (`min-w-11`) for all interactive elements
- [ ] No-results text is outside `role="listbox"` or is `role="option" aria-disabled="true"`
- [ ] `scrollIntoView` only fires on keyboard navigation, not mouse hover
- [ ] Focus ring is visible when keyboard navigation reaches the component
- [ ] Escape closes and returns focus to trigger
- [ ] Default prop values in code match documented defaults

---

*These patterns apply to any custom dropdown, combobox, or select built without a dedicated headless library. If using a library (Radix UI, Headless UI, React Aria), many of these are handled automatically.*
