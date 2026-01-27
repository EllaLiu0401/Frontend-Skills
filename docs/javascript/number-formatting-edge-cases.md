# Number Formatting: Handling Rounding Edge Cases

**Category**: JavaScript / TypeScript Utilities  
**Difficulty**: Intermediate  
**Tags**: `edge-cases`, `rounding`, `number-formatting`, `bugs`, `testing`

---

## Problem Statement

When formatting large numbers with K/M suffixes (thousands/millions), rounding can produce misleading output. For example, `999,999` might display as `"1000.0K"` instead of the more appropriate `"1.0M"`.

### Why This Matters

- **User Experience**: Displaying "1000.0K" is confusing and looks like a bug
- **Consistency**: Numbers should use the most appropriate unit suffix
- **Precision**: Rounding edge cases need careful handling

---

## The Bug Explained

### Original Implementation

```typescript
function formatNumber(value: number): string {
  if (value >= 1_000_000) {
    return `${(value / 1_000_000).toFixed(1)}M`;
  }
  if (value >= 1_000) {
    return `${(value / 1_000).toFixed(1)}K`;  // ❌ BUG HERE
  }
  return value.toString();
}
```

### What Goes Wrong

```typescript
formatNumber(999_999);
// Step 1: value >= 1_000 → true
// Step 2: 999_999 / 1_000 = 999.999
// Step 3: (999.999).toFixed(1) = "1000.0"
// Step 4: Returns "1000.0K" ❌ MISLEADING!
```

### Root Cause

The function checks if the value is >= 1,000,000 BEFORE dividing, but after division and rounding, the result can exceed 1000 in the K range, producing "1000K" or higher.

---

## The Fix

### Corrected Implementation

```typescript
function formatNumber(value: number): string {
  if (value >= 1_000_000) {
    return `${(value / 1_000_000).toFixed(1)}M`;
  }
  
  if (value >= 1_000) {
    const thousands = value / 1_000;
    
    // ✅ Check if rounding would produce 1000K or more
    if (thousands >= 999.95) {
      return `${(value / 1_000_000).toFixed(1)}M`;
    }
    
    return `${thousands.toFixed(1)}K`;
  }
  
  return value.toString();
}
```

### Why 999.95?

When `toFixed(1)` rounds a number:
- `999.94` → `"999.9"` ✅ (stays in K range)
- `999.95` → `"1000.0"` ❌ (should use M)
- `1000.00` → `"1000.0"` ❌ (should use M)

So we check `>= 999.95` to catch all values that would round to 1000 or higher.

---

## Test Coverage

### Comprehensive Test Cases

```typescript
describe('formatNumber', () => {
  it('formats numbers under 1000 as-is', () => {
    expect(formatNumber(500)).toBe('500');
    expect(formatNumber(0)).toBe('0');
    expect(formatNumber(999)).toBe('999');
  });

  it('formats thousands with K suffix', () => {
    expect(formatNumber(3_500)).toBe('3.5K');
    expect(formatNumber(1_000)).toBe('1.0K');
    expect(formatNumber(12_345)).toBe('12.3K');
    expect(formatNumber(999_949)).toBe('999.9K');  // ✅ Edge case
  });

  it('formats millions with M suffix', () => {
    expect(formatNumber(1_500_000)).toBe('1.5M');
    expect(formatNumber(2_000_000)).toBe('2.0M');
    expect(formatNumber(10_234_567)).toBe('10.2M');
  });

  it('handles rounding edge case correctly', () => {
    // Values that would round to 1000.0K should display as M instead
    expect(formatNumber(999_950)).toBe('1.0M');  // ✅ Fixed!
    expect(formatNumber(999_999)).toBe('1.0M');  // ✅ Fixed!
  });

  it('handles exact boundaries', () => {
    expect(formatNumber(1_000_000)).toBe('1.0M');
    expect(formatNumber(1_000)).toBe('1.0K');
  });
});
```

---

## Key Lessons

### 1. Always Test Edge Cases

Edge cases are where bugs hide. For number formatting:
- **Boundaries**: Test values right at thresholds (1000, 1000000)
- **Just Below**: Test values just below thresholds (999, 999999)
- **Rounding Points**: Test values that trigger rounding changes (999.95, 999.949)

### 2. Understand IEEE 754 Rounding

JavaScript uses "round half away from zero" with `toFixed()`:
- `.toFixed(1)` rounds 0.95 → 1.0
- This affects boundary calculations
- Always account for rounding behavior in your logic

### 3. Write Tests First

