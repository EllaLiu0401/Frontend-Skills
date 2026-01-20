# Contributing to Frontend Skills

This guide explains how to add new learnings to this repository while maintaining consistency and quality.

## Adding a New Entry

### Step 1: Choose the Right Topic Category

Select the most specific category that fits your learning:

| Category | Use When |
|----------|----------|
| `javascript/` | Core JavaScript language features, ES6+, async patterns |
| `typescript/` | Type system, generics, type guards, utility types |
| `react/` | Components, hooks, lifecycle, context, React-specific patterns |
| `state-management/` | Redux, Zustand, MobX, state patterns (not React Context) |
| `api-integration/` | REST, GraphQL, data fetching, caching strategies |
| `error-handling/` | Error boundaries, try-catch, error recovery, logging |
| `performance/` | Rendering optimization, bundle size, lazy loading, memoization |
| `testing/` | Unit tests, integration tests, E2E, testing patterns |
| `accessibility/` | ARIA, keyboard navigation, screen readers, WCAG compliance |
| `styling/` | CSS, Sass, styled-components, Tailwind, design systems |
| `build-tools/` | Webpack, Vite, Rollup, build optimization |
| `best-practices/` | PR learnings, code quality rules, cross-cutting concerns |

**Still unsure?** Choose the category where developers would most likely look for this information.

### Step 2: Select the Appropriate Template

Choose one of four formats based on what you're documenting:

#### PR Notes Template (`templates/pr-notes.md`)
**Use when**: You want to document learnings from a code review or PR.

**Perfect for**:
- PR review insights across multiple issues
- Comprehensive code review findings
- Cross-cutting concerns in a single PR
- Reusable rules extracted from reviews

**Example scenarios**:
- "PR #123 - Dashboard refactoring learnings"
- "PR #456 - API integration review findings"
- "PR #789 - Performance optimization insights"

#### Before/After Template (`templates/before-after.md`)
**Use when**: You have a code improvement, refactoring, or bug fix to document.

**Perfect for**:
- Performance optimizations
- Code quality improvements
- Refactoring examples
- Bug fixes with clear before/after states

**Example scenarios**:
- "Replaced nested ternaries with early returns"
- "Optimized re-renders using React.memo"
- "Fixed memory leak in useEffect"

#### Pattern Guide Template (`templates/pattern-guide.md`)
**Use when**: You want to document a general pattern or best practice.

**Perfect for**:
- Architectural patterns
- Design principles
- Reusable code patterns
- Best practices and conventions

**Example scenarios**:
- "Custom hooks for data fetching"
- "Compound component pattern"
- "Error boundary implementation pattern"

#### Problem/Solution Template (`templates/problem-solution.md`)
**Use when**: You encountered a specific problem during a PR review or debugging session.

**Perfect for**:
- PR review insights
- Production issues and their fixes
- Edge cases and gotchas
- Debugging lessons

**Example scenarios**:
- "Why array keys should be stable"
- "Stale closure in event handler"
- "Race condition in concurrent requests"

### Step 3: Create Your Documentation File

#### Naming Conventions

Use kebab-case with descriptive names:

✅ **Good**:
- `avoid-prop-drilling.md`
- `custom-hooks-best-practices.md`
- `fixing-memory-leaks-in-effects.md`
- `api-error-handling-patterns.md`

❌ **Bad**:
- `hooks.md` (too generic)
- `MyComponent.md` (not descriptive of the learning)
- `pr_review_notes.md` (underscores, not descriptive)
- `thing-i-learned-today.md` (not searchable)

#### File Location

Place your file in the appropriate topic folder:
```
docs/{topic-category}/{your-descriptive-filename}.md
```

Example:
```
docs/react/avoid-prop-drilling.md
```

### Step 4: Write Your Content

#### Code Snippet Standards

**Use syntax highlighting**:
```typescript
// ✅ Do this
function example() {
  return true;
}
```

**Keep lines under 80 characters** when possible for readability.

**Annotate important parts** with comments:
```typescript
// ❌ Problematic: Missing dependency
useEffect(() => {
  fetchData(userId);
}, []); // userId should be in dependency array
```

**Show imports** when they add clarity:
```typescript
import { useState, useEffect } from 'react';
import { debounce } from 'lodash-es';
```

#### Keeping Content General

Since this repository contains learnings from various projects including company work, follow these guidelines:

**✅ DO**:
- Abstract the concept to its essence
- Use generic, descriptive variable names
- Focus on the pattern, not the specific implementation
- Replace domain-specific logic with comments like `// business logic here`
- Use common examples (user management, shopping cart, todo lists)

**❌ DON'T**:
- Include company-specific code or logic
- Reference internal tools, APIs, or services by name
- Include sensitive information or proprietary algorithms
- Use domain-specific terminology without explanation

**Example of Abstraction**:

❌ **Too specific**:
```typescript
// Bad: Company-specific
function validateInternalUserPermissions(user: InternalUser) {
  return user.hasAccess(INTERNAL_RESOURCE_ID) && 
         user.department === OUR_DEPT_CODE;
}
```

