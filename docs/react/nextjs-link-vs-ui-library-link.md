# Using next/link vs UI Library Link for Internal Navigation

## Problem Description

**What happened**:
- A shared layout component used a `Link` component from a UI library for back navigation
- The UI library's `Link` rendered a plain `<a>` tag with no awareness of the Next.js router
- Every click on the back button caused a **full page reload** instead of client-side navigation

**Expected behavior**:
- Clicking internal navigation links should perform client-side routing — no full reload, no losing in-memory state

**Impact**:
- UX degradation: visible flash/reload on every navigation
- Loss of client-side state (scroll position, cached data, etc.)
- Wasted network requests for resources that were already loaded

## Root Cause Analysis

### Technical Explanation

UI component libraries ship framework-agnostic components. Their `Link` component is typically a styled wrapper around a native `<a>` tag:

```tsx
// What UI library Link typically looks like under the hood
export const Link = forwardRef<HTMLAnchorElement, LinkProps>(
  ({ className, children, ...rest }, ref) => {
    return (
      <a ref={ref} className={clsx('link', className)} {...rest}>
        {children}
      </a>
    );
  }
);
```

A plain `<a>` tag does not integrate with Next.js's router. Every click triggers a full browser navigation.

```tsx
// ❌ Problem: using UI library Link for internal routing
import { Link } from '@your-ui-library/ui';

function BackButton({ href, label }: { href: string; label: string }) {
  return (
    <Link href={href}>  {/* plain <a> tag — full page reload */}
      ← {label}
    </Link>
  );
}
```

### Why This Happens

Next.js's `next/link` intercepts click events and uses the client-side router to swap pages without a full reload. A plain `<a>` tag bypasses this entirely — the browser handles the navigation, causing a full HTTP request and page re-render.

### Contributing Factors

- The UI library `Link` and `next/link` share the same prop API (`href`, `className`, `children`), making it easy to use one in place of the other without noticing the behavioral difference
- No visual difference at runtime — both look identical; the difference only shows in network behavior

## Solution Approach

### Separation of Concerns

These two `Link` components serve **different purposes** and are not interchangeable:

| Component | Responsibility | Use Case |
|-----------|---------------|----------|
| `next/link` | Client-side routing | Internal app navigation |
| UI library `Link` | Styling / markup | External links, decorative anchors |

### The Fix

Replace the UI library `Link` with `next/link` for internal navigation. Keep styling via `className`:

```tsx
// ✅ Correct: next/link for internal navigation
import Link from 'next/link';

function BackButton({ href, label }: { href: string; label: string }) {
  return (
    <Link
      href={href}
      className="inline-flex items-center gap-2 text-muted hover:text-foreground no-underline"
    >
      ← {label}
    </Link>
  );
}
```

### Why This Works

`next/link` hooks into Next.js's router. It prefetches the target page in the background and performs a client-side transition on click — no full reload, state is preserved.

## Should the UI Library Wrap next/link?

**No.** Keep the UI library framework-agnostic.

Adding `next/link` into a shared design system creates a Next.js dependency, making the library unusable in non-Next.js projects (Vite, Remix, plain React, etc.).

The correct layering is:

```
Design System  →  Styles and markup (no router knowledge)
Next.js App    →  Routing logic (next/link, useRouter, etc.)
```

## Prevention Strategy

### Code Review Checklist

- [ ] Is this link navigating to an internal route within the app?
- [ ] If yes, is it using `next/link` (not a plain `<a>` or UI library Link)?
- [ ] Is this link pointing to an external URL or opening a new tab? Then a plain `<a>` / UI library Link is correct.

### Decision Rule

```
Internal route (/dashboard, /settings, etc.)  →  next/link
External URL (https://...) or mailto:          →  <a> or UI library Link
```

### How to Spot This Issue

**Code smell**: Importing `Link` from a UI library and using it with a relative `href` like `/dashboard` or `/settings`.

**Questions to ask during review**:
- "Does this `href` point to a page inside this app?"
- "What does this `Link` component render — `<a>` or `next/link`?"

## Lessons Learned

1. **Not all Links are equal**: A styled `<a>` tag and `next/link` look identical in code but behave completely differently at runtime.
2. **UI libraries are framework-agnostic by design**: They intentionally avoid framework-specific routing to stay reusable.
3. **Responsibility separation matters**: Styling is the UI library's job; routing is the framework's job. Don't mix them.

## Related Issues

- [Implicit vs Explicit Client Components](implicit-vs-explicit-client-components.md) — another case where Next.js-specific behavior is easy to miss

## References

- [Next.js Link documentation](https://nextjs.org/docs/app/api-reference/components/link)

---

**Tags**: `#nextjs` `#routing` `#navigation` `#ui-library`

**Date Added**: 2026-02-19

**Severity**: Medium

**Source**: PR Review
