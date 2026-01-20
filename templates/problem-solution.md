# [Descriptive Problem Title]

## Problem Description

Clearly describe the issue that was encountered. Be specific about symptoms and what was observed.

**What happened**:
- Describe the observable behavior or bug
- Include error messages if applicable
- Explain the context (e.g., "During PR review", "In production", "While testing")

**Expected behavior**:
- What should have happened instead?

**Impact**:
- How did this affect the application or users?
- Was this a bug, performance issue, UX problem, or maintainability concern?

Example:
"Event handler in a React component was using stale state values, causing incorrect data to be sent to the API. Users reported that clicking 'Save' would sometimes save outdated information instead of their current edits."

## Root Cause Analysis

Explain WHY the problem occurred. Dig into the underlying cause.

### Technical Explanation

Provide a clear explanation of the root cause:

```typescript
// Example showing the problematic code and why it fails
function Component() {
  const [count, setCount] = useState(0);

  // ❌ Problem: This closure captures count value at creation time
  const handleClick = () => {
    setTimeout(() => {
      console.log(count); // This will always log the initial value
    }, 1000);
  };

  return <button onClick={handleClick}>Click me</button>;
}
```

**Why this happens**:
- Explain the mechanism that causes the issue
- Reference relevant concepts (closures, rendering, lifecycle, etc.)
- Use analogies if helpful

Example:
"JavaScript closures capture variables by reference at the time the function is created. When the event handler is defined, it captures the count value from that render. Even though count updates in later renders, the old event handler still references the old value."

### Contributing Factors

List any additional factors that made this problem more likely or harder to spot:

- Factor 1: Why this contributed to the issue
- Factor 2: Another contributing element
- Factor 3: Environmental or contextual factors

## Solution Approach

Describe the strategy used to fix the problem.

### The Fix

Show the corrected code with explanations:

```typescript
// ✅ Solution: Use useCallback with proper dependencies
import { useState, useCallback } from 'react';

function Component() {
  const [count, setCount] = useState(0);

  // Fix: useCallback recreates the function when count changes
  const handleClick = useCallback(() => {
    setTimeout(() => {
      console.log(count); // Now logs the current value
    }, 1000);
  }, [count]); // count is in dependency array

  return <button onClick={handleClick}>Click me</button>;
}
```

**OR, alternative solution**:

```typescript
// ✅ Alternative: Use functional state updates
function Component() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setTimeout(() => {
      // Access latest state via functional update
      setCount(currentCount => {
        console.log(currentCount); // Always gets latest value
        return currentCount;
      });
    }, 1000);
  };

  return <button onClick={handleClick}>Click me</button>;
}
```

### Why This Works

Explain the reasoning behind the solution:

- How does this fix address the root cause?
- What makes this approach reliable?
- Are there trade-offs to consider?

## Alternative Solutions Considered

List other approaches that were considered and why they weren't chosen:

### Alternative 1: [Approach Name]

**Approach**: Brief description

**Pros**:
- Advantage 1
- Advantage 2

**Cons**:
- Disadvantage 1
- Disadvantage 2

**Why not chosen**: Explanation

### Alternative 2: [Approach Name]

**Approach**: Brief description

**Why not chosen**: Explanation

## Prevention Strategy

Provide actionable advice on how to prevent this problem in the future.

### Code Review Checklist

What should reviewers look for?

- [ ] Checklist item 1 - Specific thing to verify
- [ ] Checklist item 2 - Another check to perform
- [ ] Checklist item 3 - Additional verification step

### Linting/Tooling

Are there tools that can catch this?

- **ESLint rule**: Name of rule and configuration
- **TypeScript check**: Type-level protection if applicable
- **Testing strategy**: How to test for this scenario

Example:
```json
{
  "rules": {
    "react-hooks/exhaustive-deps": "error"
  }
}
```

### Best Practices

General guidelines to avoid similar issues:

1. **Practice 1**: Specific actionable advice
2. **Practice 2**: Another preventive measure
3. **Practice 3**: Additional guidance

## How to Spot This Issue

Provide practical tips for identifying this problem:

**Code smells**:
- Pattern or code structure that suggests this issue
- Warning signs in the codebase
- Common contexts where this appears

**Questions to ask**:
- "Does this function reference props or state?"
- "Could this value be stale when the function executes?"
- "Are dependencies properly declared?"

**Testing for this**:
- How to write tests that would catch this
- User actions that would expose the bug

```typescript
// Example test that catches the issue
test('should use current count value in event handler', async () => {
  const { getByText } = render(<Component />);
  const button = getByText('Click me');
  
  // Change state
  fireEvent.click(button);
  
  // Wait for async operation
  await waitFor(() => {
    // Verify correct value was used
    expect(consoleLogSpy).toHaveBeenCalledWith(1);
  });
});
```

## Lessons Learned

Summarize the key takeaways:

1. **Lesson 1**: Main insight from this experience
2. **Lesson 2**: Another important learning
3. **Lesson 3**: Broader principle or pattern

## Related Issues

Link to similar problems or related patterns:

- [Related Problem 1](../path/to/problem.md) - Similar issue in different context
- [Related Pattern](../path/to/pattern.md) - Pattern that helps avoid this
- [External resource](https://example.com) - Relevant documentation or article

## References

- [React documentation on closures](https://react.dev)
- [Relevant article or discussion](https://example.com)

---

**Tags**: `#bug` `#react` `#closures` `#hooks` (for your own reference)

**Date Added**: YYYY-MM-DD

**Severity**: Low | Medium | High | Critical

**Source**: PR Review | Production Bug | Code Review | Testing
