# PR 0161 — Dashboard Foundation

> **TL;DR**: Catch unused code, avoid premature type annotations, and don't ship UI triggers without working behavior.

## Context

This PR review highlighted common issues in early-stage feature development: redundant type annotations, unused components, and incomplete UI implementations.

## Issues & Fixes

### 1) Redundant Generic Type Parameters on Hooks

**Symptom**: Hooks explicitly typed with generics where TypeScript can infer the types automatically.

```typescript
// ❌ Before: Redundant generic
const [data, setData] = useState<DataType | null>(null);

// ✅ After: Let TypeScript infer
const [data, setData] = useState(null);
// TypeScript infers: useState<null | DataType> when setData(dataValue) is called
```

**Root Cause**: Over-eagerness to annotate types, possibly from misunderstanding TypeScript's inference capabilities.

**Fix**: Remove generic type parameters when TypeScript can infer them from initial values and usage.

**Prevention**: 
- Let TypeScript infer types by default
- Only add explicit types when inference fails or produces incorrect types
- Use type assertions sparingly

**Related Topics**: [TypeScript Type Inference](../typescript/)

---

### 2) Components and Data Added but Never Used

**Symptom**: Components defined but not imported/rendered anywhere; data structures created but never consumed.

```typescript
// ❌ Component defined but never used
export const UnusedDashboardWidget = () => {
  return <div>Widget</div>;
};

// ❌ Data fetched but never displayed
const unusedMetrics = await fetchMetrics();
```

**Root Cause**: Feature development in progress but code committed before integration complete.

**Fix**: 
- Remove unused code entirely, or
- Wire up components and data to working features
- Add TODO comments for work-in-progress

**Prevention**:
- Run linter checks for unused exports (`eslint-plugin-unused-imports`)
- Review git diff before committing to spot unused additions
- Complete integration before committing feature code

**Related Topics**: [Code Quality](../javascript/), [Error Handling](../error-handling/)

---

### 3) UI Triggers Shipped Before Behavior Exists

**Symptom**: Buttons, links, or interactive elements visible to users but without working functionality.

```tsx
// ❌ Button visible but does nothing
<button onClick={() => {}}>Export Data</button>

// ❌ Link visible but navigates nowhere
<Link to="/dashboard/settings">Settings</Link>
// But /dashboard/settings route doesn't exist yet
```

**Root Cause**: UI built ahead of behavior implementation; incomplete feature shipped to production.

**Fix**:
- Hide UI elements until functionality is ready: `{featureEnabled && <Button />}`
- Disable interactive elements with clear messaging: `<Button disabled>Coming Soon</Button>`
- Implement at least basic behavior before showing triggers

**Prevention**:
- Use feature flags for incomplete features
- Build behavior before UI, or in parallel
- Add integration tests that verify click handlers work
- Never ship interactive elements without working actions

**Related Topics**: [React Best Practices](../react/), [Testing](../testing/)

---

## Reusable Rules

Apply these rules in all code reviews and feature development:

1. **Type Inference First**: Rely on TypeScript's type inference for hooks and variables; avoid redundant generic type parameters
2. **No Dead Code**: Unused components, imports, or data should be removed or properly wired into the application
3. **Complete Interactions**: UI triggers (buttons, links) must have working behavior or remain hidden until implementation is complete
4. **Feature Flags Over Half-Features**: Use feature flags to hide incomplete functionality rather than shipping non-working UI
5. **Linting is Your Friend**: Set up linters to catch unused exports, variables, and imports automatically

## Checklist for Feature PRs

Before submitting a PR for review:

- [ ] All new components are actually used somewhere
- [ ] All interactive elements have working click handlers/navigation
- [ ] Types are inferred where possible (not over-annotated)
- [ ] No TODO comments for critical functionality
- [ ] Feature flags in place for incomplete features
- [ ] Linter passes with no unused code warnings

## Related Documentation

- [TypeScript Best Practices](../typescript/)
- [React Component Patterns](../react/)
- [Code Review Checklist](../../prompts/code-review/)

---

**Source**: PR Review #0161  
**Date**: January 2026  
**Topics**: TypeScript, React, Code Quality, UI/UX
