# PR Comment Lesson: Use `String.raw` for Regex Pattern Strings

## Context
A PR review comment flagged a regex pattern string with multiple escaped backslashes and suggested using `String.raw`.

## Core Lesson
When a regex pattern must be represented as a plain string (for schemas, config objects, or API contracts), prefer `String.raw` to avoid double escaping.

## Why This Matters
- Improves readability by showing the intended regex directly.
- Reduces mistakes caused by escaping confusion (for example, `\\d` vs `\d`).
- Makes future edits safer and faster during reviews.
- Communicates intent clearly: this value is a raw regex string pattern.

## Before / After Pattern

```typescript
// Before: harder to read because of double escaping
const pattern = "^\\d{4}-\\d{2}-\\d{2}$";

// After: clearer intent and easier maintenance
const pattern = String.raw`^\d{4}-\d{2}-\d{2}$`;
```

## When to Use This
- Regex is required as a **string**, not as a `RegExp` object.
- Pattern contains many backslashes (`\d`, `\w`, boundaries, etc.).
- The value is used in declarative schemas or metadata.

## When Not to Use This
- You can directly use a `RegExp` object where the API accepts it.
- The string has no escaping complexity and `String.raw` adds no clarity.

## Practical Checklist
- Is this regex stored as a string for framework/schema requirements?
- Does escaping reduce readability?
- Would `String.raw` make the pattern easier to verify in code review?

If all are yes, use `String.raw`.

## One-Line Rule
For regex **string** patterns with backslashes, default to `String.raw` for clarity and maintainability.

---

_Last updated: 2026-02-12_
