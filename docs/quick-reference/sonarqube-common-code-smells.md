# SonarQube Common Code Smells — Quick Fixes

> Three frequently flagged code smells from SonarQube static analysis, with minimal fixes that preserve behavior.

---

## 1. Avoid the `void` Operator (S3735)

**Severity**: Critical | **Quality**: Maintainability

The `void` operator discards an expression's return value. When used on promises, it obscures async intent and makes code harder to reason about.

### Before / After

```typescript
// ❌ SonarQube flags: "Remove this use of the void operator"
onClick={() => {
  void doSomethingAsync(id);
}}

// ✅ Fix: Handle the promise explicitly
onClick={() => {
  doSomethingAsync(id).catch(() => {});
}}

// ✅ Better: Full async/await with error handling
onClick={async () => {
  try {
    await doSomethingAsync(id);
  } catch {
    // error already handled upstream or intentionally swallowed
  }
}}
```

### When to use which fix

| Scenario                                       | Approach                       |
| ---------------------------------------------- | ------------------------------ |
| Fire-and-forget, errors handled elsewhere      | `.catch(() => {})`             |
| Need local error handling or retry logic       | `async/await` with `try/catch` |
| Return value is truly unused and not a promise | Remove the call or refactor    |

### Why `.catch(() => {})` is safe

If the async function already handles errors internally (e.g., shows a toast, logs, etc.), an empty `.catch()` is a lightweight safety net against unhandled rejections. It does **not** change behavior — it simply makes the promise handling explicit.

---

## 2. Remove Redundant JSX Fragments (S6749)

**Severity**: Minor | **Quality**: Maintainability

A React fragment (`<>...</>`) that wraps only a single child element is unnecessary. It adds an extra layer to the virtual DOM tree without any purpose.

### Before / After

```tsx
// ❌ SonarQube flags: "A fragment with only one child is redundant"
return (
  <>
    <PageLayout title="Settings">
      <Content />
    </PageLayout>
  </>
);

// ✅ Fix: Return the single child directly
return (
  <PageLayout title="Settings">
    <Content />
  </PageLayout>
);
```

### When fragments ARE needed

Fragments are valid and necessary when wrapping **multiple** siblings:

```tsx
// ✅ Correct use: multiple siblings need a fragment
return (
  <>
    <Header />
    <Main />
    <Footer />
  </>
);
```

### Quick rule

Count the direct children inside `<>...</>`. If there is exactly **one**, remove the fragment.

---

## 3. Prefer `.includes()` Over `.some()` for Value Existence (S7765)

**Severity**: Minor | **Quality**: Maintainability

When checking if an array contains a specific value, `.includes()` is more readable and idiomatic than `.some()` with an equality callback.

### Before / After

```typescript
const VALID_TYPES = ["call", "alert", "report"] as const;

// ❌ SonarQube flags: "Use .includes() instead of .some() when checking value existence"
function isValidType(value: string): boolean {
  return VALID_TYPES.some((v) => v === value);
}

// ✅ Fix: Use .includes() — same semantics, cleaner code
function isValidType(value: string): boolean {
  return (VALID_TYPES as ReadonlyArray<string>).includes(value);
}
```

### TypeScript gotcha: `ReadonlyArray` type narrowing

When the array is typed as `ReadonlyArray<'call' | 'alert' | 'report'>`, calling `.includes(value)` where `value` is `string` causes a type error — TypeScript expects the argument to match the array's element type.

The fix is to widen the array type at the call site:

```typescript
// ❌ Type error: Argument of type 'string' is not assignable to parameter of type '"call" | "alert" | "report"'
VALID_TYPES.includes(value);

// ✅ Widen to ReadonlyArray<string> so .includes() accepts string
(VALID_TYPES as ReadonlyArray<string>).includes(value);
```

This cast is safe because `.includes()` only **reads** from the array — it never mutates it.

### Why `.includes()` is preferred

| `.some((v) => v === value)`       | `.includes(value)`               |
| --------------------------------- | -------------------------------- |
| Requires a callback function      | Built-in, no callback overhead   |
| Less obvious intent               | Clearly communicates "contains?" |
| More characters, more indirection | Concise and idiomatic            |

Both use strict equality semantics for strings, so the behavior is identical.

---

## Decision Checklist

Before submitting code, scan for these patterns:

- [ ] Any `void somePromise()` → replace with `.catch(() => {})` or `async/await`
- [ ] Any `<>` fragment with a single child → remove the fragment
- [ ] Any `.some((x) => x === value)` → replace with `.includes(value)`

---

## Related

- [Avoid void Operator (PR Comment)](../typescript/pr-comment-avoid-void-operator.md)
- [Redundant Wrapper Lessons](./pr-comment-redundant-wrapper-lessons.md)
- [Code Quality Rules](./code-quality-rules.md)

---

**Tags**: `#sonarqube` `#code-smells` `#typescript` `#react` `#maintainability`

**Date Added**: 2026-02-19

**Source**: SonarQube Static Analysis (S3735, S6749, S7765)
