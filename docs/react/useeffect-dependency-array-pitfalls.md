# useEffect Dependency Array Pitfalls

## Context

During code review, discovered that a component wasn't properly updating when props changed. The issue was traced to an incomplete dependency array in a useEffect hook, causing the component to use stale values and not react to prop changes as expected.

## Before

The original implementation had missing dependencies, leading to stale closures and unexpected behavior.

```typescript
import { useEffect, useState } from 'react';

// ❌ Problematic: Missing dependencies in useEffect
function UserDashboard({ userId, filters }) {
  const [userData, setUserData] = useState(null);
  const [loading, setLoading] = useState(true);

  // Issue 1: userId is referenced but not in dependency array
  useEffect(() => {
    async function fetchUserData() {
      setLoading(true);
      const response = await fetch(`/api/users/${userId}`);
      const data = await response.json();
      setUserData(data);
      setLoading(false);
    }

    fetchUserData();
  }, []); // Empty array means this only runs on mount

  // Issue 2: filters is used but not declared as dependency
  useEffect(() => {
    if (userData) {
      const filtered = applyFilters(userData, filters);
      displayResults(filtered);
    }
  }, [userData]); // Missing 'filters' in dependency array

  if (loading) return <div>Loading...</div>;
  
  return <div>{/* render dashboard */}</div>;
}
```

**Issues identified**:
- **Stale userId**: When `userId` prop changes, the useEffect doesn't re-run, so old user data remains displayed
- **Stale filters**: When `filters` change, the results aren't updated because the effect doesn't track `filters`
- **User impact**: Users see incorrect data when navigating between different users or changing filter settings
- **ESLint warning ignored**: The `react-hooks/exhaustive-deps` rule would flag this, but was likely disabled or ignored

## After

The improved version includes all necessary dependencies, ensuring the component responds correctly to prop changes.

```typescript
import { useEffect, useState, useCallback } from 'react';

// ✅ Improved: All dependencies properly declared
function UserDashboard({ userId, filters }) {
  const [userData, setUserData] = useState(null);
  const [loading, setLoading] = useState(true);

  // Fix 1: Added userId to dependency array
  useEffect(() => {
    async function fetchUserData() {
      setLoading(true);
      try {
        const response = await fetch(`/api/users/${userId}`);
        const data = await response.json();
        setUserData(data);
      } catch (error) {
        console.error('Failed to fetch user data:', error);
      } finally {
        setLoading(false);
      }
    }

    fetchUserData();
  }, [userId]); // Now re-runs when userId changes

  // Fix 2: Added filters to dependency array
  useEffect(() => {
    if (userData) {
      const filtered = applyFilters(userData, filters);
      displayResults(filtered);
    }
  }, [userData, filters]); // Now re-runs when either userData or filters change

  if (loading) return <div>Loading...</div>;
  
  return <div>{/* render dashboard */}</div>;
}
```

**Improvements made**:
- **Correct dependencies**: All referenced props and state are included in dependency arrays
- **Proper reactivity**: Component now updates when userId or filters change
- **Better error handling**: Added try-catch for robustness
- **ESLint compliant**: No more warnings from exhaustive-deps rule

## Why This Matters

This seemingly small issue has significant implications:

- **Correctness**: Missing dependencies lead to bugs where components display stale data, potentially showing wrong information to users
- **User Experience**: Users expect the UI to update when they change inputs or navigate. Stale data creates confusion and erodes trust
- **Debugging difficulty**: These bugs are often hard to reproduce and debug because the component appears to work initially but fails in specific interaction patterns
- **Maintenance**: Future developers might add more dependencies and perpetuate the pattern if it's not corrected early

## How to Spot This in Code Reviews

When reviewing useEffect usage, systematically check for this issue:

1. **Look for**: useEffect hooks with non-empty bodies but empty dependency arrays
2. **Scan the effect body**: Identify all props, state, and functions referenced inside
3. **Compare**: Check if all referenced values are in the dependency array
4. **Check ESLint warnings**: Don't ignore `react-hooks/exhaustive-deps` warnings
5. **Ask yourself**: "If this prop/state changes, should the effect re-run?"

**Red flags**:
- Comments like `// eslint-disable-next-line react-hooks/exhaustive-deps`
- Empty dependency arrays `[]` with non-trivial effect bodies
- Effects that reference props or state from outer scope

## Key Takeaways

Follow these rules to avoid dependency array issues:

- **Include all dependencies**: Every value referenced from component scope must be in the dependency array
- **Don't ignore ESLint**: The `react-hooks/exhaustive-deps` rule exists for a reason - listen to it
- **Use functional updates**: For state updates, use `setState(prev => ...)` to avoid needing state in dependencies
- **Extract static values**: Move constants and helper functions outside the component if they don't need component scope
- **Be explicit**: An empty array `[]` means "run once on mount" - make sure that's actually what you want

**Exception**: Only use empty arrays when the effect truly should run once (like initializing a third-party library that doesn't depend on component state).

## Related Patterns

- [Custom Hooks for Data Fetching](custom-hooks-data-fetching.md) - Pattern that encapsulates data fetching with proper dependency management
- [Memoization with useMemo and useCallback](memoization-patterns.md) - How to prevent unnecessary re-runs while maintaining correct dependencies
- [Stale Closures in Event Handlers](stale-closures-event-handlers.md) - Related issue with closures capturing old values

---

**Tags**: `#react` `#hooks` `#useEffect` `#bugs` `#dependencies`

**Date Added**: 2026-01-20

**Difficulty**: Intermediate
