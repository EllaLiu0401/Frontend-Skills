# [Descriptive Title of the Learning]

## Context

Provide a brief explanation of the situation that led to this learning. What was the problem or challenge? What prompted the refactoring or improvement?

Example:
- "Component was re-rendering unnecessarily causing performance issues"
- "Code was difficult to read and maintain due to nested conditionals"
- "Memory leak occurred because cleanup wasn't properly implemented"

## Before

Show the original code with issues. Use comments to annotate problematic areas.

```typescript
// ❌ Problematic: [Brief description of what's wrong]
function ExampleComponent({ userId, data }) {
  // Issue 1: Missing dependency in useEffect
  useEffect(() => {
    fetchUserData(userId);
  }, []); // userId should be in dependency array

  // Issue 2: Not memoized, causes unnecessary re-renders
  const processedData = data.map(item => expensiveOperation(item));

  return (
    <div>
      {processedData.map(item => (
        <Item key={item.id} data={item} />
      ))}
    </div>
  );
}
```

**Issues identified**:
- List the specific problems with bullet points
- Explain why each issue matters
- Describe the impact (performance, bugs, maintainability, etc.)

## After

Show the improved code with annotations explaining the improvements.

```typescript
// ✅ Improved: [Brief description of improvements]
import { useEffect, useMemo } from 'react';

function ExampleComponent({ userId, data }) {
  // Fix 1: Added userId to dependency array
  useEffect(() => {
    fetchUserData(userId);
  }, [userId]); // Now properly tracks userId changes

  // Fix 2: Memoized expensive calculation
  const processedData = useMemo(
    () => data.map(item => expensiveOperation(item)),
    [data]
  ); // Only recalculates when data changes

  return (
    <div>
      {processedData.map(item => (
        <Item key={item.id} data={item} />
      ))}
    </div>
  );
}
```

**Improvements made**:
- List the specific changes with bullet points
- Explain the reasoning behind each change
- Describe the benefits (better performance, correctness, readability)

## Why This Matters

Explain the broader implications of this learning:

- **Performance**: How does this affect application performance?
- **Correctness**: Does this prevent bugs or unexpected behavior?
- **Maintainability**: How does this make the code easier to work with?
- **User Experience**: What impact does this have on the end user?

## How to Spot This in Code Reviews

Provide practical guidance for identifying similar issues:

1. **Look for**: Specific patterns or code smells to watch out for
2. **Ask yourself**: Questions to ask when reviewing similar code
3. **Red flags**: Warning signs that indicate this problem might exist

Example:
- "Look for useEffect hooks with empty dependency arrays"
- "Ask yourself: Does this hook reference any props or state?"
- "Red flag: ESLint warning about missing dependencies"

## Key Takeaways

Summarize the main lessons learned in bullet points. These should be actionable insights:

- Keep these concise and memorable
- Focus on the "rules" or "principles" to remember
- Include "do this" and "avoid that" guidance

Example:
- Always include all dependencies in useEffect dependency arrays
- Use useMemo for expensive calculations that don't need to run every render
- Profile before optimizing - don't guess at performance issues

## Related Patterns

Link to other relevant learnings or patterns in this repository:

- [Link to related pattern 1](../path/to/pattern.md) - Brief description
- [Link to related pattern 2](../path/to/pattern.md) - Brief description
- [External resource](https://example.com) - If applicable

---

**Tags**: `#performance` `#react` `#hooks` (for your own reference)

**Date Added**: YYYY-MM-DD

**Difficulty**: Beginner | Intermediate | Advanced
