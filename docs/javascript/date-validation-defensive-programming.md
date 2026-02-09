# Date Validation: Defensive Programming for Date Parsing

## The Problem

When parsing dates from external sources (APIs, user input, etc.), `new Date(string)` can silently fail and return an `Invalid Date` object. Without validation, this propagates through your code and displays "Invalid Date" in the UI—a poor user experience.

```javascript
// ❌ Unsafe: No validation
const date = new Date(apiData.timestamp);
const formatted = date.toLocaleDateString(); // Could be "Invalid Date"
```

## The Issue with Invalid Dates

JavaScript's `Date` constructor returns an "Invalid Date" object (not `null` or `undefined`) when given invalid input:

```javascript
const invalid = new Date("not-a-date");
console.log(invalid); // Invalid Date
console.log(typeof invalid); // "object"
console.log(invalid.toString()); // "Invalid Date"
```

This means:

- You can't use simple truthiness checks (`if (date)`)
- The object exists, but all methods return `NaN` or invalid strings
- It will pass through your code until it reaches display logic

## The Solution: Validate with `Number.isNaN(date.getTime())`

The `.getTime()` method returns `NaN` for invalid dates, which we can check:

```javascript
// ✅ Safe: Always validate before formatting
const dateString = apiData.timestamp;
let formattedDate = "Unknown";

if (dateString) {
  const date = new Date(dateString);

  // Validate the date is actually valid
  if (!Number.isNaN(date.getTime())) {
    formattedDate = date.toLocaleDateString();
  }
}
```

### Why `Number.isNaN()` and not `isNaN()`?

```javascript
// ❌ Global isNaN() has type coercion issues
isNaN("hello"); // true (coerces to NaN)
isNaN(undefined); // true
isNaN({}); // true

// ✅ Number.isNaN() is more precise
Number.isNaN("hello"); // false (not actually NaN)
Number.isNaN(undefined); // false
Number.isNaN(NaN); // true (only true for actual NaN)
```

**ESLint rule**: Most linters enforce `no-restricted-globals` to ban global `isNaN()` in favor of `Number.isNaN()`.

## Real-World Pattern

```javascript
function formatDate(dateString, locale = "en-US", fallback = "N/A") {
  if (!dateString) {
    return fallback;
  }

  const date = new Date(dateString);

  // Guard against invalid dates
  if (Number.isNaN(date.getTime())) {
    console.warn(`Invalid date string: ${dateString}`);
    return fallback;
  }

  return date.toLocaleDateString(locale, {
    year: "numeric",
    month: "short",
    day: "numeric",
  });
}

// Usage
formatDate("2024-01-15T10:30:00Z"); // "Jan 15, 2024"
formatDate("invalid-date"); // "N/A" (+ console warning)
formatDate(null); // "N/A"
formatDate(undefined); // "N/A"
```

## Why This Matters (Defensive Programming)

**Always validate external data**, even from your own APIs:

- API responses can be corrupted in transit
- Backend changes might alter date formats
- Database migrations can introduce bad data
- Third-party integrations may send unexpected formats

### Cost vs. Benefit

**Cost**: One additional check (`~5ms`)  
**Benefit**:

- No "Invalid Date" displayed to users
- Graceful degradation with fallback text
- Easier debugging (can log warnings)
- Production-ready code

## Quick Reference

```javascript
// Pattern for any date formatting in UI components
const formatSafely = (dateString) => {
  if (!dateString) return "Unknown";

  const date = new Date(dateString);
  if (Number.isNaN(date.getTime())) return "Unknown";

  return date.toLocaleDateString();
};
```

## Common Mistakes

### ❌ Mistake 1: Trusting external data

```javascript
// Assumes API always returns valid dates
const label = new Date(apiResponse.date).toLocaleDateString();
```

### ❌ Mistake 2: Only checking for null/undefined

```javascript
// Invalid dates still pass this check
const label = apiResponse.date
  ? new Date(apiResponse.date).toLocaleDateString()
  : "Unknown";
```

### ❌ Mistake 3: Using global `isNaN()`

```javascript
// Fails linting + has coercion issues
if (!isNaN(date.getTime())) {
  /* ... */
}
```

### ✅ Correct: Validate + Use Number.isNaN

```javascript
let label = "Unknown";

if (apiResponse.date) {
  const date = new Date(apiResponse.date);
  if (!Number.isNaN(date.getTime())) {
    label = date.toLocaleDateString();
  }
}
```

## Testing Date Validation

Always test edge cases:

```javascript
describe("formatDate", () => {
  it("formats valid ISO dates", () => {
    expect(formatDate("2024-01-15T10:30:00Z")).toMatch(/Jan.*15.*2024/);
  });

  it("handles invalid date strings gracefully", () => {
    expect(formatDate("not-a-date")).toBe("N/A");
  });

  it("handles null/undefined", () => {
    expect(formatDate(null)).toBe("N/A");
    expect(formatDate(undefined)).toBe("N/A");
  });

  it("handles empty strings", () => {
    expect(formatDate("")).toBe("N/A");
  });
});
```

## Related Concepts

- **Defensive programming**: Don't trust external data
- **Graceful degradation**: Fail safely with fallback values
- **User experience**: Never show raw error messages like "Invalid Date"
- **Type safety**: In TypeScript, consider using branded types for validated dates

## When to Apply This

✅ **Always validate when**:

- Parsing dates from API responses
- Formatting dates for display in UI
- Accepting date strings from user input
- Working with third-party data

❌ **Can skip validation when**:

- Using `new Date()` with no arguments (always valid)
- Date objects created programmatically in your code
- Already validated earlier in the call stack (but document this!)

## Summary

**Key Takeaway**: Never format dates for display without validation. Use `Number.isNaN(date.getTime())` to check validity and provide user-friendly fallback text.

This is a **zero-cost safety check** that prevents bad UX and makes your code production-ready.

---

**Learned from**: PR code review identifying missing date validation in chart data formatting
**Related**: ESLint `no-restricted-globals` rule enforcing `Number.isNaN` over `isNaN`
