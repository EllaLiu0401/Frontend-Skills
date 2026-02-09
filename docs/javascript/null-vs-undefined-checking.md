# Null vs Undefined: Checking Optional Values

## Problem

When checking if an optional property has a value, using `!== null` only checks for `null`, **not `undefined`**.

```typescript
interface Data {
  value?: string; // Optional property
}

const data1: Data = { value: "hello" };
const data2: Data = { value: null };
const data3: Data = {}; // value is undefined

// ❌ WRONG - Only checks for null
data1.value !== null; // ✅ true
data2.value !== null; // ✅ false
data3.value !== null; // ⚠️ TRUE (bug! undefined !== null)
```

## Solution

Use `!= null` (loose equality) to check for **both null and undefined**:

```typescript
// ✅ CORRECT - Checks for both null and undefined
data1.value != null; // ✅ true
data2.value != null; // ✅ false
data3.value != null; // ✅ false (fixed!)
```

## Why This Happens

### Optional Properties in TypeScript

```typescript
interface User {
  email?: string; // Type: string | undefined
}

const user1: User = { email: "test@example.com" }; // email = "test@example.com"
const user2: User = { email: null }; // email = null
const user3: User = {}; // email = undefined
```

Optional properties (`property?: type`) can be:

- The actual type value
- `null` (if explicitly set)
- `undefined` (if omitted)

### JavaScript Equality

```javascript
null === undefined; // false (strict equality)
null == undefined; // true  (loose equality)
```

## Best Practices

### ✅ Do: Use `!= null` for optional values

```typescript
function processUser(user: { name?: string }) {
  // Check if name exists (exclude both null and undefined)
  if (user.name != null) {
    console.log(user.name.toUpperCase()); // Safe to use
  }
}
```

### ✅ Do: Use `!== undefined` when specifically checking undefined

```typescript
function hasProperty(obj: object, key: string): boolean {
  // Specifically check if property exists (vs being null)
  return obj[key] !== undefined;
}
```

### ❌ Don't: Use `!== null` for optional properties

```typescript
interface Config {
  timeout?: number;
}

function setupConfig(config: Config) {
  // ❌ WRONG - misses undefined case
  if (config.timeout !== null) {
    // This runs even when timeout is undefined!
  }

  // ✅ CORRECT
  if (config.timeout != null) {
    // Only runs when timeout has a real value
  }
}
```

## Common Patterns

### Filtering Arrays

```typescript
// Remove null and undefined from array
const values: (string | null | undefined)[] = ["a", null, "b", undefined, "c"];

// ✅ CORRECT
const filtered = values.filter((v) => v != null);
// Result: ['a', 'b', 'c']

// ❌ WRONG
const wrongFilter = values.filter((v) => v !== null);
// Result: ['a', 'b', undefined, 'c'] - still has undefined!
```

### Checking Object Properties

```typescript
interface ApiResponse {
  data?: {
    items: string[];
  };
}

function processResponse(response: ApiResponse) {
  // ✅ CORRECT - checks for both null and undefined
  if (response.data != null) {
    return response.data.items.length;
  }
  return 0;
}
```

### Array.some() Validation

```typescript
interface Item {
  groupValue?: string;
}

const items: Item[] = [
  { groupValue: "group-1" },
  { groupValue: null },
  {}, // groupValue is undefined
];

// Check if any item has a valid groupValue
const hasValidGroup = items.some((item) => item.groupValue != null);
// Result: true (only the first item has a value)

// ❌ WRONG
const wrongCheck = items.some((item) => item.groupValue !== null);
// Result: true (incorrectly includes undefined as "valid")
```

## ESLint Rules

Some codebases enforce `!= null` vs `!== null` patterns:

```json
{
  "rules": {
    "eqeqeq": ["error", "always", { "null": "ignore" }]
  }
}
```

This allows `== null` and `!= null` (for checking both null/undefined) while requiring `===` for all other comparisons.

## Quick Reference

| Check                 | Meaning                           | Use When                            |
| --------------------- | --------------------------------- | ----------------------------------- |
| `value != null`       | Not null AND not undefined        | Checking optional properties        |
| `value !== null`      | Not null (but could be undefined) | Specifically excluding null         |
| `value !== undefined` | Not undefined (but could be null) | Specifically excluding undefined    |
| `value == null`       | Is null OR undefined              | Checking if value is missing        |
| `value === null`      | Is exactly null                   | Specifically checking for null      |
| `value === undefined` | Is exactly undefined              | Specifically checking for undefined |

## Summary

- **Optional properties** (`property?: type`) can be `undefined` when omitted
- Use `!= null` to exclude **both** `null` and `undefined`
- Use `!== null` only when you specifically want to allow `undefined`
- This is one of the few cases where loose equality (`==`, `!=`) is preferred over strict equality
