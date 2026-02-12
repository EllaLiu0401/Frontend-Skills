# Structural Typing and Redundant Union Types

## What I Learned

In TypeScript, function types are compared by structure, not by parameter names.  
If two function types only differ by parameter name, they are the same type.

This means a union of those two types is redundant and can be simplified.

## Why This Matters in Frontend Code

- Keeps hooks and utility APIs easier to read.
- Reduces mental overhead during PR review.
- Avoids fake flexibility that does not add real type safety.
- Supports KISS: choose the simplest type that expresses intent.

## Before vs After (Generic Example)

### Before

```ts
type ActionA<TInput, TResult> = (input: TInput) => Promise<TResult>;
type ActionB<TInput, TResult> = (options: TInput) => Promise<TResult>;

type Action<TInput, TResult> = ActionA<TInput, TResult> | ActionB<TInput, TResult>;
```

### After

```ts
type Action<TInput, TResult> = (input: TInput) => Promise<TResult>;
```

## Main Logic

1. `ActionA` and `ActionB` are structurally identical.
2. Parameter names (`input` vs `options`) do not affect type identity.
3. The union does not widen behavior or safety.
4. Keep one canonical function type.

## Practical PR Review Checklist

- Are these two types truly different in shape, or just differently named?
- Does this union provide real value, or only noise?
- Can the public API stay just as expressive with one type alias?
- Does simplifying reduce complexity without changing behavior?

## Rule of Thumb

If two TypeScript types are structurally the same, keep one and delete the duplicate.

