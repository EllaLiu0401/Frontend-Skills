# Link-as-Button Component: Common Pitfalls in Design Systems

## Overview

When building a design system, it's common to create a `ButtonLink` component — an `<a>` element styled to look like a `<button>`. This pattern enables navigation with button-like visuals. However, wrapping `<a>` in button clothing introduces subtle gaps that are easy to miss and hard to catch in review.

This document covers three categories of pitfalls learned from real code review findings, and how to critically evaluate such feedback.

## Pitfall 1: Missing Secure Defaults for External Links

### The Problem

A `ButtonLink` component that renders `<a>` and spreads props does not automatically enforce `rel="noopener noreferrer"` when `target="_blank"` is present. Storybook examples may show the correct usage, but the component API itself allows unsafe external links.

```tsx
// ❌ Unsafe: relies on consumer to remember security attributes
const ButtonLink = forwardRef<HTMLAnchorElement, ButtonLinkProps>(
  ({ className, children, ...rest }, ref) => (
    <a ref={ref} className={clsx('btn', className)} {...rest}>
      {children}
    </a>
  )
);

// Consumer must manually add rel — easy to forget
<ButtonLink href="https://example.com" target="_blank" rel="noopener noreferrer">
  Visit Site
</ButtonLink>
```

### Why It Matters

- **`noopener`** prevents the opened page from accessing `window.opener`, blocking tabnabbing attacks
- **`noreferrer`** hides the referrer header from the destination (privacy concern — may or may not be desired)
- While modern browsers (Chrome 88+, Firefox 79+, Safari 12.1+) implicitly apply `noopener` behavior on `target="_blank"`, a design system should be **secure-by-default** as defense-in-depth

### The Fix

Auto-inject `rel` tokens when `target="_blank"` is detected:

```tsx
const ButtonLink = forwardRef<HTMLAnchorElement, ButtonLinkProps>(
  ({ className, children, target, rel, ...rest }, ref) => {
    const computedRel =
      target === '_blank' ? mergeRel(rel, 'noopener noreferrer') : rel;

    return (
      <a
        ref={ref}
        target={target}
        rel={computedRel}
        className={clsx('btn', className)}
        {...rest}
      >
        {children}
      </a>
    );
  }
);

function mergeRel(
  existing: string | undefined,
  required: string
): string {
  const tokens = new Set([
    ...(existing?.split(/\s+/) ?? []),
    ...required.split(/\s+/),
  ]);
  return [...tokens].filter(Boolean).join(' ');
}
```

### Severity Assessment

This is often reported as a P0/blocker, but realistically:

- **Modern browsers mitigate tabnabbing** by default since ~2021
- `noreferrer` is a separate privacy decision, not strictly a security fix
- **Appropriate severity: P1** — good defense-in-depth improvement, not an emergency

### Key Principle

> **Secure-by-default > Secure-by-documentation.** If a component can be used unsafely, it will be. Don't rely on Storybook examples or docs to enforce security — enforce it in the component API.

---

## Pitfall 2: `<a>` Elements Don't Support Native Disabled State

### The Problem

The `<button>` element has a native `disabled` attribute that:
- Applies `:disabled` CSS pseudo-class styling
- Removes the element from tab order
- Prevents click events
- Announces disabled state to screen readers

The `<a>` element has **none of these behaviors**. If your CSS uses `:disabled` for disabled styling, it will silently fail on `<a>` elements:

```css
/* This ONLY works on form elements (<button>, <input>, <select>) */
.btn:disabled {
  cursor: not-allowed;
  opacity: 0.4;
}

/* ❌ This will NEVER match an <a> element, even with disabled attribute */
```

