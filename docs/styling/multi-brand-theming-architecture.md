# Multi-Brand Theming Architecture

Lessons learned from building and reviewing a multi-brand theme system that supports multiple product brands with independent light/dark modes.

---

## 1. Theme Layer Separation: Mode vs. Resolved Theme

A common mistake is conflating what the **user selects** with what the **CSS engine receives**.

### The Three Layers

```
User Selection (mode)  →  Resolution Logic  →  CSS Theme (data-theme)
─────────────────────     ────────────────     ──────────────────────
light                     brand + mode          brand-a
dark                      brand + mode          brand-a-dark
system                    matchMedia query       brand-b
                                                brand-b-dark
```

- **Mode** is brand-agnostic: `light | dark | system`. This is what gets stored in `localStorage`.
- **Resolved theme** is what gets applied to `data-theme` on `<html>`. It combines brand + mode (e.g., `"myapp"`, `"myapp-dark"`, `"otherbrand"`, `"otherbrand-dark"`).

### Why This Matters

If you store the resolved theme name (e.g., `"myapp-dark"`) directly, switching brands later requires migrating every user's stored value. Storing just the mode (`"dark"`) keeps the storage format stable across brand changes.

```typescript
// ❌ Storing resolved theme — ties localStorage to brand
localStorage.setItem('theme', 'myapp-dark');

// ✅ Storing mode — brand-agnostic, stable across changes
localStorage.setItem('theme', 'dark');
```

---

## 2. Brand Resolution Map Pattern

Use a lookup table to map `(brand, mode)` → resolved CSS theme name:

```typescript
type Brand = 'app-a' | 'app-b';
type Mode = 'light' | 'dark' | 'system';
type ResolvedTheme = 'app-a' | 'app-a-dark' | 'app-b' | 'app-b-dark';

const BRAND_THEMES: Record<Brand, { light: ResolvedTheme; dark: ResolvedTheme }> = {
  'app-a': { light: 'app-a', dark: 'app-a-dark' },
  'app-b': { light: 'app-b', dark: 'app-b-dark' },
};

function resolveTheme(mode: Mode, brand: Brand): ResolvedTheme {
  const scheme = mode === 'system' ? getSystemColorScheme() : mode;
  return BRAND_THEMES[brand][scheme];
}
```

Adding a new brand is a one-line addition to the map — no control flow changes needed.

---

## 3. SSR Theme Init Script Must Match the Client Provider

In SSR frameworks (Next.js, Remix), an inline `<script>` runs before React hydration to set `data-theme` and prevent FOUC (Flash of Unstyled Content). This script and the React ThemeProvider **must agree on the storage format**.

### The Bug Pattern

```
1. ThemeProvider stores mode as 'light' / 'dark' / 'system'
2. Init script validates against ['brand-a', 'brand-a-dark', 'system']
3. User selects 'dark' → stored as 'dark'
4. On next page load, init script doesn't recognize 'dark' → falls back to 'system'
5. Flash of wrong theme until React hydrates and corrects it
```

### The Fix

The init script must:
1. Accept the **same values** the ThemeProvider stores (modes: `light`, `dark`, `system`)
2. Handle **legacy values** for backward compatibility (e.g., old `"brand-a-dark"` → treat as `"dark"`)
3. Resolve to the correct `data-theme` attribute using the same brand→theme mapping

```javascript
// Pseudocode for the init script
(function() {
  var stored = localStorage.getItem('theme-key');
  
  // Migrate legacy values
  if (stored === 'brand-a') stored = 'light';
  if (stored === 'brand-a-dark') stored = 'dark';
  
  // Validate
  var valid = ['light', 'dark', 'system'];
  var mode = (stored && valid.indexOf(stored) !== -1) ? stored : 'system';
  
  // Resolve system preference
  if (mode === 'system') {
    mode = matchMedia('(prefers-color-scheme:dark)').matches ? 'dark' : 'light';
  }
  
  // Map mode to brand-specific theme name
  var theme = (mode === 'dark') ? 'brand-a-dark' : 'brand-a';
  document.documentElement.setAttribute('data-theme', theme);
})();
```

### Rule

> Any time you change how the ThemeProvider reads/writes localStorage, you **must** update the SSR init script to match. They are a coupled pair.

---

## 4. Design Token Governance: Only Use Defined Tokens

When creating brand-specific styles, only use colors that exist in that brand's design token set. Do not borrow tokens from another brand.

### Why This Breaks

```css
/* ❌ Brand B's theme references Brand A's token */
[data-theme='brand-b'] .accent-element {
  background: var(--primitive-brand-a-cyan);
  /* This variable doesn't exist in Brand B's theme → transparent/invisible */
}
```

### The Fix: Derive from the Brand's Own Tokens

Use `color-mix()` to create derived values from existing brand tokens:

