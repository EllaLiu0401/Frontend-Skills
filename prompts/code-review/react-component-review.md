# React Component Review

## Purpose

Conduct a comprehensive review of a React component, identifying issues with hooks, rendering, performance, accessibility, and best practices.

## Template

```
Review this React component and provide detailed feedback on:

1. **Hooks Usage**
   - Are dependencies correctly specified?
   - Are hooks used in the right order and conditions?
   - Any unnecessary re-renders due to hook misuse?

2. **Performance**
   - Unnecessary re-renders
   - Missing memoization opportunities
   - Expensive operations that could be optimized

3. **Code Quality**
   - Readability and maintainability
   - Component structure and organization
   - Prop types and TypeScript usage

4. **Accessibility**
   - Semantic HTML
   - Keyboard navigation
   - ARIA attributes

5. **Best Practices**
   - React patterns and anti-patterns
   - Error handling
   - Edge cases

Please provide specific suggestions with code examples where applicable.

Component code:
```{LANGUAGE}
{COMPONENT_CODE}
```
```

## Variables

- `{LANGUAGE}`: Programming language (typescript, javascript, tsx, jsx)
- `{COMPONENT_CODE}`: The React component code to review

## Example

```
Review this React component and provide detailed feedback on:

1. **Hooks Usage**
   - Are dependencies correctly specified?
   - Are hooks used in the right order and conditions?
   - Any unnecessary re-renders due to hook misuse?

2. **Performance**
   - Unnecessary re-renders
   - Missing memoization opportunities
   - Expensive operations that could be optimized

3. **Code Quality**
   - Readability and maintainability
   - Component structure and organization
   - Prop types and TypeScript usage

4. **Accessibility**
   - Semantic HTML
   - Keyboard navigation
   - ARIA attributes

5. **Best Practices**
   - React patterns and anti-patterns
   - Error handling
   - Edge cases

Please provide specific suggestions with code examples where applicable.

Component code:
```typescript
import { useState, useEffect } from 'react';

function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => {
        setUser(data);
        setLoading(false);
      });
  }, []);

  if (loading) return <div>Loading...</div>;

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}
```
```

## Expected Output

The AI will provide:
- Detailed analysis of each focus area
- Specific issues found with explanations
- Code examples showing improvements
- Prioritization of issues (critical, important, nice-to-have)
- Alternative approaches when applicable

Example response structure:
```
🔴 Critical Issues:
1. Missing userId dependency in useEffect...

🟡 Important Improvements:
1. Add error handling for fetch...

🟢 Suggestions:
1. Consider extracting data fetching to custom hook...

Improved version:
[refactored code]
```

## Tips

### For Better Results

- **Include TypeScript definitions** if using TypeScript - helps identify type-related issues
- **Mention project constraints** (e.g., "We use React 18", "Must support IE11")
- **Specify priority** if you want focus on specific aspects
- **Provide context** about where this component is used

### Common Follow-ups

After the initial review, you might ask:
- "Show me how to implement the custom hook you suggested"
- "Are there any security concerns with this approach?"
- "How would this perform with 1000+ items?"
- "What tests should I write for this component?"

### Variations

**Quick Performance Review**:
```
Focus only on performance issues in this React component:
[component code]
```

**Accessibility-Only Review**:
```
Review this component specifically for accessibility issues:
[component code]
```

**Hooks-Specific Review**:
```
Check this component's hooks usage for any issues:
[component code]
```

## Related Prompts

- [Performance Audit](performance-audit.md) - Deep dive into performance
- [Accessibility Check](../learning/accessibility-check.md) - Detailed a11y review
- [Refactor Component](../refactoring/component-refactor.md) - Restructuring suggestions

---

**Tags**: `#react` `#code-review` `#components` `#hooks`

**Last Updated**: 2026-01-20
