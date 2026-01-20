# Type Inference Best Practices

> **TL;DR**: Let TypeScript infer types whenever possible. Only add explicit annotations when inference fails or for public API boundaries.

## The Problem

Over-annotating types makes code verbose and harder to maintain. TypeScript's inference engine is powerful—use it.

## When to Let TypeScript Infer

### ✅ Hook State with Simple Initial Values

```typescript
// ❌ Redundant: TypeScript already knows the type
const [count, setCount] = useState<number>(0);
const [name, setName] = useState<string>('');
const [isOpen, setOpen] = useState<boolean>(false);

// ✅ Let TypeScript infer from initial value
const [count, setCount] = useState(0);        // inferred as number
const [name, setName] = useState('');         // inferred as string  
const [isOpen, setOpen] = useState(false);    // inferred as boolean
```

### ✅ Variables with Initialization

```typescript
// ❌ Redundant annotations
const items: string[] = ['a', 'b', 'c'];
const config: { timeout: number } = { timeout: 5000 };

// ✅ Inferred automatically
const items = ['a', 'b', 'c'];              // string[]
const config = { timeout: 5000 };           // { timeout: number }
```

### ✅ Function Return Types (Internal Functions)

```typescript
// ❌ Explicit return type not needed for simple functions
const double = (x: number): number => x * 2;

// ✅ Return type inferred from implementation
const double = (x: number) => x * 2;        // (x: number) => number
```

**Note**: Keep explicit return types for public APIs and exported functions—they improve documentation and catch errors.

### ✅ Array Methods

```typescript
const numbers = [1, 2, 3, 4, 5];

// ❌ Redundant callback typing
const doubled = numbers.map<number>((n: number): number => n * 2);

// ✅ All types inferred from context
const doubled = numbers.map(n => n * 2);    // number[]
```

## When to Add Explicit Types

### ❌ Don't Rely on Inference: Union Types or Nullable State

```typescript
// ❌ Wrong: TypeScript infers `null` only
const [user, setUser] = useState(null);
setUser({ id: 1, name: 'Alice' }); // ❌ Type error!

// ✅ Correct: Explicit union type needed
const [user, setUser] = useState<User | null>(null);
```

### ❌ Don't Rely on Inference: Empty Arrays

```typescript
// ❌ Wrong: Inferred as never[]
const [items, setItems] = useState([]);
setItems([{ id: 1 }]); // ❌ Type error!

// ✅ Correct: Explicit type parameter
const [items, setItems] = useState<Item[]>([]);
```

### ❌ Don't Rely on Inference: Function Parameters

```typescript
// ❌ Parameters cannot be inferred
const greet = (name) => `Hello, ${name}`;  // ❌ Implicit 'any'

// ✅ Always type parameters
const greet = (name: string) => `Hello, ${name}`;
```

### ❌ Don't Rely on Inference: Public API Boundaries

```typescript
// ❌ Missing return type on exported function
export const calculatePrice = (items: Item[]) => {
  return items.reduce((sum, item) => sum + item.price, 0);
};

// ✅ Explicit return type for exported functions
export const calculatePrice = (items: Item[]): number => {
  return items.reduce((sum, item) => sum + item.price, 0);
};
```

## Decision Tree

```
Do you need to add a type annotation?
│
├─ Is it a function parameter?
│  └─ YES → Always add type
│
├─ Is it an empty array/object or null that will hold different types later?
│  └─ YES → Add explicit type
│
├─ Is it a public API (exported function, component props)?
│  └─ YES → Add explicit types for documentation
│
├─ Does TypeScript infer the wrong type?
│  └─ YES → Add explicit type or type assertion
│
└─ Otherwise
   └─ NO → Let TypeScript infer
```

## Common Mistakes

### Mistake 1: Over-Typing Hooks

```typescript
// ❌ Redundant
const [loading, setLoading] = useState<boolean>(false);
const [error, setError] = useState<string | null>(null);

// ✅ Inferred correctly
const [loading, setLoading] = useState(false);
const [error, setError] = useState<string | null>(null); // Explicit needed for union
```

### Mistake 2: Typing Obvious Variables

```typescript
// ❌ Redundant
const API_URL: string = 'https://api.example.com';
const MAX_RETRIES: number = 3;

// ✅ Inferred
const API_URL = 'https://api.example.com';
const MAX_RETRIES = 3;
```

### Mistake 3: Not Typing Union/Nullable Cases

```typescript
// ❌ TypeScript infers too narrow
const [data, setData] = useState(null);  // Type: null (can never change!)

// ✅ Explicit union type
const [data, setData] = useState<Data | null>(null);
```

## Best Practices Summary

1. **Default to inference**: Let TypeScript figure out types from values
2. **Always type parameters**: Function parameters cannot be inferred
3. **Type union cases**: Explicitly annotate nullable and union types
4. **Type public APIs**: Add return types to exported functions and component props
5. **Trust inference for simple cases**: Primitives, arrays with values, object literals
6. **Use inference for maintainability**: Less annotation = easier refactoring

## Tooling

Enable strict TypeScript settings in `tsconfig.json`:

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true
  }
}
```

These settings will force you to add types only where necessary and catch inference gaps.

## Related Resources

- [TypeScript Handbook: Type Inference](https://www.typescriptlang.org/docs/handbook/type-inference.html)
- [PR-0161: Redundant Generic Type Parameters](../best-practices/pr-0161-dashboard-foundation.md)

---

**Key Takeaway**: Write less TypeScript to get better TypeScript. Use inference as the default, explicit types as the exception.
