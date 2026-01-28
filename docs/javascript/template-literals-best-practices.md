# Template Literals Best Practices

> Avoiding nested template literals and writing cleaner string interpolation code

## Overview

Template literals (backticks) are powerful for string interpolation in JavaScript/TypeScript. However, nesting template literals within template literals creates hard-to-read code that violates the KISS principle (Keep It Simple, Stupid).

**Key Learning**: Nested template literals are a code smell that indicates you should refactor your code for better readability.

---

## What Are Nested Template Literals?

Nested template literals occur when you use a template literal inside another template literal's interpolation expression.

### Example of the Problem

```typescript
// ❌ Bad: Nested template literals
const userId = 123;
const status = "active";

const message = `User ${userId} is ${status ? `currently ${status}` : "unknown"}`;
//                                            ^^^^^^^^^^^^^^^^^^
//                                            Template literal nested inside another
```

This creates two layers of template strings:

1. Outer: `` `User ${userId} is ${...}` ``
2. Inner: `` `currently ${status}` `` (inside the ternary)

---

## Why Avoid Nested Template Literals?

### 1. Readability Issues

```typescript
// ❌ Hard to read - multiple nesting levels
const html = `<div class="${isActive ? `${baseClass} active` : baseClass}">`;

// ✅ Much clearer with intermediate variable
const className = isActive ? `${baseClass} active` : baseClass;
const html = `<div class="${className}">`;
```

### 2. Maintenance Difficulty

```typescript
// ❌ Hard to modify - easy to make bracket/quote mistakes
const result = `${a ? `${b ? `${c}` : "d"}` : "e"}`;

// ✅ Easy to understand and modify
const innerValue = b ? c : "d";
const result = a ? innerValue : "e";
```

### 3. Debugging Challenges

When errors occur, nested template literals make it harder to:

- Identify which part of the string is problematic
- Set breakpoints at the right location
- Understand the flow of data

### 4. Code Review Friction

Reviewers need extra time to parse nested structures, increasing review time and potential for missed issues.

---

## Common Patterns to Avoid

### Pattern 1: Conditional Nested Templates

```typescript
// ❌ Avoid: Nested conditional templates
const greeting = `Hello, ${user ? `${user.name || "Guest"}` : "Stranger"}!`;

// ✅ Better: Extract to variable
const displayName = user?.name || "Guest";
const greeting = user ? `Hello, ${displayName}!` : "Hello, Stranger!";

// ✅ Best: Use a function for complex logic
function getGreeting(user?: User): string {
  if (!user) return "Hello, Stranger!";
  const name = user.name || "Guest";
  return `Hello, ${name}!`;
}
const greeting = getGreeting(user);
```

### Pattern 2: Multi-level Nesting

```typescript
// ❌ Avoid: Multiple nesting levels
const label = `${type === "user" ? `${status === "active" ? `Active User` : `Inactive User`}` : "Guest"}`;

// ✅ Better: Use intermediate variables
const userStatus = status === "active" ? "Active User" : "Inactive User";
const label = type === "user" ? userStatus : "Guest";

// ✅ Best: Use a lookup table or function
const LABEL_MAP = {
  "user-active": "Active User",
  "user-inactive": "Inactive User",
  guest: "Guest",
};

const key = type === "user" ? `user-${status}` : "guest";
const label = LABEL_MAP[key] || "Unknown";
```

### Pattern 3: Nested Templates with Complex Expressions

```typescript
// ❌ Avoid: Complex nested expressions
const summary = `Total: ${items.length > 0 ? `${items.length} item${items.length > 1 ? "s" : ""}` : "No items"}`;

// ✅ Better: Break down into steps
const itemCount = items.length;
const itemLabel = itemCount === 1 ? "item" : "items";
const itemText = itemCount > 0 ? `${itemCount} ${itemLabel}` : "No items";
const summary = `Total: ${itemText}`;

// ✅ Best: Use a function
function formatItemCount(count: number): string {
  if (count === 0) return "No items";
  const label = count === 1 ? "item" : "items";
  return `${count} ${label}`;
}
const summary = `Total: ${formatItemCount(items.length)}`;
```

---

## Refactoring Strategies

### Strategy 1: Extract to Variables

**Before:**

```typescript
const msg = `Status: ${isOnline ? `Online ${isIdle ? "(idle)" : "(active)"}` : "Offline"}`;
```

**After:**

```typescript
const statusDetail = isIdle ? "(idle)" : "(active)";
const onlineStatus = `Online ${statusDetail}`;
const msg = `Status: ${isOnline ? onlineStatus : "Offline"}`;
```

### Strategy 2: Use Helper Functions

**Before:**

```typescript
const display = `${user ? `${user.firstName || ""} ${user.lastName || ""}`.trim() : "Anonymous"}`;
```

**After:**

```typescript
function getUserDisplayName(user?: User): string {
  if (!user) return "Anonymous";
  return `${user.firstName || ""} ${user.lastName || ""}`.trim();
}

const display = getUserDisplayName(user);
```

