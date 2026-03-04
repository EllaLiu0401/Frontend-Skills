# Full-Stack Field Wiring — Don't Stop at the Database Layer

## What Happened

A new enum column was added to a database table via migration. The generated types were updated, the test factory was updated to include the new field — but the change stopped there. A code review flagged three gaps:

1. **Auth context silently dropped the field** — The middleware already fetched the full row (including the new column) on every request, but only copied a subset of fields into the request context object. The new field was fetched and thrown away.

2. **The shared type package didn't export the new type** — The type was added to an internal types file and re-exported in a barrel file, but the package's `index.ts` (the public entry point) didn't include it. Other packages importing from the shared package couldn't resolve the type, causing ESLint to report it as an "error type that acts as any."

3. **Tests only covered one variant of the enum** — The factory defaulted to one enum value. No test ever created a record with the other value, so accidental type narrowing (e.g., removing a variant from the union) would go undetected.

## Root Cause

When adding a new field to a data model, it's easy to think "I'll wire it through later when I actually use it." But this creates hidden debt:

- **N+1 query risk**: If the field is already being fetched but not passed through the context, every downstream handler that eventually needs it will have to make a separate database query.
- **Stale package builds**: Monorepo build caches (e.g., Turborepo) can mask missing exports. The type exists in source but not in the compiled output, and cached builds won't recompile.
- **Silent type failures**: In TypeScript, an unresolvable imported type becomes `any` (an "error type"). Union types like `SomeErrorType | undefined` silently degrade to `any`, hiding the real issue until a linter catches it.

## Key Lessons

### 1. Wire new fields through the full request chain

When adding a column that's read on every request (e.g., fetched during auth/middleware), pass it through immediately — even if nothing consumes it yet.

```
DB Row → Middleware fetches → Context object → Downstream code
         (selectAll)          (DON'T DROP IT)
```

If the middleware already fetches the field via something like `SELECT *` or `selectAll()`, adding it to the context is a one-liner. Skipping it means someone later will either:
- Re-query the database (N+1 problem)
- Forget to add it and introduce a bug

### 2. The "export chain" must be complete

In a monorepo with shared packages, adding a type requires touching three places:

```
generated-types.ts  →  types.ts (barrel)  →  index.ts (package entry)
   (definition)         (re-export)           (public API)
```

Missing any link in this chain means consumers can't import the type. Build caches make this worse — the package may appear to build successfully while serving stale compiled output that lacks the new type.

**Debugging tip**: If you get `'SomeType' is an 'error' type that acts as 'any'` from ESLint's `@typescript-eslint/no-redundant-type-constituents`, the type import is broken. Check:
- Is the type exported from the package's entry point?
- Is the package dist up to date? Force rebuild: `turbo run build --filter=package-name --force`

### 3. Test all variants of enum/union types

When adding a union type like `'option_a' | 'option_b'`, don't just test with the default value. Add at least one test that uses each variant:

```typescript
// Bad: only tests default, won't catch accidental narrowing
const item = buildItem(); // defaults to 'option_a'

// Good: proves the type system accepts both variants
const itemA = buildItem({ status: 'option_a' });
const itemB = buildItem({ status: 'option_b' });
expect(itemB.status).toBe('option_b');
```

This guards against someone accidentally narrowing the type (e.g., changing the type to only `'option_a'`). The test will fail at compile time if the type no longer accepts `'option_b'`.

### 4. Impersonation/context-switching must mirror all overrides

If your system has a mechanism where one user can act as another (e.g., admin impersonation, org switching), every field that gets set in the normal auth flow must also be overridden in the impersonation flow. Otherwise the impersonating user sees stale data from their own context.

```typescript
// Normal auth flow sets these fields:
context.orgId = org.id;
context.orgName = org.name;
context.brand = org.brand;  // NEW FIELD

// Impersonation MUST also override the new field:
context.orgId = targetOrg.id;
context.orgName = targetOrg.name;
context.brand = targetOrg.brand;  // DON'T FORGET THIS
```

## Checklist: Adding a New Field to a Data Model

- [ ] Migration adds the column with appropriate type, default, and constraints
- [ ] Generated types are regenerated and include the new field
- [ ] Shared type package exports the new type from its entry point (`index.ts`)
- [ ] Shared package is rebuilt (force rebuild if using build caches)
- [ ] Auth middleware / request context includes the new field (if fetched on every request)
- [ ] Impersonation / context-switching also overrides the new field
- [ ] API response schemas include the field (if it should be visible to clients)
- [ ] Test factory defaults to one value, and at least one test uses each other variant
- [ ] Existing tests still pass after the change

## When to Apply

- Any time you add a column to a table that's queried in middleware or auth flows
- Any time you add a new type to a shared package in a monorepo
- Any time you introduce an enum/union type with multiple variants
