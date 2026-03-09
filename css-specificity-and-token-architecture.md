# CSS Specificity Traps and Token-Based Architecture

Lessons learned from a design system PR review covering CSS specificity issues, TypeScript type guard correctness, and design token derivation patterns.

---

## 1. Selector Specificity vs Utility Classes

### Problem

When overriding component styles for specific themes, a common approach is:

```css
[data-theme='brand'] .card:not(.bg-success, .bg-error, .bg-warning) {
  background-color: #fffaf4;
}
```

This looks reasonable — it targets cards under a specific theme and excludes certain status classes. However, the **combined specificity** of `[attribute] + .class + :not(.class, .class, ...)` is **higher than a single utility class** like `.bg-base-200`.

This means consumers of the design system cannot override the card background with standard utility classes.

### Why It Happens

CSS specificity is calculated by counting selectors:
- `.bg-base-200` = specificity `(0, 1, 0)` — one class
- `[data-theme='brand'] .card:not(.bg-success, .bg-error, ...)` = specificity `(0, 1+1+N, 0)` — attribute + class + each class inside `:not()`

The `:not()` pseudo-class contributes the specificity of its **most specific argument** (in newer CSS) or the sum (in older behavior). Either way, it wins over a single utility class.

### Fix: Use Semantic Tokens Instead of Selector Overrides

Define a semantic token in each theme and let the component consume it:

```css
/* In theme definition */
--background-card: #fffaf4;  /* brand-specific */

/* In base component style */
.card {
  background: var(--background-card, var(--background-surface));
}
```

Benefits:
- **Utility classes override naturally** — they come later in cascade order (Tailwind v4)
- **Follows the token architecture** — primitive → semantic → component
- **Fallback safety** — `var(--background-card, var(--background-surface))` works even if a theme doesn't define the card token
- **No exclusion list to maintain** — the `:not()` approach requires listing every class that should be allowed to override, which is fragile and incomplete

### Rule of Thumb

> If you find yourself writing `[data-theme] .component:not(...)` to style a component differently per theme, you should be defining a semantic token instead.

---

## 2. TypeScript Type Guards Must Be Sound

### Problem

```typescript
type ThemeMode = 'light' | 'dark' | 'system';

function isValidMode(value: string): value is ThemeMode {
  if (value === 'legacy-light') return true;   // NOT a ThemeMode!
  if (value === 'legacy-dark') return true;     // NOT a ThemeMode!
  return value === 'light' || value === 'dark' || value === 'system';
}
```

This type guard returns `true` for `'legacy-light'`, telling TypeScript the value is a `ThemeMode`. But `'legacy-light'` is **not** in the `ThemeMode` union. The type system now trusts a lie.

Even if the value is always migrated afterward (`migrateStoredValue(raw)`), the type annotation is unsound. A future refactor could remove the migration step and introduce a subtle runtime bug that TypeScript won't catch.

### Fix: Separate Recognition from Narrowing

If a function checks for values **outside** the target type (e.g., legacy/deprecated values), it should not be a type guard:

```typescript
function isRecognizedValue(value: string): boolean {
  return (
    value === 'legacy-light' ||
    value === 'legacy-dark' ||
    value === 'light' ||
    value === 'dark' ||
    value === 'system'
  );
}

function migrateStoredValue(value: string): ThemeMode {
  if (value === 'legacy-light') return 'light';
  if (value === 'legacy-dark') return 'dark';
  if (value === 'light' || value === 'dark' || value === 'system') return value;
  return 'system'; // safe default
}

// Usage
const stored = raw !== null && isRecognizedValue(raw)
  ? migrateStoredValue(raw)  // migrateStoredValue handles the actual narrowing
  : null;
```

### Rule of Thumb

> A type guard (`value is T`) must only return `true` when the value genuinely satisfies `T`. If it also accepts legacy/deprecated values, use a plain `boolean` return and let a separate migration function handle the narrowing.

---

## 3. Deriving Colors from a Limited Design Token Palette

### Problem

Your design system defines a strict set of base colors (e.g., 8 core colors). A theme needs a hover variant that's lighter than the accent color, but you can't add arbitrary hex values.

### Solution: `color-mix()` with Existing Tokens

CSS `color-mix()` lets you derive new shades from existing colors without introducing new hex values:

```css
/* Derive hover shade: 80% accent + 20% white */
--interactive-hover: color-mix(in srgb, var(--accent) 80%, var(--white));
```

This produces a visibly lighter shade (~`#33cbef` from `#00beeb` + `#ffffff`) while staying within the authorized color palette.

### When to Use

- **Hover/focus states** that need a lighter or darker variant of a base color
- **Alpha/opacity tokens** — `color-mix(in srgb, var(--accent) 10%, transparent)` for subtle backgrounds
- **Dark mode adjustments** — mix with black instead of white for darker variants

### Pattern: Override Outside the Theme Block

If theme definition blocks have processing constraints (e.g., DaisyUI plugin blocks), define the `color-mix()` override in a regular CSS selector:

```css
/* Theme block defines the base value */
@plugin "theme" {
  name: 'brand';
  --interactive-hover: var(--accent); /* placeholder */
}

/* Regular CSS overrides with derived value */
:root[data-theme='brand'] {
  --interactive-hover: color-mix(in srgb, var(--accent) 80%, var(--white));
}
```

### Rule of Thumb

> When a design token palette is locked, use `color-mix()` to derive variants instead of adding new hex values. This keeps the palette authoritative while allowing necessary visual differentiation.

---

## 4. Keeping Documentation in Sync with Architecture

### Problem

A theming guide recommended using CSS selector overrides for custom card backgrounds:

```
| Custom card background | Add `[data-theme='brand'] .card { background-color: ... }` |
```

This recommendation directly caused the specificity bug described in Section 1. After fixing the code to use semantic tokens, the documentation still recommended the old (broken) approach.

### Lesson

When fixing an architectural bug, check if any documentation or guides recommend the pattern you're removing. If the guide is not updated, the next developer adding a new theme will reintroduce the same bug.

### Checklist

- [ ] Does the fix change an established pattern?
- [ ] Is that pattern documented in any guide, README, or onboarding doc?
- [ ] Does the token contract (list of required tokens per theme) need updating?
- [ ] Are code comments referencing the old approach?

---

## Summary Cheat Sheet

| Situation | Wrong Approach | Right Approach |
|-----------|---------------|----------------|
| Theme-specific component background | `[data-theme] .card:not(...)` selector | Semantic token in theme definition |
| Type guard for values outside target type | `value is T` (unsound) | `boolean` + separate migration function |
| Need a color variant from locked palette | New hex value | `color-mix()` from existing tokens |
| Fixed a pattern in code | Update code only | Update code + documentation + token contract |
