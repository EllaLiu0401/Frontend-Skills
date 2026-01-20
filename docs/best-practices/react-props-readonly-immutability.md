# React Props — Readonly and Immutability

> **TL;DR**: Always mark React component props as `readonly` with `readonly` arrays to enforce immutability at compile time.

## Context

React props should be treated as immutable data — components receive props but should never modify them. TypeScript's `readonly` modifier helps enforce this principle at compile time, preventing accidental mutations that can lead to subtle bugs and unpredictable behavior.

## The Problem

### 1) Props Without Readonly Protection

**Symptom**: Props interface allows modification of both the property reference and array contents.

```typescript
// ❌ Before: Props can be mutated
interface UserListProps {
  users: User[];
  selectedId: string;
}

function UserList({ users, selectedId }: UserListProps) {
  // Nothing prevents these mutations (at compile time):
  users.push({ id: '123', name: 'New User' }); // Modifies parent's data!
  users[0] = { id: '456', name: 'Changed' };   // Unexpected mutation!
  selectedId = 'different-id';                  // Won't work but shouldn't compile
}

// ✅ After: Readonly enforces immutability
interface UserListProps {
  readonly users: readonly User[];
  readonly selectedId: string;
}

function UserList({ users, selectedId }: UserListProps) {
  // TypeScript compile errors:
  users.push(...);      // ❌ Error: Property 'push' does not exist
  users[0] = ...;       // ❌ Error: Index signature is readonly
  selectedId = '...';   // ❌ Error: Cannot assign to 'selectedId'
  
  // Correct approach: create new data
  const updatedUsers = [...users, newUser]; // ✅ Creates new array
}
```

**Root Cause**: Missing TypeScript safeguards that prevent props mutation, which violates React's unidirectional data flow principle.

**Fix**: Add `readonly` modifier to both:
1. The property itself (prevents reassignment)
2. Arrays/objects (prevents mutation of contents)

**Prevention**: 
- Always mark props interfaces with `readonly` modifiers
- For arrays: `readonly items: readonly ItemType[]`
- For objects: Use `Readonly<T>` or `readonly` on each property
- Set up ESLint rules to enforce this pattern

**Related Topics**: [TypeScript](../typescript/), [React Patterns](../react/)

---

## Understanding the Two Levels of Readonly

### Level 1: Property Readonly

```typescript
interface Props {
  readonly items: string[];
  //       ↑
  //       Prevents reassigning the entire property
}

function Component({ items }: Props) {
  items = ['new', 'array']; // ❌ Error: Cannot assign to 'items'
  items.push('new');        // ✅ But this works! (we need level 2)
}
```

### Level 2: Array/Object Content Readonly

```typescript
interface Props {
  readonly items: readonly string[];
  //              ↑
  //              Prevents modifying array contents
}

function Component({ items }: Props) {
  items = ['new'];     // ❌ Error: Cannot assign to 'items'
  items.push('new');   // ❌ Error: Property 'push' does not exist
  items[0] = 'new';    // ❌ Error: Index signature is readonly
  
  // ✅ Correct: Create new arrays
  const newItems = [...items, 'new'];
  const filtered = items.filter(item => item !== 'old');
}
```

### Combined Protection

```typescript
// ✅ Full immutability protection
interface Props {
  readonly user: Readonly<{
    id: string;
    name: string;
    tags: readonly string[];
  }>;
  readonly items: readonly Item[];
  readonly count: number;
}
```

---

## Common Patterns

### Basic Component Props

```typescript
// ✅ Readonly props with primitive and array types
interface CardProps {
  readonly title: string;
  readonly description: string | undefined;
  readonly tags: readonly string[];
  readonly onClick?: () => void;
}

export function Card({ title, description, tags, onClick }: CardProps) {
  return (
    <div onClick={onClick}>
      <h2>{title}</h2>
      {description && <p>{description}</p>}
      <div>
        {tags.map(tag => <span key={tag}>{tag}</span>)}
      </div>
    </div>
  );
}
```

### Complex Nested Objects

```typescript
// ✅ Deep readonly for nested structures
interface DashboardProps {
  readonly config: Readonly<{
    title: string;
    widgets: readonly Readonly<{
      id: string;
      type: 'chart' | 'table' | 'metric';
      data: readonly number[];
    }>[];
  }>;
}

// Alternative: Use utility type for cleaner syntax
type DeepReadonly<T> = {
  readonly [P in keyof T]: T[P] extends (infer U)[]
    ? readonly DeepReadonly<U>[]
    : T[P] extends object
    ? DeepReadonly<T[P]>
    : T[P];
};

interface DashboardProps {
  readonly config: DeepReadonly<Config>;
}
```

### Props with Callbacks

```typescript
// ✅ Callbacks in props
interface FormProps {
  readonly fields: readonly Field[];
  readonly onSubmit: (values: Record<string, unknown>) => void;
  readonly onCancel: () => void;
}

// Callbacks don't need readonly on their parameters (they own them)
function handleSubmit(values: Record<string, unknown>) {
  // This function owns 'values', can modify if needed
  values.timestamp = Date.now();
}
```

---

## Reusable Rules

Apply these rules for all React components:

1. **Always Readonly Props**: Mark all props interface properties as `readonly`
2. **Readonly Arrays**: Use `readonly T[]` for array props (not just `T[]`)
3. **Readonly Objects**: Use `Readonly<T>` or `readonly` on each property for object props
4. **Deep Structures**: Consider `DeepReadonly<T>` utility type for complex nested data
5. **Primitives Too**: Even though reassignment doesn't work in destructured props, `readonly` makes intent explicit
6. **Linter Rules**: Set up ESLint rules like `@typescript-eslint/prefer-readonly` and custom rules for props

---

## TypeScript Configuration

Add these to your `tsconfig.json` for stronger immutability:

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictPropertyInitialization": true
  }
}
```

Consider ESLint rules:

```javascript
// .eslintrc.js
module.exports = {
  rules: {
    '@typescript-eslint/prefer-readonly': 'error',
    '@typescript-eslint/prefer-readonly-parameter-types': 'warn',
  }
};
```

---

## Checklist for Component Props

Before defining props interfaces:

- [ ] All properties marked with `readonly`
- [ ] Arrays use `readonly T[]` not `T[]`
- [ ] Complex objects use `Readonly<T>` or manual readonly properties
- [ ] Consider deep readonly for nested structures
- [ ] Callback parameters don't need readonly (they're owned by the callback)
- [ ] Optional props use `?` or `| undefined`

---

## Related Documentation

- [TypeScript Type Inference](../typescript/type-inference-best-practices.md)
- [React Best Practices](../react/)
- [TypeScript Utility Types](../typescript/)
- [React Rules](../../quick-reference/react-rules.md)

---

## Additional Resources

- [React Docs: Components and Props](https://react.dev/learn/passing-props-to-a-component)
- [TypeScript Handbook: readonly](https://www.typescriptlang.org/docs/handbook/2/objects.html#readonly-properties)
- [Why Immutability Matters in React](https://react.dev/learn/updating-objects-in-state#treat-state-as-read-only)

---

**Date**: January 2026  
**Topics**: TypeScript, React, Props, Immutability, Type Safety, Best Practices
