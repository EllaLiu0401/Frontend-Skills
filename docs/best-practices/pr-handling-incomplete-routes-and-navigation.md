# PR — Handling Incomplete Routes and Navigation

> **TL;DR**: Disable UI triggers for unimplemented routes instead of removing them. Keep visual consistency while preventing 404 errors.

## Context

During a code review, a button was found navigating to a route (`/analytics`) that didn't exist yet, causing 404 errors when users clicked it. This raised the question: should we remove the button entirely or handle it gracefully?

## Issues & Fixes

### 1) Navigation Button Pointing to Non-Existent Route

**Symptom**: A "View Reports" button was triggering navigation to `/analytics`, but no such route existed in the application.

```tsx
// ❌ Before: Button causes 404 when clicked
import { useRouter } from 'next/navigation';

export function ReportsCard() {
  const router = useRouter();

  const handleViewReports = () => {
    router.push('/analytics'); // ⚠️ Route doesn't exist!
  };

  return (
    <div>
      <h2>Top Reports</h2>
      <button onClick={handleViewReports}>
        View Reports
      </button>
    </div>
  );
}

// ✅ After: Button is visible but disabled until route exists
export function ReportsCard() {
  return (
    <div>
      <h2>Top Reports</h2>
      {/* Disabled until /analytics route is implemented */}
      <button disabled>
        View Reports
      </button>
    </div>
  );
}
```

**Root Cause**: UI was built ahead of route implementation. The button was added during initial design but the corresponding route wasn't created yet.

**Fix**: 
- Remove unused router imports and handler functions
- Add `disabled` attribute to the button
- Add clear comment explaining why it's disabled
- Keep visual consistency for users

**Prevention**: 
- Verify all routes exist before adding navigation triggers
- Use feature flags for incomplete features
- Add integration tests that verify routes exist
- Review navigation paths during code review

**Related Topics**: [React Best Practices](../react/), [Routing](../api-integration/)

---

### 2) Redundant Test Mocks After Functionality Removed

**Symptom**: After removing the router functionality, test mocks for `useRouter` and `onClick` handler remained in the test file.

```typescript
// ❌ Before: Unnecessary mocks for removed functionality
import userEvent from '@testing-library/user-event';

const mockPush = vi.fn();

vi.mock('next/navigation', () => ({
  useRouter: () => ({
    push: mockPush,
    replace: vi.fn(),
    // ... other router methods
  }),
}));

// Test that's no longer relevant
it('navigates to /analytics when clicked', async () => {
  const user = userEvent.setup();
  render(<ReportsCard />);
  
  const button = screen.getByText('View Reports');
  await user.click(button);
  
  expect(mockPush).toHaveBeenCalledWith('/analytics'); // ❌ No longer needed
});

// ✅ After: Clean test for disabled state
it('renders View Reports button in disabled state', () => {
  render(<ReportsCard />);
  
  const button = screen.getByText('View Reports');
  expect(button).toBeInTheDocument();
  expect(button).toBeDisabled(); // ✅ Test the actual current behavior
});
```

**Root Cause**: Tests weren't updated when component behavior changed. Old mocks and test cases remained after functionality was removed.

**Fix**: 
- Remove `useRouter` mock entirely
- Remove `userEvent` import (no longer testing clicks)
- Remove click navigation tests
- Add test for disabled state
- Simplify Button mock to only include needed props

**Prevention**:
- Review and update tests whenever component behavior changes
- Remove unused mocks immediately
- Keep test code as simple as possible (KISS principle)
- Run tests after refactoring to catch broken assertions

**Related Topics**: [Testing Best Practices](../testing/), [Test Refactoring](../testing/)

---

### 3) Redundant Props in Component Mocks

**Symptom**: Test mocks included `onClick` parameter even though no tests used it anymore.

```typescript
// ❌ Before: onClick included but never used
vi.mock('@ui/components', () => ({
  Button: ({
    children,
    onClick,    // ❌ Not used in any test
    disabled,
  }: {
    children: React.ReactNode;
    onClick?: () => void;  // ❌ Redundant type
    disabled?: boolean;
  }) => (
    <button onClick={onClick} disabled={disabled}>
      {children}
    </button>
  ),
}));

// ✅ After: Only include what's tested
vi.mock('@ui/components', () => ({
  Button: ({ children, disabled }: { 
    children: React.ReactNode; 
    disabled?: boolean 
  }) => (
    <button disabled={disabled}>
      {children}
    </button>
  ),
}));
```

