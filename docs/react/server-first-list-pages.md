# Server-First List Pages in Next.js App Router

## Overview

List and overview pages should be Server Components by default. When a page only needs a small piece of client interactivity (e.g., a delete button with a confirmation dialog), extract that interaction into a small Client Component child rather than making the entire page a Client Component.

This eliminates the "flash-of-spinner" — the brief loading state users see while data is fetched client-side — and gives an instant first paint with real content.

## When to Use

✅ **Use this pattern when**:

- A page primarily displays a list of items fetched from an API
- The only interactive parts are small actions (delete, toggle, inline edit)
- Navigation (create, edit, view detail) doesn't require client-side state
- You want consistent behavior with other server-rendered pages in the app

## When to Avoid

❌ **Avoid this pattern when**:

- The page has heavy client interactivity throughout (forms, drag-and-drop, modals controlling the whole page)
- Data depends on client-only state (e.g., user input, browser APIs like geolocation)
- The page is a form page where the entire content is interactive

## The Problem

### ❌ Bad: Entire List Page as Client Component

```typescript
'use client';

import { useState } from 'react';
import { useTranslations } from 'next-intl';
import { useRouter } from 'next/navigation';
import { Loading, Button, Card } from '@/components/ui';
import { useDataFetch } from '@/hooks/useDataFetch';
import { useDeleteMutation } from '@/hooks/useDeleteMutation';

export default function ItemsPage() {
  const t = useTranslations('items');
  const router = useRouter();
  const { data, isLoading, error } = useDataFetch('/api/items');
  const deleteMutation = useDeleteMutation();

  // Loading spinner flashes on every page visit
  if (isLoading) {
    return <Loading size="lg" />;
  }

  return (
    <PageLayout title={t('title')}>
      <Button onClick={() => router.push('/items/new')}>
        {t('create')}
      </Button>

      {data?.items.map((item) => (
        <Card key={item.id} title={item.name}>
          <Button onClick={() => router.push(`/items/${item.id}`)}>
            {t('edit')}
          </Button>
          <Button onClick={() => deleteMutation.mutate(item.id)}>
            {t('delete')}
          </Button>
        </Card>
      ))}
    </PageLayout>
  );
}
```

**Issues**:

- Full page is a Client Component — larger JS bundle shipped to browser
- Loading spinner flashes before content appears (no instant first paint)
- `router.push` used for simple navigation that could be `<Link>`
- `useTranslations` used where `getTranslations` (server) would suffice
- Pattern inconsistency with other server-rendered pages

## The Solution

### ✅ Good: Server Component Page + Small Client Child

**Step 1: Page as Server Component** — fetch data server-side, use `Link` for navigation

```typescript
// page.tsx (Server Component — no 'use client')
import Link from 'next/link';
import { getTranslations, getFormatter } from 'next-intl/server';
import { Button, Card, EmptyState, PageLayout } from '@/components/ui';
import { listItems } from '@/api/actions';
import { ItemDeleteButton } from '@/components/items/ItemDeleteButton';

export default async function ItemsPage() {
  const t = await getTranslations('items');
  const format = await getFormatter();

  const result = await listItems();

  if (result.error) {
    return (
      <PageLayout title={t('title')}>
        <Alert type="error">{t('fetchError')}</Alert>
      </PageLayout>
    );
  }

  const items = result.data.items;

  return (
    <PageLayout
      title={t('title')}
      actions={
        <Link href="/items/new">
          <Button tone="primary">{t('create')}</Button>
        </Link>
      }
    >
      {items.length === 0 && (
        <EmptyState title={t('noItems')} />
      )}

      {items.map((item) => (
        <Card key={item.id} title={item.name}>
          <Link href={`/items/${item.id}`}>
            <Button tone="ghost">{t('edit')}</Button>
          </Link>
          {/* Only the delete button needs client-side JS */}
          <ItemDeleteButton itemId={item.id} itemName={item.name} />
        </Card>
      ))}
    </PageLayout>
  );
}
```

**Step 2: Small Client Component** — only the interactive piece

```typescript
// ItemDeleteButton.tsx
'use client';

import { useCallback } from 'react';
import { useRouter } from 'next/navigation';
import { useTranslations } from 'next-intl';
import { Button, toast } from '@/components/ui';
import { deleteItem } from '@/api/actions';
import { useDeleteMutation } from '@/hooks/useDeleteMutation';
import { useConfirm } from '@/hooks/useConfirm';

interface ItemDeleteButtonProps {
  readonly itemId: string;
  readonly itemName: string;
}

export function ItemDeleteButton({ itemId, itemName }: ItemDeleteButtonProps) {
  const t = useTranslations('items');
  const router = useRouter();
  const confirm = useConfirm();

  const deleteMutation = useDeleteMutation(deleteItem, {
    onSuccess: () => {
      toast.success(t('deleted'));
      router.refresh(); // Re-fetches the server component data
    },
  });

  const handleDelete = useCallback(async () => {
    const confirmed = await confirm({
      title: t('deleteConfirmTitle'),
      message: t('deleteConfirmMessage', { name: itemName }),
    });
    if (confirmed) {
      deleteMutation.mutate({ id: itemId });
    }
  }, [confirm, t, itemName, itemId, deleteMutation.mutate]);

  return (
    <Button
      tone="ghost"
      className="text-error"
      onClick={() => { handleDelete().catch(() => {}); }}
    >
      {t('delete')}
    </Button>
  );
}
```

