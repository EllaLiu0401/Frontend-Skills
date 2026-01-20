# [Pattern Name]

## Overview

Provide a concise description of what this pattern is and what problem it solves.

Example:
"The custom hook pattern extracts reusable stateful logic into a dedicated function that can be shared across multiple components."

## When to Use

Describe the scenarios where this pattern is most appropriate:

✅ **Use this pattern when**:
- Specific condition or requirement 1
- Specific condition or requirement 2
- Specific condition or requirement 3

Example:
- Multiple components need the same stateful logic
- You want to separate concerns and keep components focused
- The logic is complex enough to warrant extraction

## When to Avoid

Describe scenarios where this pattern is NOT appropriate:

❌ **Avoid this pattern when**:
- Specific anti-condition 1
- Specific anti-condition 2
- Specific anti-condition 3

Example:
- The logic is used in only one place
- The abstraction adds more complexity than it removes
- A simpler pattern would suffice

## Implementation

Show a complete, working example of the pattern:

```typescript
// Main implementation example
import { useState, useEffect } from 'react';

/**
 * Custom hook for data fetching with loading and error states
 */
function useDataFetch<T>(url: string) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    let cancelled = false;

    async function fetchData() {
      try {
        setLoading(true);
        const response = await fetch(url);
        
        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }
        
        const json = await response.json();
        
        if (!cancelled) {
          setData(json);
          setError(null);
        }
      } catch (e) {
        if (!cancelled) {
          setError(e instanceof Error ? e : new Error('Unknown error'));
          setData(null);
        }
      } finally {
        if (!cancelled) {
          setLoading(false);
        }
      }
    }

    fetchData();

    // Cleanup function to prevent state updates if unmounted
    return () => {
      cancelled = true;
    };
  }, [url]);

  return { data, loading, error };
}

export default useDataFetch;
```

### Usage Example

Show how to use the pattern in a real scenario:

```typescript
// Component using the pattern
function UserProfile({ userId }: { userId: string }) {
  const { data, loading, error } = useDataFetch<User>(
    `/api/users/${userId}`
  );

  if (loading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;
  if (!data) return <NotFound />;

  return (
    <div>
      <h1>{data.name}</h1>
      <p>{data.email}</p>
    </div>
  );
}
```

## Variations

Describe alternative approaches or variations of the pattern:

### Variation 1: [Variation Name]

Brief description of the variation and when to use it.

```typescript
// Code example of variation
```

### Variation 2: [Variation Name]

Brief description of another variation.

```typescript
// Code example of variation
```

## Common Pitfalls

List common mistakes when implementing or using this pattern:

### Pitfall 1: [Pitfall Name]

**Problem**: Describe what developers commonly do wrong.

❌ **Bad**:
```typescript
// Example of the mistake
```

✅ **Good**:
```typescript
// Example of the correct approach
```

### Pitfall 2: [Pitfall Name]

**Problem**: Another common mistake.

❌ **Bad**:
```typescript
// Example of the mistake
```

✅ **Good**:
```typescript
// Example of the correct approach
```

## Real-World Scenarios

Provide concrete examples of where this pattern has been useful:

### Scenario 1: [Scenario Name]

Describe a real situation where this pattern solved a problem:
- What was the challenge?
- How did the pattern help?
- What was the outcome?

### Scenario 2: [Scenario Name]

Another example from experience.

## Benefits

List the advantages of using this pattern:

- **Benefit 1**: Explanation of advantage
- **Benefit 2**: Explanation of advantage
- **Benefit 3**: Explanation of advantage

## Trade-offs

Be honest about the costs or limitations:

- **Trade-off 1**: What you give up or need to consider
- **Trade-off 2**: Potential downsides or complexity added
- **Trade-off 3**: Performance or maintainability considerations

## Best Practices

Provide guidelines for implementing this pattern effectively:

1. **Practice 1**: Specific actionable advice
2. **Practice 2**: Another guideline
3. **Practice 3**: Additional best practice

## Testing Considerations

Explain how to test code using this pattern:

```typescript
// Example test
import { renderHook, waitFor } from '@testing-library/react';
import useDataFetch from './useDataFetch';

describe('useDataFetch', () => {
  it('should fetch data successfully', async () => {
    const { result } = renderHook(() => 
      useDataFetch('/api/test')
    );

    expect(result.current.loading).toBe(true);

    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    expect(result.current.data).toBeDefined();
    expect(result.current.error).toBeNull();
  });
});
```

## Related Patterns

Link to related patterns or concepts:

- [Pattern 1](../path/to/pattern.md) - How it relates
- [Pattern 2](../path/to/pattern.md) - How it compares
- [External resource](https://example.com) - Additional reading

## References

List any external resources, articles, or documentation:

- [React Hooks Documentation](https://react.dev/reference/react)
- [Relevant article or blog post](https://example.com)

---

**Tags**: `#pattern` `#react` `#hooks` (for your own reference)

**Date Added**: YYYY-MM-DD

**Difficulty**: Beginner | Intermediate | Advanced
