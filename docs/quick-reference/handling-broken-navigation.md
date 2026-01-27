# Quick Reference: Handling Broken Navigation

## The Problem

You have a button/link that navigates to a route that doesn't exist yet.

```tsx
// ⚠️ PROBLEM: /analytics doesn't exist
<button onClick={() => router.push('/analytics')}>
  View Reports
</button>
```

**Result**: Users get 404 errors when they click.

---

## The Solution

**Disable the trigger, don't remove it.**

```tsx
// ✅ SOLUTION: Button visible but disabled
{/* Disabled until /analytics route is implemented */}
<button disabled>
  View Reports
</button>
```

**Why?**
- Maintains visual consistency
- Shows users what's coming
- Easy to enable later
- No 404 errors

---

## Component Cleanup

### Remove Unnecessary Code

```tsx
// ❌ DELETE THESE
import { useRouter } from 'next/navigation'; // Not needed anymore
const router = useRouter();                  // Not needed anymore
const handleClick = () => { ... };          // Not needed anymore
onClick={handleClick}                        // Not needed anymore

// ✅ KEEP THIS
disabled                                     // New attribute
```

---

## Test Cleanup

### Remove Obsolete Mocks

```typescript
// ❌ DELETE THESE
import userEvent from '@testing-library/user-event';
const mockPush = vi.fn();
vi.mock('next/navigation', () => ({ ... }));

// ✅ KEEP THESE
import { render, screen } from '@testing-library/react';
import { describe, it, expect } from 'vitest';
```

### Update Test Cases

```typescript
// ❌ DELETE THIS TEST
it('navigates to /analytics when clicked', async () => {
  const user = userEvent.setup();
  await user.click(button);
  expect(mockPush).toHaveBeenCalled();
});

// ✅ ADD THIS TEST
it('renders button in disabled state', () => {
  const button = screen.getByText('View Reports');
  expect(button).toBeDisabled();
});
```

### Simplify Component Mocks

```typescript
// ❌ BEFORE: Unused props
Button: ({ children, onClick, disabled }) => (
  <button onClick={onClick} disabled={disabled}>
    {children}
  </button>
)

// ✅ AFTER: Only what's tested
Button: ({ children, disabled }) => (
  <button disabled={disabled}>
    {children}
  </button>
)
```

---

## Quick Checklist

When fixing broken navigation:

- [ ] Disable the trigger (don't delete it)
- [ ] Remove router imports and hooks
- [ ] Remove click handler functions
- [ ] Add comment explaining why it's disabled
- [ ] Remove router mocks from tests
- [ ] Remove userEvent import if unused
- [ ] Delete navigation test cases
- [ ] Add disabled state test
- [ ] Simplify component mocks
- [ ] Run tests to verify
- [ ] Check linter for unused imports

---

## When to Remove vs Disable

| Scenario | Action |
|----------|--------|
| Route coming soon | **Disable** - keeps UI consistent |
| Feature cancelled | **Remove** - no point showing it |
| Temporary placeholder | **Disable** - enable when ready |
| Broken implementation | **Fix** - don't hide problems |
| Experimental feature | **Hide** - use feature flag |

---

## Progressive Implementation Pattern

```tsx
// Stage 1: Disabled UI
<button disabled>Launch</button>

// Stage 2: Basic functionality
<button onClick={() => router.push('/route')}>
  Launch
</button>

// Stage 3: Enhanced UX
<button 
  onClick={handleClick}
  disabled={isLoading}
  aria-busy={isLoading}
>
  {isLoading ? 'Loading...' : 'Launch'}
</button>
```

---

## Common Mistakes

### ❌ Leaving Broken Navigation

```tsx
// Users will get 404
<button onClick={() => router.push('/nonexistent')}>
  Click Me
</button>
```

### ❌ Removing UI Entirely

```tsx
// Inconsistent UI, users confused
{routeExists && <button>Click Me</button>}
```

### ❌ Silent Failure

```tsx
// Button does nothing, no feedback
<button onClick={() => {}}>
  Click Me
</button>
```

### ✅ Correct Approach

```tsx
// Clear state, prevents errors
<button disabled title="Coming soon">
  Click Me
</button>
```

---

## Red Flags in Code Review

🚩 Navigation to hardcoded paths without route verification  
🚩 Click handlers with empty functions `onClick={() => {}}`  
🚩 TODO comments on user-facing functionality  
🚩 Console errors about routes not found  
🚩 Test mocks for functionality not in component  

---

## Related Rules

- **No Broken Links in Production**: All navigation must work or be disabled
- **Disable, Don't Delete**: Keep UI consistent during development
- **Test What Exists**: Don't test removed functionality
- **KISS in Tests**: Remove unused mocks immediately
- **Comments Explain Intent**: Why is this disabled? What's needed to enable?

---

## See Also

- [Handling Incomplete Routes (Full Guide)](../best-practices/pr-handling-incomplete-routes-and-navigation.md)
- [Test Cleanup After Refactoring](../testing/test-cleanup-after-refactoring.md)
- [PR Review Checklist](pr-review-checklist.md)

---

**Last Updated**: January 2026
