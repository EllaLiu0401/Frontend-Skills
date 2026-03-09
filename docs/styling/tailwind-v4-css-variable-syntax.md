# Tailwind CSS v4 — CSS Variable Syntax

## What Changed in v4

Tailwind CSS v4 introduced shorthand syntax for using CSS custom properties (variables) directly in utility classes.

### Old syntax (v3 and still valid in v4)

```html
<div class="bg-[var(--my-color)]">
```

### New shorthand syntax (v4 only)

```html
<div class="bg-(--my-color)">
```

Both are equivalent. The parenthesis shorthand `(--var-name)` is v4's way of saying "resolve this CSS variable inline."

---

## When Each Syntax Applies

| Syntax | Tailwind version | Notes |
|---|---|---|
| `bg-[var(--token)]` | v3 + v4 | Always safe, verbose |
| `bg-(--token)` | v4 only | Shorthand, cleaner |

If your project is on Tailwind v4, both work. Pick one and be consistent throughout the codebase.

---

## How It Works Under the Hood

Tailwind generates the utility class by substituting the variable reference:

```css
/* bg-(--my-bg-color) generates: */
.bg-\(--my-bg-color\) {
  background-color: var(--my-bg-color);
}
```

The CSS variable must be defined somewhere in scope (`:root`, a parent element, or a theme block):

```css
:root {
  --my-bg-color: #f0f0f0;
}
```

---

## Common Mistake: Confusing with Arbitrary Values

Tailwind arbitrary values use square brackets `[]`. The `()` syntax is specifically for CSS variables:

```html
<!-- ✅ Arbitrary pixel value — square brackets -->
<div class="w-[42px]">

<!-- ✅ CSS variable — parentheses (v4 shorthand) -->
<div class="bg-(--my-color)">

<!-- ✅ CSS variable — explicit var() in square brackets (v3 style) -->
<div class="bg-[var(--my-color)]">

<!-- ❌ Wrong — mixing syntax -->
<div class="bg-(42px)">
<div class="bg-[--my-color]">
```

---

## Practical Rule

> If you see `bg-(--something)` in a v4 codebase — it's valid, not a typo.

If the CSS variable is properly defined in your stylesheet, the class will resolve. If it silently has no effect, check:

1. Is the variable defined in `:root` or a relevant theme scope?
2. Are you on Tailwind v4? (Check `package.json` for `tailwindcss: ^4.x`)
3. Is PostCSS configured with `@tailwindcss/postcss`?
