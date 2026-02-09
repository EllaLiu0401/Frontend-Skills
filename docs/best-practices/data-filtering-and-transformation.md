# Data Filtering and Transformation Best Practices

## Overview
A critical lesson about understanding the full data transformation pipeline before applying filters. Premature filtering can silently drop legitimate data.

## The Problem Pattern

### Anti-Pattern: Filter Before Format
```typescript
// ❌ BAD: Filtering null values before formatting
function aggregateData(items: DataItem[]) {
  const filtered = items
    .filter((item) => item.value !== null)  // Removes nulls
    .map((item) => ({
      name: formatValue(item.value),  // Never sees null values
      count: item.count
    }));
  
  return filtered;
}

function formatValue(value: string | null): string {
  if (!value) return 'Unknown';  // This handling never gets used!
  return value;
}
```

**Problem**: The filter removes null values before `formatValue` can handle them, even though `formatValue` is designed to handle nulls by displaying them as "Unknown".

**Impact**:
- Silent data loss
- Totals don't match expectations
- Legitimate null values (e.g., "No Category") don't appear in results

### Correct Pattern: Format Then Filter (If Needed)
```typescript
// ✅ GOOD: Let the formatter handle all values
function aggregateData(items: DataItem[]) {
  const formatted = items
    .map((item) => ({
      name: formatValue(item.value),  // Handles nulls properly
      count: item.count
    }));
  
  // Only filter if you have a business reason AFTER formatting
  return formatted;
}

function formatValue(value: string | null): string {
  if (!value) return 'Unknown';  // Now this works as intended
  return value;
}
```

## Core Principles

### 1. Understand the Full Pipeline
Before adding a filter, trace the data through the entire transformation:
- What happens to null/undefined values downstream?
- Are there formatters that handle edge cases?
- Will filtering break aggregations or totals?

### 2. Trust Your Formatters
If you have formatting functions that handle null/undefined:
```typescript
function formatCategory(cat: string | null): string {
  switch(cat) {
    case null:
    case undefined:
      return 'Uncategorized';
    default:
      return cat;
  }
}
```

Don't filter nulls before calling the formatter - let it do its job.

### 3. Document Null Handling
```typescript
// Step 1: Map buckets to display format
// Note: formatGroupByValue handles null values (e.g., "Uncategorized", "Unknown")
const rawData = buckets.map((bucket) => ({
  name: formatGroupByValue(bucket.value, dimension),
  count: bucket.count
}));
```

Clear comments prevent future developers from "fixing" working code.

## When to Filter

### Filter Before Processing - Valid Cases
```typescript
// ✅ Filtering malformed/invalid data
const validItems = items.filter(item => 
  item.count >= 0 && !isNaN(item.count)
);

// ✅ Filtering based on business rules
const activeItems = items.filter(item => 
  item.status === 'active'
);
```

### Filter After Formatting - Common Cases
```typescript
// ✅ Filtering after transformation
const displayData = items
  .map(item => ({
    label: formatLabel(item),  // Handles all values
    value: item.value
  }))
  .filter(item => item.value > threshold);  // Filter on actual values
```

## Testing Strategy

Always test edge cases explicitly:

```typescript
describe('aggregateData', () => {
  it('includes null values with proper labels', () => {
    const data = [
      { value: 'category-a', count: 50 },
      { value: null, count: 30 }  // Important test case!
    ];
    
    const result = aggregateData(data);
    
    expect(result).toHaveLength(2);
    expect(result[1]).toMatchObject({
      name: 'Unknown',  // Verify null formatting works
      count: 30
    });
  });
  
  it('totals match input data including nulls', () => {
    const data = [
      { value: 'a', count: 50 },
      { value: null, count: 30 }
    ];
    
    const result = aggregateData(data);
    const total = result.reduce((sum, item) => sum + item.count, 0);
    
    expect(total).toBe(80);  // Must include null values
  });
});
```

## Real-World Implications

### Example: Analytics Dashboard
```typescript
// Scenario: Grouping interactions by assistant
// Some interactions have no assistant assigned (assistantId = null)

// ❌ BAD: Users lose 20% of data
const breakdown = buckets
  .filter(b => b.assistantId !== null)  // Drops unassigned interactions
  .map(b => ({
    name: formatAssistant(b.assistantId),
    count: b.count
  }));
// Result: Total = 80, but database has 100 interactions

// ✅ GOOD: All data preserved
const breakdown = buckets
  .map(b => ({
    name: formatAssistant(b.assistantId),  // Returns "No Assistant" for null
    count: b.count
  }));
// Result: Total = 100, matches database
```

## Code Review Checklist

When reviewing data transformations:

- [ ] Are there filters removing null/undefined values?
- [ ] Do downstream formatters handle null/undefined?
- [ ] Do totals/aggregations include all data?
- [ ] Are there tests for null/undefined inputs?
- [ ] Is null handling documented in comments?

## Key Takeaways

1. **Don't filter prematurely** - Let formatters handle edge cases
2. **Test null values explicitly** - They're legitimate data, not errors
3. **Verify totals** - Filtered data should match expected aggregations
4. **Document intentions** - Comment why nulls are handled a certain way
5. **Trust your formatters** - If they handle nulls, don't filter them out

## Related Patterns

- [Type Inference Best Practices](../typescript/type-inference-best-practices.md)
- [Testing Edge Cases](../testing/testing-semantic-behavior-not-implementation.md)
- [Number Formatting Edge Cases](../javascript/number-formatting-edge-cases.md)

---

**Source**: PR code review - discovered data loss from premature filtering before null-aware formatting
**Last Updated**: 2026-02-05