```css
/* ✅ Derived from Brand B's own tokens */
[data-theme='brand-b'] .accent-element {
  background: linear-gradient(
    145deg,
    color-mix(in srgb, var(--primitive-brand-b-accent) 50%, var(--primitive-brand-b-dark)) 0%,
    var(--primitive-brand-b-accent) 100%
  );
}
```

### Rule

> Before adding any color to a brand theme, verify it exists in that brand's design token document. If the exact color doesn't exist, derive it from existing tokens using `color-mix()` or `opacity`.

---

## 5. CSS `color-mix()` for Theme-Derived Colors

`color-mix()` is powerful for generating hover states, subtle backgrounds, and gradients without introducing new hardcoded colors.

```css
/* Syntax */
color-mix(in <color-space>, <color1> <percentage>, <color2>)

/* Mix 70% of accent with 30% of white → lighter accent */
color-mix(in srgb, var(--accent) 70%, white)

/* Mix 50% accent with 50% dark → deep accent */
color-mix(in srgb, var(--accent) 50%, var(--dark))
```

### Common Use Cases

| Need | Pattern |
|------|---------|
| Hover state (lighter) | `color-mix(in srgb, var(--color) 85%, white)` |
| Hover state (darker) | `color-mix(in srgb, var(--color) 85%, black)` |
| Subtle background | `color-mix(in srgb, var(--color) 10%, transparent)` |
| Gradient endpoint | `color-mix(in srgb, var(--accent) 50%, var(--dark))` |

### Browser Support

`color-mix()` is supported in all modern browsers (Chrome 111+, Firefox 113+, Safari 16.2+). For older browser support, provide a fallback:

```css
.element {
  background: var(--accent); /* fallback */
  background: color-mix(in srgb, var(--accent) 50%, var(--dark));
}
```

---

## 6. `data-theme` Selector Patterns for Brand Overrides

When a shared component needs different styling per brand, use `[data-theme]` attribute selectors:

```css
/* Base style — works for default brand */
.icon-container {
  background: linear-gradient(145deg, #color1, #color2);
}

/* Brand-specific override using data-theme selector */
[data-theme='brand-b'] .icon-container,
[data-theme='brand-b-dark'] .icon-container {
  background: linear-gradient(145deg, var(--brand-b-start), var(--brand-b-end));
}
```

### Specificity Notes

- `[data-theme='x'] .class` has higher specificity than `.class` alone — no `!important` needed.
- Group light and dark variants of the same brand when they share the same override.
- Keep overrides minimal: only override what actually differs between brands.

---

## 7. localStorage Migration Strategy

When changing the storage format (e.g., from theme names to mode names), handle backward compatibility:

```typescript
function migrateStoredValue(raw: string): Mode {
  // Legacy values from old format
  if (raw === 'brand-a') return 'light';
  if (raw === 'brand-a-dark') return 'dark';
  
  // Current format
  if (raw === 'light' || raw === 'dark' || raw === 'system') return raw;
  
  // Unknown value
  return 'system';
}

// On load: read → migrate → persist the migrated value
const raw = localStorage.getItem(key);
const migrated = migrateStoredValue(raw);
if (raw !== migrated) {
  localStorage.setItem(key, migrated);
}
```

### Key Points

- Migration runs once per user, transparently.
- Always persist the migrated value back so the migration doesn't run again.
- The migration function must exist in **both** the SSR init script and the React ThemeProvider.

---

## 8. Storybook Stories Must Match Design Token Documentation

Visual documentation stories (color palettes, typography specimens) serve as living references. They must reflect the source-of-truth design token document exactly.

### Common Discrepancies to Watch For

| Issue | Example |
|-------|---------|
| Missing swatch | Token doc lists 8 colors, story shows 7 |
| Wrong font size | Story shows `1.875rem`, token doc says `3rem` |
| Wrong font family | Story uses system font, token doc specifies a custom typeface |
| Incorrect copy | Story says "applied to badges", token doc says badges use a different font |

### Verification Practice

Before shipping visual documentation stories:
1. Open the design token document side by side
2. Count: does every token have a corresponding visual element?
3. Verify: do the displayed values (hex codes, font sizes, font families) match exactly?

---

## Quick Reference: Multi-Brand Theme Checklist

- [ ] Store **mode** (`light`/`dark`/`system`) in localStorage, not resolved theme names
- [ ] Use a brand→theme lookup map for resolution
- [ ] SSR init script and ThemeProvider use the **same** storage format
- [ ] SSR init script handles legacy stored values (migration)
- [ ] Brand-specific CSS only references that brand's design tokens
- [ ] Use `color-mix()` for derived colors instead of hardcoded hex values
- [ ] Use `[data-theme='x']` selectors for brand overrides (no `!important`)
- [ ] Storybook stories match the design token document exactly

---

**Source**: PR Review (CodeRabbit + Manual)  
**Date**: March 2026  
**Topics**: Theming, Multi-Brand, DaisyUI, CSS Custom Properties, color-mix(), SSR Hydration, Design Tokens, Storybook
