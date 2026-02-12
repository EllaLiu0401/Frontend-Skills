# Best Practices & PR Learnings

Key learnings and best practices extracted from code reviews, refactoring experiences, and production issues.

## Overview

This section captures actionable insights from real-world development, organized as quick reference guides for common pitfalls and best practices.

## Architecture & Organization

- **[Frontend Project Structure](frontend-project-structure.md)** - Best practices for organizing utilities, services, and integrations. Learn when to use `utils/` vs `services/` vs `integrations/` and how to enforce conventions with ESLint.

## Data Handling

- **[Data Filtering and Transformation](data-filtering-and-transformation.md)** - Critical patterns for data transformation pipelines. Learn why premature filtering causes data loss, how to handle null values correctly, and when to filter vs format data.

## API & Backend Patterns

- **[API Contract Correctness](api-contract-correctness.md)** - Ensuring operations return accurate results. Learn why silent failures are dangerous, how to verify database operations, when to use `.executeTakeFirstOrThrow()`, and best practices for error handling in data layers.
- **[Database Repository Patterns](database-repository-patterns.md)** - Essential patterns for database repositories including soft deletes, timestamp handling, and defense-in-depth security. Learn why every query must filter soft-deleted records and when to use database-side vs application-side timestamps.
- **[Audit Logging Patterns](audit-logging-patterns.md)** - Complete guide to implementing audit trails using database triggers and session context. Learn why transactions and audit context are critical, how to avoid NULL audit logs, and patterns for compliance-ready logging.

## User Experience (UX)

- **[Handling Incomplete UI Features](handling-incomplete-ui-features.md)** - Best practices for managing features that aren't ready yet. Learn why empty onClick handlers are UX bugs, when to remove vs disable UI elements, and how to avoid confusing users with non-functional interfaces.

## Internationalization (i18n)

- **[Comprehensive i18n Audit](i18n-comprehensive-audit.md)** - Systematic approach to ensuring complete internationalization coverage. Learn how to identify hardcoded strings, implement proper i18n patterns, and maintain translations across multiple locales.

## PR Notes

Condensed lessons from code reviews:

- [PR: UX & Accessibility Fixes](pr-ux-and-accessibility-fixes.md) - Empty onClick handlers and semantic HTML heading hierarchy
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

### Data Handling

- Data transformation pipelines
- Null value handling
- Filter placement in transformation chains
- Edge case testing

### API & Backend

- API contract correctness
- Database operation verification
- Error handling patterns
- Repository layer best practices
- Silent failure prevention
- Audit logging and compliance
- Transaction patterns and isolation
- Database context management

### UI/UX Patterns

- Trigger-behavior synchronization
- Progressive enhancement
- User feedback patterns
- Handling incomplete features
- Empty onClick handlers and user confusion

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
5. **API Contract Accuracy**: Operations must return results that reflect what actually happened; use `.executeTakeFirstOrThrow()` for updates/deletes that require a match
6. **Consistent Error Handling**: If reads throw on missing records, writes should too; maintain consistent patterns across the codebase
7. **Audit Context**: Wrap data modifications in transactions and set audit context before operations; without it, audit logs capture NULL for user_id, request_id, and trace_id

---

_As you conduct PR reviews, extract and document key learnings here._
