# Best Practices & PR Learnings

Key learnings and best practices extracted from code reviews, refactoring experiences, and production issues.

## Overview

This section captures actionable insights from real-world development, organized as quick reference guides for common pitfalls and best practices.

## Architecture & Organization

- **[Frontend Project Structure](frontend-project-structure.md)** - Best practices for organizing utilities, services, and integrations. Learn when to use `utils/` vs `services/` vs `integrations/` and how to enforce conventions with ESLint.

## Internationalization (i18n)

- **[Comprehensive i18n Audit](i18n-comprehensive-audit.md)** - Systematic approach to ensuring complete internationalization coverage. Learn how to identify hardcoded strings, implement proper i18n patterns, and maintain translations across multiple locales.

## PR Notes

Condensed lessons from code reviews:

- [PR-0161: Dashboard Foundation](pr-0161-dashboard-foundation.md) - Type inference, dead code, and UI triggers
- [Handling Incomplete Routes and Navigation](pr-handling-incomplete-routes-and-navigation.md) - Disabling vs removing UI, navigation bug fixes, test cleanup
- [React Props: Readonly & Immutability](react-props-readonly-immutability.md) - Enforcing props immutability with TypeScript readonly

## Categories

### Code Quality

- Unused code detection and cleanup
- Type inference vs explicit annotations
- Code organization and structure
- Project structure conventions and enforcement
- Internationalization auditing and enforcement

### UI/UX Patterns

- Trigger-behavior synchronization
- Progressive enhancement
- User feedback patterns

### Type Safety

- When to use type inference
- Avoiding redundant generics
- Type system best practices

## PR Notes Template

When documenting new PR learnings, follow this structure:

```markdown
# PR #### — <short title>

## TL;DR

- <one-line takeaway>

## Issues & Fixes

### 1) <issue title>

- Symptom:
- Root cause:
- Fix:
- Prevention:
- Files:

## Reusable Rules

- <rule 1>
- <rule 2>
```

## Quick Rules

Common patterns from PR reviews:

1. **Type Inference**: Rely on TypeScript's inference for hooks; avoid redundant generics
2. **Dead Code**: Remove unused components/data or wire them up properly
3. **UI Triggers**: Every interactive element must have working behavior or stay hidden
4. **Progressive Disclosure**: Build features completely before exposing them to users

---

_As you conduct PR reviews, extract and document key learnings here._