When fixing a bug:
1. **Reproduce**: Write a failing test that demonstrates the bug
2. **Fix**: Implement the minimal fix
3. **Verify**: Ensure all tests pass
4. **Expand**: Add more edge case tests

### 4. Consider Alternatives

Other approaches to avoid this issue:

**Option A: Floor Instead of Round**
```typescript
// Round down instead of to nearest
Math.floor(value / 1000 * 10) / 10
```

**Option B: Pre-check with Threshold**
```typescript
// Adjust threshold to account for rounding
if (value >= 999_950) return formatMillions(value);
```

**Option C: Use Math.round**
```typescript
// More explicit control
const rounded = Math.round(value / 1000 * 10) / 10;
if (rounded >= 1000) return formatMillions(value);
```

---

## Pattern: Threshold Guards

### General Pattern

When formatting with unit suffixes:

```typescript
function formatWithUnits(value: number, units: Unit[]): string {
  // Sort units by threshold descending
  for (const unit of units) {
    if (value >= unit.threshold) {
      const scaled = value / unit.divisor;
      
      // ✅ Check if rounding would overflow to next unit
      const roundingThreshold = unit.nextUnit 
        ? calculateRoundingThreshold(unit, decimals)
        : Infinity;
        
      if (scaled >= roundingThreshold) {
        continue;  // Try next larger unit
      }
      
      return `${scaled.toFixed(decimals)}${unit.symbol}`;
    }
  }
  return value.toString();
}
```

### Reusable Helper

```typescript
/**
 * Calculate the threshold where rounding would overflow to next unit
 * @param decimals - Number of decimal places for display
 * @returns The threshold value
 */
function getRoundingOverflowThreshold(decimals: number): number {
  // For decimals=1: 999.95 rounds to 1000.0
  // For decimals=2: 999.995 rounds to 1000.00
  return 1000 - (5 / Math.pow(10, decimals + 1));
}

// Usage
const thousands = value / 1_000;
if (thousands >= getRoundingOverflowThreshold(1)) {
  // Use next unit (M)
}
```

---

## Related Patterns

- **[Number Formatting with Intl](./intl-number-formatting.md)** - Using browser APIs
- **[Locale-Aware Formatting](./locale-formatting.md)** - i18n considerations
- **[Unit Testing Numerical Code](../testing/numerical-testing.md)** - Floating point concerns

---

## Common Pitfalls

### ❌ Pitfall 1: Not Testing Boundaries

```typescript
// Only testing happy path
expect(formatNumber(3500)).toBe('3.5K');  // ✅ Passes
expect(formatNumber(1500000)).toBe('1.5M');  // ✅ Passes

// Missing edge cases
expect(formatNumber(999999)).toBe('1.0M');  // ❌ Would fail with buggy code
```

### ❌ Pitfall 2: Ignoring Floating Point Precision

```typescript
// Dangerous: Floating point comparison
if (thousands === 999.95) { ... }  // Might miss due to precision

// Better: Use threshold range
if (thousands >= 999.95) { ... }  // Catches all overflow cases
```

### ❌ Pitfall 3: Over-Engineering

```typescript
// Don't do this for simple cases
class NumberFormatter {
  private config: FormatterConfig;
  private cache: Map<number, string>;
  private strategy: FormattingStrategy;
  // ... 200 lines of abstraction
}

// Keep it simple when simple works
function formatNumber(value: number): string { ... }
```

---

## Checklist for Number Formatting Functions

When implementing number formatting:

- [ ] Test exact boundary values (1000, 1000000)
- [ ] Test values just below boundaries (999, 999999)
- [ ] Test values that trigger rounding edge cases (999.95, 999.949)
- [ ] Test negative numbers if applicable
- [ ] Test zero and very small numbers
- [ ] Consider locale/internationalization needs
- [ ] Document expected behavior at boundaries
- [ ] Handle overflow to next unit gracefully

---

## Real-World Impact

### Before Fix
```
Dashboard showing: "1000.0K calls made"
User reaction: "Is this a bug? Why not 1M?"
```

### After Fix
```
Dashboard showing: "1.0M calls made"
User reaction: ✅ Clear and professional
```

---

## Further Reading

- [MDN: Number.prototype.toFixed()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/toFixed)
- [MDN: Intl.NumberFormat](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/NumberFormat)
- [IEEE 754 Floating Point](https://en.wikipedia.org/wiki/IEEE_754)
- [JavaScript Number Precision Issues](https://floating-point-gui.de/languages/javascript/)

---

**Last Updated**: January 2026  
**Contributor**: Based on real production bug fix
