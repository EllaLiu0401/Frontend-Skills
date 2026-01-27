# Number Formatting Checklist

Quick reference for implementing and debugging number formatting functions.

---

## Pre-Implementation Checklist

Before writing a number formatter:

- [ ] Define exact requirements (decimal places, units, ranges)
- [ ] Identify all threshold boundaries (e.g., 1K, 1M, 1B)
- [ ] Determine rounding behavior (round, floor, ceil)
- [ ] Consider internationalization needs
- [ ] Document expected behavior at boundaries

---

## Common Edge Cases to Test

### Boundary Values
```typescript
✓ Test exact thresholds: 1000, 1000000, 1000000000
✓ Test just below: 999, 999999, 999999999
✓ Test just above: 1001, 1000001, 1000000001
✓ Test zero and negative numbers
```

### Rounding Edge Cases
```typescript
✓ Test values that round to threshold: 999.95 → 1000.0
✓ Test values just below rounding: 999.949 → 999.9
✓ Test maximum safe integer: Number.MAX_SAFE_INTEGER
```

### Special Cases
```typescript
✓ Test NaN, Infinity, -Infinity
✓ Test very small numbers (< 0.01)
✓ Test floating point precision issues
```

---

## The "1000K Bug" Pattern

### Problem
```typescript
❌ Bug: 999,999 displays as "1000.0K"
```

### Root Cause
```typescript
value / 1000 = 999.999
(999.999).toFixed(1) = "1000.0"
Result: "1000.0K" // Misleading!
```

### Fix Pattern
```typescript
✅ Check if rounding overflows to next unit:

if (value >= 1_000) {
  const scaled = value / 1_000;
  
  // Prevent overflow to next unit
  if (scaled >= 999.95) {  // Will round to 1000+
    return formatNextUnit(value);
  }
  
  return `${scaled.toFixed(1)}K`;
}
```

---

## Quick Decision Guide

### When to round up to next unit?

```typescript
For toFixed(1):  999.95 and above → next unit
For toFixed(2):  999.995 and above → next unit

Formula: threshold = 1000 - (5 / 10^(decimals + 1))
```

### Threshold calculation
```typescript
function getRoundingThreshold(decimals: number): number {
  return 1000 - (5 / Math.pow(10, decimals + 1));
}

getRoundingThreshold(1)  // 999.95
getRoundingThreshold(2)  // 999.995
```

---

## Testing Template

```typescript
describe('formatNumber', () => {
  describe('boundaries', () => {
    it('formats exact thresholds correctly', () => {
      expect(formatNumber(1_000)).toBe('1.0K');
      expect(formatNumber(1_000_000)).toBe('1.0M');
    });
  });

  describe('rounding edge cases', () => {
    it('prevents overflow to next unit', () => {
      expect(formatNumber(999_949)).toBe('999.9K');  // Just under
      expect(formatNumber(999_950)).toBe('1.0M');    // Would round to 1000K
      expect(formatNumber(999_999)).toBe('1.0M');    // Would round to 1000K
    });
  });

  describe('negative numbers', () => {
    it('handles negative values correctly', () => {
      expect(formatNumber(-1_500)).toBe('-1.5K');
      expect(formatNumber(-999_999)).toBe('-1.0M');
    });
  });

  describe('special values', () => {
    it('handles special numeric values', () => {
      expect(formatNumber(0)).toBe('0');
      expect(formatNumber(NaN)).toBe('NaN');
      expect(formatNumber(Infinity)).toBe('Infinity');
    });
  });
});
```

---

## Common Mistakes

### ❌ Mistake 1: Not checking rounding overflow
```typescript
// Bug: Can produce 1000K or higher
if (value >= 1_000) {
  return `${(value / 1_000).toFixed(1)}K`;
}
```

### ❌ Mistake 2: Using === for floating point
```typescript
// Dangerous: May miss due to precision
if (thousands === 999.95) { ... }
```

### ❌ Mistake 3: Forgetting negative numbers
```typescript
// Incomplete: Doesn't handle negatives
if (value >= 1_000_000) { ... }  // What about -1M?
```

### ❌ Mistake 4: Not testing boundaries
```typescript
// Incomplete tests
expect(formatNumber(3500)).toBe('3.5K');     // ✓ Happy path
expect(formatNumber(1500000)).toBe('1.5M');  // ✓ Happy path
// Missing: 999999, 999950, boundaries!
```

---

## Debug Checklist

When debugging number formatting issues:

1. **Reproduce the exact input**
   ```typescript
   console.log('Input:', value);
   console.log('Scaled:', value / divisor);
   console.log('Rounded:', (value / divisor).toFixed(decimals));
   ```

2. **Check boundary conditions**
   - Is the value near a threshold?
   - Does rounding push it over?

3. **Verify test coverage**
   - Do tests cover this specific value?
   - Are boundary cases tested?

4. **Review rounding logic**
   - Is the threshold check correct?
   - Does it account for toFixed() rounding?

---

## Reusable Patterns

### Pattern 1: Threshold Guard
```typescript
function formatWithThresholdGuard(
  value: number,
  divisor: number,
  nextDivisor: number,
  suffix: string,
  decimals: number = 1
): string {
  const scaled = value / divisor;
  const threshold = 1000 - (5 / Math.pow(10, decimals + 1));
  
  if (scaled >= threshold) {
    // Would overflow to next unit
    return formatWithThresholdGuard(
      value,
      nextDivisor,
      nextDivisor * 1000,
      nextSuffix,
      decimals
    );
  }
  
  return `${scaled.toFixed(decimals)}${suffix}`;
}
```

### Pattern 2: Configuration-Driven
```typescript
const units = [
  { threshold: 1e9, divisor: 1e9, suffix: 'B' },
  { threshold: 1e6, divisor: 1e6, suffix: 'M' },
  { threshold: 1e3, divisor: 1e3, suffix: 'K' },
];

function formatNumber(value: number, decimals = 1): string {
  for (let i = 0; i < units.length; i++) {
    if (Math.abs(value) >= units[i].threshold) {
      const scaled = value / units[i].divisor;
      const threshold = 1000 - (5 / Math.pow(10, decimals + 1));
      
      // Check next larger unit if overflow
      if (i > 0 && Math.abs(scaled) >= threshold) {
        continue;
      }
      
      return `${scaled.toFixed(decimals)}${units[i].suffix}`;
    }
  }
  return value.toString();
}
```

---

## When to Use Libraries

Consider using established libraries when:

- [ ] Need locale-aware formatting (Intl.NumberFormat)
- [ ] Need currency formatting
- [ ] Need complex unit systems
- [ ] Team lacks time for comprehensive testing

Popular options:
- `Intl.NumberFormat` (built-in)
- `numeral.js`
- `accounting.js`
- `dinero.js` (for currency)

---

## Related Checklists

- [Testing Numerical Code](./testing-numerical-code.md)
- [Floating Point Gotchas](./floating-point-gotchas.md)
- [Internationalization Checklist](./i18n-checklist.md)

---

**Remember**: Edge cases are where bugs hide. Always test boundaries, rounding behavior, and special values!
