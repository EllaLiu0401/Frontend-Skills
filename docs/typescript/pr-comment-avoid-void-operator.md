# PR Comment Learning: Avoid the `void` Operator in TypeScript

## Why this matters

Using `void` to silence promises or unused values often hides intent and can reduce maintainability.  
Linters and code quality tools commonly flag this because it makes async behavior less explicit.

## Common anti-patterns

- `void refetch();` in click handlers or effects
- `void someAsyncAction();` to ignore promise results
- `void unusedVariable;` to bypass unused-variable warnings

## Better patterns

### 1) Make async intent explicit in event handlers

Prefer:

```ts
onClick={async () => {
  try {
    await refetch();
  } catch {
    // Keep current UI state and error messaging stable
  }
}}
```

Why:

- Clear that this is async behavior
- Easier to add retry/error UX
- Better readability for reviewers and future maintainers

### 2) Remove non-persisted fields without fake usage

If you need to drop a field from an object before persistence, avoid destructuring into an unused variable just to silence lint.

Prefer:

```ts
const payload = { ...input };
Reflect.deleteProperty(payload, "transientField");
```

Why:

- No unused bindings
- Intent is explicit: this field is intentionally excluded
- Works well with strict lint rules

## Decision checklist

Before using `void`, ask:

- Am I hiding an async side effect that should be `await`ed?
- Am I suppressing a lint warning instead of fixing the real issue?
- Can I make the code intent explicit with a small refactor?

## Quick rule of thumb

If a line starts with `void`, there is usually a clearer alternative:

- For promises: use `async/await` with `try/catch`
- For unused values: refactor data flow so the value is not bound unnecessarily