If a developer passes `disabled` to a link-as-button component, the prop will be spread onto the `<a>` tag as an HTML attribute. But:
- No visual change occurs (`:disabled` doesn't match)
- The link remains clickable
- The link remains in tab order
- Screen readers don't announce it as disabled
- **It fails silently with zero indication**

### The Fix

Implement disabled behavior explicitly for `<a>`-based components:

```tsx
interface ButtonLinkProps extends Omit<ComponentPropsWithoutRef<'a'>, 'href'> {
  href: string;
  disabled?: boolean;
  tone?: ButtonTone;
  size?: ButtonSize;
}

const ButtonLink = forwardRef<HTMLAnchorElement, ButtonLinkProps>(
  ({ disabled, className, children, onClick, ...rest }, ref) => (
    <a
      ref={ref}
      aria-disabled={disabled || undefined}
      tabIndex={disabled ? -1 : undefined}
      className={clsx(
        'btn',
        disabled && 'btn-disabled pointer-events-none opacity-40',
        className
      )}
      onClick={
        disabled
          ? (e: React.MouseEvent) => e.preventDefault()
          : onClick
      }
      {...rest}
    >
      {children}
    </a>
  )
);
```

Required behaviors for a disabled `<a>`:

| Behavior | Native `<button disabled>` | Must implement for `<a>` |
|---|---|---|
| Visual disabled styling | `:disabled` pseudo-class | CSS class (e.g., `.btn-disabled`) |
| Remove from tab order | Automatic | `tabIndex={-1}` |
| Prevent click/navigation | Automatic | `onClick → preventDefault()` |
| Screen reader announcement | Automatic | `aria-disabled="true"` |

### Key Principle

> **Styled-as does not mean behaves-as.** An `<a>` styled as a button is still an `<a>` — it inherits anchor semantics, not button semantics. Every behavioral gap must be manually bridged.

---

## Pitfall 3: API Parity Between Sibling Components

### The Problem

When a design system has `Button` (renders `<button>`) and `ButtonLink` (renders `<a>`) sharing the same visual presentation, developers expect the same behavioral API surface. If `Button` supports `disabled` but `ButtonLink` doesn't, this creates an **API inconsistency** that leads to:

- Developer confusion ("Why doesn't `disabled` work here?")
- Silent failures (prop is accepted but has no effect)
- Workarounds scattered across the codebase

### The Rule

If two components share visual presentation, audit these properties for parity:

| Property | `<button>` native | `<a>` needs manual |
|---|---|---|
| `disabled` | Yes | Yes — see Pitfall 2 |
| Focus styles | `:focus-visible` works | `:focus-visible` works (same) |
| Hover/active states | `:hover`, `:active` | `:hover`, `:active` (same) |
| Click prevention | `disabled` blocks clicks | Must use `preventDefault` |
| Keyboard activation | Enter + Space | Enter only (by default) |

### Key Principle

> **Component API parity reduces cognitive load.** When developers learn one component's API, they should be able to transfer that knowledge to its sibling. Missing properties should be intentional and documented, not accidental.

---

## Bonus: How to Critically Evaluate Code Review Findings

Not every reported issue is valid. Here's a framework for assessing findings:

### 1. Check Browser Reality

Reviewers may cite vulnerabilities that browsers have already mitigated. Before accepting a security finding:
- Check MDN and caniuse.com for current browser behavior
- Verify if the attack vector is still exploitable in target browsers
- Assess if the fix is defense-in-depth (good) or addressing a phantom threat (wasteful)

### 2. Check for Intentional Naming

Example: A reviewer flags `ErrorTone` as inconsistent, suggesting renaming to `Error`. But `Error` shadows JavaScript's built-in `Error` constructor, which triggers ESLint's `no-shadow` rule. The "inconsistency" is actually an intentional workaround.

**Before accepting a naming issue:** Check if the current name avoids a lint rule, keyword collision, or reserved word conflict.

### 3. Verify Claims Against Actual Code

Reviewers may not actually compare files side-by-side. Example: A reviewer claims heading styles are "inconsistent" across two story files, but both files use identical markup. Always diff the actual code before acting on consistency complaints.

### 4. Distinguish Bugs from Feature Requests

| Finding Type | Example | Action |
|---|---|---|
| **Bug** | `:disabled` CSS doesn't work on `<a>` | Fix immediately |
| **Security gap** | Missing `rel` on external links | Fix with appropriate priority |
| **Feature request** | "Add auto external-link icon" | Evaluate against roadmap |
| **Style preference** | "Rename story export" | Check if change is safe |

### Severity Assessment Cheat Sheet

| Claimed Severity | Questions to Ask |
|---|---|
| P0 (Blocker) | Is there actual data loss, security exploit, or crash? Can users work around it? |
| P1 (Important) | Does it cause silent failures? Does it affect multiple consumers? |
| P2 (Polish) | Is it cosmetic? Is the "inconsistency" actually intentional? |

---

## Checklist: Building Link-as-Button Components

When implementing or reviewing a link styled as a button, verify:

- [ ] `rel="noopener noreferrer"` is auto-applied when `target="_blank"` is present
- [ ] Disabled state works: visual styling, tab order removal, click prevention, `aria-disabled`
- [ ] CSS disabled styles use class selectors (not `:disabled` pseudo-class) for `<a>` elements
- [ ] API parity with sibling `Button` component is documented and intentional
- [ ] Keyboard behavior is tested (Enter activates `<a>`, Enter+Space activates `<button>`)
- [ ] Focus-visible styles apply consistently across both components
- [ ] TypeScript types expose the correct props (`disabled`, `rel`, `target`)

## Related Patterns

- [next/link vs UI Library Link](./nextjs-link-vs-ui-library-link.md) - When to use framework router links vs component library links
- [Semantic HTML Heading Hierarchy](../accessibility/semantic-html-heading-hierarchy.md) - Related accessibility patterns

## References

- [MDN: Link types — noopener](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/noopener)
- [MDN: HTMLAnchorElement](https://developer.mozilla.org/en-US/docs/Web/API/HTMLAnchorElement)
- [WHATWG: Links created by `<a>` elements with `target="_blank"` imply `noopener`](https://html.spec.whatwg.org/multipage/links.html#link-type-noopener)
- [WAI-ARIA: aria-disabled](https://www.w3.org/TR/wai-aria-1.2/#aria-disabled)

---

**Tags**: `#react` `#design-system` `#accessibility` `#security` `#component-api`

**Date Added**: 2026-02-23

**Difficulty**: Intermediate
