# PR Review — RBAC Server Guards, Fallback Patterns & Test Hygiene

> **TL;DR**: Never put role guards after hooks in client components — use a server component wrapper with `notFound()`. Collapse repetitive fallback merges into a single destructure. Keep e2e nav label defaults role-aware and in sync with actual UI.

## Context

A PR review on an RBAC feature surfaced four findings across a role-gated page, a layout component, and e2e test helpers. Three were real bugs or clear improvements; one was intentionally skipped under KISS.

## Issues & Fixes

### 1) Late Role Guard in Client Component — Double Error UI

**Symptom**: A role-gated page was a `'use client'` component. The role check happened _after_ all hooks (including data-fetching queries). When an unauthorized user visited the page:

1. A data query fired → got 403 → triggered a global error modal
2. Then the role guard rendered an inline "insufficient permissions" alert
3. Result: **two error UIs displayed simultaneously**

```tsx
// ❌ Before: Client component with late role guard
'use client';

export default function ProtectedPage() {
  const { user } = useUser();
  const { isAdmin } = getRoleFlags(user);

  // These hooks fire BEFORE the guard below — even for unauthorized users
  const { data, error } = useQuery(['protected-data'], fetchProtectedData);

  if (user && !isAdmin) {
    return <Alert type="error">Insufficient permissions</Alert>;
  }

  return <Dashboard data={data} />;
}
```

```tsx
// ✅ After: Thin server component wrapper + extracted client component
// page.tsx (server component)
import { notFound } from 'next/navigation';
import { getServerRoleFlags } from '@/lib/auth';
import { ProtectedPageClient } from './ProtectedPageClient';

export default async function ProtectedPage() {
  const { isAdmin } = await getServerRoleFlags();

  if (!isAdmin) {
    notFound(); // Returns 404 — doesn't reveal route exists
  }

  return <ProtectedPageClient />;
}
```

**Root Cause**: React's Rules of Hooks require all hooks to execute unconditionally at the top of a component. In a client component, you cannot short-circuit before hooks — so data queries fire for every visitor, regardless of role.

**Fix**: Wrap the client component in a thin server component that checks roles server-side and calls `notFound()` before any client code loads.

**Three things this fixes at once:**
- No wasted API queries for unauthorized users
- No double error UI (global modal + inline alert)
- Returns 404 instead of revealing the route exists (better security)

**Prevention**:
- All role-gated pages should use a server component wrapper with `getServerRoleFlags()` + `notFound()`
- Client components should never be the first line of defense for authorization
- Check how existing role-gated pages in the codebase are structured before creating new ones

**Related Topics**: [Protecting Next.js Routes](../authentication/protecting-nextjs-app-router-routes.md), [Rules of Hooks](../react/rules-of-hooks.md)

---

### 2) Repetitive Fallback Merge Pattern

**Symptom**: A layout component merged server-rendered role flags with live client-side flags using repeated `??` lines. Each new flag required another merge line.

```typescript
// ❌ Before: Repeated fallback pattern — grows with every new flag
const clientFlags = user ? getRoleFlags(user) : null;
const isSuperuser = clientFlags?.isSuperuser ?? serverIsSuperuser;
const isOwner = clientFlags?.isOwner ?? serverIsOwner;
const isAdmin = clientFlags?.isAdmin ?? serverIsAdmin;
// ... if a 4th flag is added, someone must remember to add another line
```

```typescript
// ✅ After: Single destructure with object fallback
const clientFlags = user ? getRoleFlags(user) : null;
const { isSuperuser, isOwner, isAdmin } = clientFlags ?? {
  isSuperuser: serverIsSuperuser,
  isOwner: serverIsOwner,
  isAdmin: serverIsAdmin,
};
```

**Root Cause**: The initial implementation was correct but didn't anticipate scaling. When `clientFlags` is either a complete object or null, per-field fallback is equivalent to a whole-object fallback — but harder to maintain.

**Fix**: Use a single null-coalescing expression on the whole object, then destructure.

**Prevention**:
- When merging two sources of the same shape, prefer whole-object fallback over per-field fallback
- Ask: "If a new field is added to this type, how many places need updating?"