## Key Techniques

### 1. Server-Side i18n

```typescript
// ❌ Client: ships i18n runtime to browser
'use client';
import { useTranslations } from 'next-intl';
const t = useTranslations('items');

// ✅ Server: translations resolved at build/request time
import { getTranslations } from 'next-intl/server';
const t = await getTranslations('items');
```

### 2. Link Instead of router.push for Navigation

```typescript
// ❌ Requires 'use client' just for navigation
const router = useRouter();
<Button onClick={() => router.push('/items/new')}>Create</Button>

// ✅ Works in Server Components, no JS needed
import Link from 'next/link';
<Link href="/items/new"><Button>Create</Button></Link>
```

### 3. router.refresh() for Post-Mutation Data Refresh

After a client-side mutation (delete, update), call `router.refresh()` to re-execute the server component and fetch fresh data — no need for React Query cache invalidation.

```typescript
const router = useRouter();

const mutation = useMutation(deleteItem, {
  onSuccess: () => {
    router.refresh(); // Server component re-runs, list updates
  },
});
```

### 4. Server-Side Error Handling

Handle API errors at the server level rather than with client-side error states:

```typescript
const result = await listItems();

if (result.error) {
  return <Alert type="error">{t('fetchError')}</Alert>;
}

// TypeScript narrows: result.data is guaranteed here
const items = result.data.items;
```

## Common Pitfalls

### Pitfall 1: Making the Page Client-Side Just for useTranslations

**Problem**: Using `useTranslations` (client hook) forces the entire page to be `'use client'`.

❌ **Bad**:
```typescript
'use client';
import { useTranslations } from 'next-intl';
```

✅ **Good**:
```typescript
import { getTranslations } from 'next-intl/server';
const t = await getTranslations('items');
```

### Pitfall 2: Using router.push for Static Navigation

**Problem**: Using `useRouter().push()` for links that don't depend on dynamic state.

❌ **Bad**:
```typescript
'use client';
const router = useRouter();
<Button onClick={() => router.push('/items/new')}>Create</Button>
```

✅ **Good**:
```typescript
<Link href="/items/new"><Button>Create</Button></Link>
```

### Pitfall 3: Forgetting TypeScript Narrowing After Error Guard

After checking `result.error` and returning early, TypeScript narrows the type — `result.data` is guaranteed non-nullable. Don't use optional chaining or nullish coalescing on narrowed values.

❌ **Bad** (ESLint will flag):
```typescript
if (result.error) return <ErrorUI />;
const items = result.data?.items ?? []; // Unnecessary ?. and ??
```

✅ **Good**:
```typescript
if (result.error) return <ErrorUI />;
const items = result.data.items; // TypeScript knows data exists
```

## Decision Checklist

When deciding if a page should be a Server Component, ask:

- [ ] Does the page primarily display data? → **Server Component**
- [ ] Is the only interactivity a small action (delete, toggle)? → **Extract client child**
- [ ] Is navigation the only "interactive" part? → **Use `Link`, stay server**
- [ ] Does every other similar page use Server Components? → **Match the pattern**
- [ ] Does it use `useTranslations` but no other client hooks? → **Switch to `getTranslations`**

## Benefits

- **Instant first paint**: No loading spinner; HTML arrives with real content
- **Smaller JS bundle**: Only the delete button ships client-side JavaScript
- **Pattern consistency**: All list pages behave the same way
- **Better SEO**: Content is in the initial HTML response
- **Simpler code**: Less client state management, fewer hooks

## Trade-offs

- **No automatic refetch**: Server data is a snapshot; use `router.refresh()` after mutations
- **Slightly more files**: One extra component file for the client child
- **Error handling style**: Server errors must be handled with return values, not try/catch in hooks

## Related Patterns

- [Implicit vs Explicit Client Components](implicit-vs-explicit-client-components.md) - When and why to add `'use client'`
- [Loading States and Guards](../../loading-states-and-guards.md) - Patterns for when client-side loading is appropriate

## References

- [Next.js Server and Client Composition Patterns](https://nextjs.org/docs/app/building-your-application/rendering/composition-patterns)
- [Next.js Data Fetching in Server Components](https://nextjs.org/docs/app/building-your-application/data-fetching)
- [next-intl Server Components Guide](https://next-intl-docs.vercel.app/docs/environments/server-client-components)

---

**Tags**: `#nextjs` `#react` `#server-components` `#performance` `#architecture` `#code-review`

**Date Added**: 2026-02-19

**Difficulty**: Intermediate