✅ **Properly generalized**:
```typescript
// Good: Generic pattern
function validateUserPermissions(user: User) {
  return user.hasAccess(resourceId) && 
         user.meetsRequirement(condition);
}
```

#### Structure Your Content

Follow the template structure consistently:

1. **Start with context**: What problem or situation are you addressing?
2. **Show the code**: Use clear, formatted examples
3. **Explain the "why"**: Why is this important? What breaks if ignored?
4. **Provide takeaways**: What should readers remember?
5. **Link related content**: Reference similar patterns or learnings

### Step 5: Update the Topic Index

After creating your entry, update the topic's `README.md` to include a link:

```markdown
## Contents

- [Avoid Prop Drilling](avoid-prop-drilling.md) - Patterns to prevent excessive prop passing
- [Custom Hooks Best Practices](custom-hooks-best-practices.md) - Guidelines for creating reusable hooks
- [Your New Entry](your-new-entry.md) - Brief one-line description
```

Keep the list alphabetically sorted or logically grouped (basic → advanced).

## Quality Standards

### Every Entry Should Have

- **Clear title**: Immediately conveys what the entry is about
- **Context**: Why this matters and when it applies
- **Code examples**: Concrete illustrations of the concept
- **Explanation**: Not just what, but why
- **Takeaways**: Actionable insights to remember
- **Self-contained**: Understandable without external context

### Avoid

- Overly long entries (split into multiple if needed)
- Unexplained jargon or acronyms
- Code examples without explanation
- Generic advice without concrete examples
- Duplicate content (reference existing entries instead)

## Writing Style Guide

**Tone**: Professional but conversational, like explaining to a colleague

**Perspective**: Use "we" for collective insights, "you" when guiding the reader

**Code comments**: Use ✅ and ❌ to mark good and bad examples

**Formatting**:
- Use `code` formatting for inline code references
- Use **bold** for emphasis on key concepts
- Use lists for multiple related points
- Use tables for comparisons

## Review Checklist

Before considering your entry complete:

- [ ] File is in the correct topic folder
- [ ] Filename is descriptive and follows kebab-case
- [ ] Content follows the chosen template structure
- [ ] Code examples are properly formatted with syntax highlighting
- [ ] All code is generalized and contains no sensitive information
- [ ] Explanations are clear and provide the "why"
- [ ] Key takeaways are explicitly stated
- [ ] Topic's README.md is updated with a link to the new entry
- [ ] Related patterns are cross-referenced where applicable
- [ ] Spelling and grammar are correct

## Examples of Good Entries

### Good Before/After Entry
```markdown
# Optimizing Re-renders with React.memo

## Context
Component was re-rendering unnecessarily when parent state changed,
even though its props hadn't changed.

## Before
[code showing component without optimization]

## After
[code showing React.memo usage]

## Key Takeaways
- Use React.memo for expensive pure components
- Always profile before optimizing
...
```

### Good Pattern Guide Entry
```markdown
# Custom Hooks for Data Fetching

## Pattern Name
Extracting data fetching logic into reusable custom hooks.

## When to Use
- When multiple components fetch similar data
- When you need consistent loading/error states
...
```

### Good Problem/Solution Entry
```markdown
# Stale Closure in Event Handler

## Problem Description
Event handler was using an old value of state because
it closed over the initial render's value.

## Root Cause Analysis
[Explanation of closure capture]
...
```

## Adding Prompt Templates

In addition to documenting learnings, you can also save effective AI prompts in the `prompts/` directory.

### When to Save a Prompt

Save a prompt when:
- You've refined it through multiple iterations and it works well
- It's reusable across different scenarios
- It consistently produces high-quality results
- It could save time for future tasks

### Prompt Template Structure

Each prompt template should include:

```markdown
# [Prompt Name]

## Purpose
What the prompt achieves

## Template
```
The actual prompt with {VARIABLES}
```

## Variables
- {VAR}: What to fill in

## Example
Concrete example with variables filled in

## Expected Output
What kind of response to expect

## Tips
- Best practices for using this prompt
```

### Categories

Place prompts in the appropriate category:
- `code-review/` - For reviewing code
- `debugging/` - For diagnosing issues
- `code-generation/` - For generating code
- `refactoring/` - For improving code
- `learning/` - For understanding concepts
- `documentation/` - For generating docs

### Security Note

Never include in prompts:
- API keys or credentials
- Proprietary code or algorithms
- Sensitive business logic
- Internal system details

## Getting Help

If you're unsure about:
- **Which category to use**: Choose the most specific one, or ask
- **Which template fits best**: Default to Before/After for code improvements
- **How to generalize**: Focus on the pattern, not the specifics
- **Whether content is too company-specific**: If in doubt, generalize more

## Maintenance

Periodically review entries for:
- Outdated patterns (update or mark as deprecated)
- Broken internal links
- Opportunities to merge similar entries
- Missing cross-references between related topics

---

**Remember**: The goal is to create a valuable knowledge base. Quality over quantity. Take the time to make each entry clear, useful, and well-organized.