**Root Cause**: Over-engineering test mocks. Including functionality "just in case" rather than keeping mocks minimal.

**Fix**: 
- Remove `onClick` prop and type
- Simplify mock to single-line function signature
- Only mock what's actually being tested

**Prevention**:
- Follow KISS (Keep It Simple, Stupid) principle in tests
- Only mock the props your tests actually verify
- Regularly audit test files for redundant code
- Use linter rules to detect unused parameters

**Related Topics**: [Testing](../testing/testing-semantic-behavior-not-implementation.md), [Code Quality](../quick-reference/code-quality-rules.md)

---

## Reusable Rules

Apply these rules in all code reviews and feature development:

1. **Disable, Don't Delete**: When a route doesn't exist yet, disable the navigation trigger rather than removing it. This maintains UI consistency and visual completeness.

2. **No Broken Links in Production**: Never ship navigation triggers (buttons, links) that point to non-existent routes. Either implement the route or disable the trigger.

3. **Test What Exists, Not What's Removed**: When functionality changes, immediately update tests. Remove mocks for deleted features and test the new behavior.

4. **Minimal Test Mocks**: Only mock the props and functions that your tests actually verify. Avoid "just in case" mocking.

5. **Comments Explain Intent**: When disabling features temporarily, add clear comments explaining why and what needs to happen to enable them.

6. **KISS in Tests**: Keep test code as simple as possible. If a mock has unused parameters, remove them immediately.

## Checklist for Navigation PRs

Before submitting a PR with navigation elements:

- [ ] All navigation targets (routes, pages) exist in the application
- [ ] If a route doesn't exist, the trigger is disabled with a comment
- [ ] Tests verify the actual current behavior (disabled state, not removed functionality)
- [ ] No unused mocks in test files (router, event handlers, etc.)
- [ ] Mock components only include props that tests actually verify
- [ ] Linter passes with no unused imports or variables
- [ ] Integration tests verify routes are accessible

## Checklist for Test Refactoring

When updating tests after component changes:

- [ ] Remove mocks for deleted functionality immediately
- [ ] Remove unused imports (`userEvent`, router mocks, etc.)
- [ ] Update test descriptions to match new behavior
- [ ] Simplify component mocks to only tested props
- [ ] Run full test suite to catch broken assertions
- [ ] Check for redundant helper functions or setup

## Pattern: Progressive Feature Implementation

When building features incrementally:

```tsx
// Stage 1: UI placeholder (disabled)
export function FeatureCard() {
  return (
    <Card>
      <h2>New Feature</h2>
      {/* TODO: Enable when /new-feature route is ready */}
      <button disabled>Launch Feature</button>
    </Card>
  );
}

// Stage 2: Add route, enable button
export function FeatureCard() {
  const router = useRouter();
  
  return (
    <Card>
      <h2>New Feature</h2>
      <button onClick={() => router.push('/new-feature')}>
        Launch Feature
      </button>
    </Card>
  );
}

// Stage 3: Add loading/error states
export function FeatureCard() {
  const router = useRouter();
  const [isNavigating, setIsNavigating] = useState(false);
  
  const handleClick = () => {
    setIsNavigating(true);
    router.push('/new-feature');
  };
  
  return (
    <Card>
      <h2>New Feature</h2>
      <button onClick={handleClick} disabled={isNavigating}>
        {isNavigating ? 'Loading...' : 'Launch Feature'}
      </button>
    </Card>
  );
}
```

**Benefits**:
- UI stays visually complete at all stages
- No 404 errors for users
- Clear progression path
- Easy to enable once route is ready

## Related Documentation

- [React Component Patterns](../react/)
- [Testing Semantic Behavior](../testing/testing-semantic-behavior-not-implementation.md)
- [Code Quality Rules](../quick-reference/code-quality-rules.md)
- [PR Review Checklist](../quick-reference/pr-review-checklist.md)

---

**Source**: Bug Fix PR Review  
**Date**: January 2026  
**Topics**: React, Testing, Navigation, Code Quality, KISS Principle
