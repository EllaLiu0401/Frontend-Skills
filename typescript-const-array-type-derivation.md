# TypeScript: Deriving Types from Const Arrays & Type Guards

## Problem

When a union type and its runtime validation are defined separately, they can silently drift apart. Adding a new value to the type but forgetting the validation (or vice versa) introduces bugs that the compiler won't catch.

**Bad** ❌ — two separate sources of truth:

```typescript
type Status = 'active' | 'inactive' | 'pending';

function isValidStatus(value: string): boolean {
  return value === 'active' || value === 'inactive';
  // 'pending' is missing — no compiler error!
}
```

## Solution: Const Array as Single Source of Truth

Define the values once in a `const` array, then derive the type from it.

**Good** ✅:

```typescript
const VALID_STATUSES = ['active', 'inactive', 'pending'] as const;
type Status = (typeof VALID_STATUSES)[number];
// = 'active' | 'inactive' | 'pending' — always in sync
```

### How it works

1. `as const` makes the array a readonly tuple of literal types
2. `(typeof VALID_STATUSES)[number]` extracts the union of all element types
3. Adding or removing an element updates both the array and the type automatically

## Runtime Validation with Type Guards

Use a type guard function instead of `as` assertions for safe narrowing.

**Bad** ❌ — `as` assertion bypasses type safety (and many ESLint configs forbid it):

```typescript
function parseStatus(raw: string): Status | null {
  return (VALID_STATUSES as readonly string[]).includes(raw)
    ? (raw as Status)  // unsafe cast
    : null;
}
```

**Good** ✅ — type guard narrows the type safely:

```typescript
function isValidStatus(value: string): value is Status {
  return (VALID_STATUSES as readonly string[]).includes(value);
}

function parseStatus(raw: string): Status | null {
  return isValidStatus(raw) ? raw : null;
  // raw is narrowed to Status after the guard — no assertion needed
}
```

### Why `as readonly string[]` is needed on the array

`VALID_STATUSES` is typed as `readonly ['active', 'inactive', 'pending']`. The `.includes()` method on this tuple expects a parameter of type `'active' | 'inactive' | 'pending'`, but we're passing an arbitrary `string`. Widening to `readonly string[]` allows `.includes()` to accept any string.

This is a safe widening (not a narrowing assertion), so it doesn't violate strict type-assertion lint rules.

## Common Use Cases

### localStorage / external input validation

```typescript
const VALID_THEMES = ['light', 'dark', 'system'] as const;
type Theme = (typeof VALID_THEMES)[number];

function isValidTheme(value: string): value is Theme {
  return (VALID_THEMES as readonly string[]).includes(value);
}

const raw = localStorage.getItem('theme');
const theme: Theme = raw !== null && isValidTheme(raw) ? raw : 'system';
```

### Discriminated union keys

```typescript
const SORT_OPTIONS = ['name', 'date', 'size'] as const;
type SortOption = (typeof SORT_OPTIONS)[number];

// Render options dynamically — always in sync with the type
SORT_OPTIONS.map((option) => <option key={option} value={option} />);
```

### Excluding specific values

```typescript
const ALL_MODES = ['auto', 'manual', 'off'] as const;
type Mode = (typeof ALL_MODES)[number];
type ActiveMode = Exclude<Mode, 'off'>;
// = 'auto' | 'manual'
```

## Anti-Patterns to Avoid

### ❌ Separate type and validation

```typescript
type Role = 'admin' | 'editor' | 'viewer';

// This array has no type relationship with Role
const validRoles = ['admin', 'editor'];  // 'viewer' silently missing
```

### ❌ Duplicating values across files

```typescript
// types.ts
type Color = 'red' | 'blue' | 'green';

// validation.ts
const COLORS = ['red', 'blue', 'green'];  // can drift from type

// ui.ts
const colorOptions = ['red', 'blue', 'green'];  // third copy!
```

**Fix**: One `as const` array, everything else derives from it.

### ❌ Using `as` assertion instead of type guard

```typescript
const parsed = raw as Theme;  // no runtime check, ESLint violation
```

**Fix**: Always use a `value is T` type guard for runtime narrowing.

## Key Takeaways

1. **One array, one type** — `as const` + indexed access keeps them permanently in sync
2. **Type guards over assertions** — `value is T` is compiler-verified; `as T` is not
3. **The array is iterable** — use it in UI rendering, validation, and tests
4. **Safe to widen** — `as readonly string[]` for `.includes()` is a widening, not a narrowing
5. **Zero runtime cost** — types are erased at compile time; the array is the only runtime artifact