---

### 3) E2E Test Default Labels Assume Wrong Role

**Symptom**: A test helper's default parameter included role-gated nav items, silently assuming admin role for any test that didn't pass explicit labels. Also, a nav item visible to all users was missing from both the default and the non-admin test.

```typescript
// ❌ Before: Default includes admin-only item, misses universal item
async expectNavigation(
  labels = ['Dashboard', 'Agents', 'Analytics', 'Call History', 'Phone Numbers'],
  //        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ admin-only item in default
  //        missing: 'Notifications' (visible to all users)
) { ... }
```

```typescript
// ✅ After: Shared constants, role-aware defaults
const BASE_NAV = ['Dashboard', 'Agents', 'Analytics', 'Call History', 'Notifications'];
const ADMIN_NAV = [...BASE_NAV, 'Phone Numbers'];

async expectNavigation(labels = BASE_NAV) { ... }

// In tests:
// Owner/admin test → expectNavigation(ADMIN_NAV)
// Analyst test     → expectNavigation(BASE_NAV) or expectNavigation() (uses default)
```

**Root Cause**: The default was written when the only tests ran as admin. When analyst tests were added, the default wasn't revisited.

**Fix**: Extract `BASE_NAV` and `ADMIN_NAV` constants. Default to `BASE_NAV` (what everyone sees). Admin tests explicitly pass `ADMIN_NAV`.

**Prevention**:
- Test helper defaults should represent the **least-privileged** user
- When role-gated UI changes, audit test helpers for stale assumptions
- Shared constants keep label lists in sync across test files
- New nav items only need updating in one place

**Related Topics**: [Testing Semantic Behavior](../testing/testing-semantic-behavior-not-implementation.md)

---

### 4) Conditional Spread for Nav Items → Skipped (KISS)

**Suggestion**: Three conditional spreads for role-gated nav items could be replaced with a `visible` flag on the nav item type + `.filter()`.

**Why it was skipped**: With only 3 conditional items across two components, adding a `visible` field means changing the `NavItem` interface and both components that consume it. The abstraction doesn't pay for itself yet. The reviewer phrased it as "just thinking out loud" — not a blocking request.

**Decision rule**: Introduce an abstraction when the pattern repeats **enough times that the boilerplate causes real maintenance pain**. Three conditional spreads is not that threshold.

**One-line rule**: Don't build abstractions for "future flexibility" — wait until the pattern actually hurts (YAGNI).

---

## Reusable Rules

Apply these rules in all code reviews and feature development:

1. **Server-first authorization**: Role-gated pages must check roles in a server component before rendering any client component. Never rely on a client-side guard that runs after hooks.
2. **Whole-object fallback over per-field fallback**: When merging two sources of the same shape (one possibly null), use `objectA ?? objectB` and destructure — not `objectA?.x ?? objectB.x` repeated per field.
3. **Least-privilege test defaults**: Test helper defaults should assume the lowest role. Higher-privilege tests pass explicit arguments.
4. **YAGNI on abstractions**: Don't refactor a pattern into an abstraction until the repetition causes real maintenance pain. Three instances is not enough.

## Checklist for Role-Gated Pages

Before submitting a PR that adds or modifies role-gated pages:

- [ ] Page uses a server component wrapper that checks roles before rendering
- [ ] Unauthorized access returns `notFound()` (not an inline error in a client component)
- [ ] No data queries fire for unauthorized users
- [ ] E2e test nav labels include all items visible to the tested role
- [ ] E2e test nav labels exclude items gated behind higher roles
- [ ] Test helper defaults assume the least-privileged role

## Related Documentation

- [Protecting Next.js Routes](../authentication/protecting-nextjs-app-router-routes.md)
- [Rules of Hooks](../react/rules-of-hooks.md)
- [Testing Semantic Behavior](../testing/testing-semantic-behavior-not-implementation.md)
- [Code Quality Rules](../quick-reference/code-quality-rules.md)

---

**Source**: PR Review (RBAC phone-numbers feature)
**Date**: March 2026
**Topics**: RBAC, server-components, next.js, authorization, e2e-testing, KISS, destructuring
