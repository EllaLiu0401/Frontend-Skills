# Query Key Single Source of Truth

## What I learned from this PR comment

When a query key is hardcoded in multiple places (for example: read + update + refetch), future refactors can silently break cache updates.

Even if the app works today, this is a maintainability bug risk.

## Why it matters

- Query libraries depend on exact key matching.
- If one place renames a key and another place is missed, cache sync fails quietly.
- The UI may show stale state after a mutation, especially around navigation guards.

## Simple fix (KISS)

Use one shared constant for the key and reuse it everywhere that touches the same resource.

- Define one exported key constant close to the domain hook/module.
- Reuse the same constant in:
  - data reads
  - `setQueryData` / optimistic updates
  - `invalidateQueries` / `refetchQueries`
  - related tests

This avoids over-engineering while removing drift risk.

## Review checklist

- Is the same query key repeated in multiple files?
- Does mutation success update/refetch with the exact same key used by reads?
- Are tests asserting the shared key (not duplicated literals)?
- Can this be solved with a constant instead of a larger abstraction?

## Example pattern (generic)

```ts
export const USER_QUERY_KEY = ['users', 'me'] as const;
```

Use `USER_QUERY_KEY` in all read/mutate/refetch/test references.
