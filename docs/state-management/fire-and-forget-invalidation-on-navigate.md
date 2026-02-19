# Fire-and-Forget Cache Invalidation When Navigating

## Context

When a mutation succeeds and the next step is to navigate away (e.g., form → list page), `invalidateQueries` should **not** be awaited. The user is stuck on a page they're about to leave while React Query refetches data they can't even see yet.

## The Anti-Pattern

```typescript
const mutation = useMutation({
  mutationFn: createItem,
  onSuccess: async () => {
    // ❌ Blocks navigation until the refetch completes
    await queryClient.invalidateQueries({ queryKey: ['items'] });
    router.push('/items');
  },
});
```

**What happens:**

1. Mutation succeeds.
2. `invalidateQueries` triggers a background refetch of the `['items']` query.
3. `await` pauses execution until that refetch resolves.
4. The user sits on the form (submit button still spinning) waiting for list data to load — data they cannot see.
5. Only then does `router.push` fire.

## The Fix

```typescript
const mutation = useMutation({
  mutationFn: createItem,
  onSuccess: () => {
    // ✅ Fire-and-forget: marks query as stale, navigates immediately
    void queryClient.invalidateQueries({ queryKey: ['items'] });
    router.push('/items');
  },
});
```

**What happens:**

1. Mutation succeeds.
2. `invalidateQueries` marks `['items']` as stale (refetch starts in background).
3. `router.push` fires immediately — the user navigates without delay.
4. When the list page mounts, React Query sees the stale query and either uses the in-flight refetch or starts a new one.
5. The list page shows its own loading state if the data isn't ready yet.

## Why `void`?

TypeScript (with `@typescript-eslint/no-floating-promises`) requires that returned Promises are either awaited, caught, or explicitly ignored. The `void` operator tells the linter: "I know this returns a Promise, and I'm intentionally not awaiting it."

```typescript
// ❌ Lint error: floating promise
queryClient.invalidateQueries({ queryKey: ['items'] });

// ✅ Explicit fire-and-forget
void queryClient.invalidateQueries({ queryKey: ['items'] });
```

## When to Await vs Fire-and-Forget

| Scenario | Await? | Reason |
|---|---|---|
| Navigating to a different page after mutation | No (`void`) | User leaves immediately; destination page handles its own loading |
| Staying on the same page after mutation | Yes (`await`) | User needs to see the updated data right away |
| Invalidation result affects the next action | Yes (`await`) | e.g., refreshing auth state before a routing guard decides where to redirect |

### Examples

**Fire-and-forget — navigating away:**

```typescript
// Create form → redirect to list page
onSuccess: () => {
  void queryClient.invalidateQueries({ queryKey: ['items'] });
  router.push('/items');
},

// Delete action on a detail page → redirect to list
onSuccess: () => {
  toast.success('Deleted');
  void queryClient.invalidateQueries({ queryKey: ['items'] });
  router.push('/items');
},
```

**Await — staying on the same page:**

```typescript
// Inline edit on a list page — user expects instant update
onSuccess: async () => {
  toast.success('Updated');
  await queryClient.invalidateQueries({ queryKey: ['items'] });
},

// Assign/unassign action within a table
onSuccess: async () => {
  toast.success('Assigned');
  await queryClient.invalidateQueries({ queryKey: ['items'] });
},
```

**Await — invalidation affects routing logic:**

```typescript
// Accept Terms of Service → auth state determines redirect target
onSuccess: async () => {
  await queryClient.invalidateQueries({ queryKey: ['current-user'] });
  router.push('/dashboard'); // routing guard reads refreshed user data
},
```

## Decision Rule

> **If the user is leaving the page, don't make them wait for data they can't see.**

1. Is the next step `router.push()` or `router.replace()`? → **Fire-and-forget** (`void`).
2. Does the user stay on the current page and need to see fresh data? → **Await**.
3. Does the invalidated data affect what happens next (e.g., auth guard)? → **Await**.

## Key Takeaways

- `invalidateQueries()` without `await` still marks the query as stale — React Query handles the refetch when the relevant component mounts.
- Awaiting invalidation before navigating adds unnecessary latency with no user benefit.
- Use the `void` operator to satisfy `no-floating-promises` when intentionally fire-and-forgetting.
- The destination page is responsible for its own loading states — trust the framework.

## Related Patterns

- [PR Comment: KISS + Query Invalidation](pr-comment-kiss-query-invalidation.md) — shared query keys and avoiding premature cache optimization
- [Loading States and Guards](../../loading-states-and-guards.md) — handling loading states on the destination page