### Strategy 3: Simplify Logic First

**Before:**

```typescript
const text = `Price: ${discount ? `$${(price * (1 - discount)).toFixed(2)} (${(discount * 100).toFixed(0)}% off)` : `$${price.toFixed(2)}`}`;
```

**After:**

```typescript
function formatPrice(price: number, discount?: number): string {
  if (!discount) return `$${price.toFixed(2)}`;

  const discountedPrice = price * (1 - discount);
  const discountPercent = (discount * 100).toFixed(0);
  return `$${discountedPrice.toFixed(2)} (${discountPercent}% off)`;
}

const text = `Price: ${formatPrice(price, discount)}`;
```

### Strategy 4: Use Array Join for Complex Strings

**Before:**

```typescript
const fullName = `${title ? `${title} ` : ""}${firstName}${middleName ? ` ${middleName}` : ""} ${lastName}${suffix ? `, ${suffix}` : ""}`;
```

**After:**

```typescript
function formatFullName(parts: NameParts): string {
  const nameParts = [
    parts.title,
    parts.firstName,
    parts.middleName,
    parts.lastName,
  ].filter(Boolean);

  const fullName = nameParts.join(" ");
  return parts.suffix ? `${fullName}, ${parts.suffix}` : fullName;
}

const fullName = formatFullName({
  title,
  firstName,
  middleName,
  lastName,
  suffix,
});
```

---

## Testing Context

### When Mocking String Functions

In testing, avoid complex mock implementations:

```typescript
// ❌ Avoid: Complex nested mock
vi.mock("i18n-library", () => ({
  translate: vi.fn(
    () => (key: string, values?: Record<string, string>) =>
      `${key}${values ? `:${JSON.stringify(values)}` : ""}`,
  ),
}));

// ✅ Better: Simple mock (if it's overridden in beforeEach)
vi.mock("i18n-library", () => ({
  translate: vi.fn(() => (key: string) => key),
}));

// ✅ Best: Meaningful mock implementation
vi.mock("i18n-library", () => ({
  translate: vi.fn(() => (key: string, values?: Record<string, string>) => {
    if (!values) return key;
    return Object.entries(values).reduce(
      (str, [k, v]) => str.replace(`{${k}}`, v),
      key,
    );
  }),
}));
```

**Key principle**: Even in tests, code should be readable and maintainable.

---

## Best Practices Summary

### ✅ DO:

1. **Extract to variables** - Break complex interpolations into named variables
2. **Use functions** - Encapsulate complex string logic in pure functions
3. **Keep it flat** - Avoid nesting template literals inside template literals
4. **Name intermediate values** - Give meaningful names to calculation steps
5. **Use helper utilities** - Create reusable string formatting functions

### ❌ DON'T:

1. **Nest templates** - Don't put template literals inside template literals
2. **Complex ternaries in templates** - Extract complex conditional logic
3. **Chain optional expressions** - Break down `user?.profile?.name || 'Unknown'` chains with nested templates
4. **Inline calculations** - Move math operations outside template literals when combined with nesting
5. **Sacrifice readability** - Never optimize for character count at the expense of clarity

---

## When Template Literals Are Perfectly Fine

```typescript
// ✅ Simple interpolation - perfectly fine!
const greeting = `Hello, ${name}!`;
const path = `${baseUrl}/api/users/${userId}`;
const message = `Found ${count} results in ${time}ms`;

// ✅ Single conditional - acceptable
const label = `${count} ${count === 1 ? "item" : "items"}`;
const status = isActive ? `Active since ${date}` : "Inactive";

// ✅ Simple expressions - no problem
const fullName = `${firstName} ${lastName}`;
const coordinate = `(${x}, ${y})`;
```

---

## Rule of Thumb

**If you find yourself nesting template literals:**

1. ❓ Ask: "Can I extract this to a variable?"
2. ❓ Ask: "Should this be a helper function?"
3. ❓ Ask: "Is there a simpler way to express this?"

**If the answer to any is "yes", refactor before committing.**

---

## Code Review Checklist

When reviewing code with template literals:

- [ ] No template literals nested inside template literals
- [ ] Complex string logic is extracted to functions
- [ ] Intermediate calculations have meaningful variable names
- [ ] Conditional logic is easy to follow
- [ ] String formatting is consistent across the codebase

---

## Related Topics

- [Code Quality Rules](../quick-reference/code-quality-rules.md#complexity) - Managing code complexity
- [Number Formatting Edge Cases](./number-formatting-edge-cases.md) - Formatting patterns

---

## References

- **KISS Principle**: Keep It Simple, Stupid - prefer simple, readable solutions
- **DRY Principle**: Don't Repeat Yourself - extract common string patterns
- **Single Responsibility**: Each function/variable should have one clear purpose

---

_Last updated: 2026-01-28_
