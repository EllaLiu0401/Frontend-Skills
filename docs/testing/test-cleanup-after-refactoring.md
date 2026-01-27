# Test Cleanup After Refactoring

> **TL;DR**: When you remove functionality from a component, immediately remove related mocks, imports, and test cases. Keep tests lean and focused.

## Problem

After refactoring a component to remove functionality (e.g., removing a click handler, simplifying state), test files often retain:
- Unused mocks for removed dependencies
- Test cases for deleted behavior
- Redundant mock props and parameters
- Unnecessary imports

This creates **test debt**: code that doesn't test anything but still needs maintenance.

## Real-World Example

### Scenario: Removing Navigation from a Button

You're working on a dashboard card with a "View Reports" button. During review, you discover the `/analytics` route doesn't exist yet, so you disable the button instead of letting it cause 404 errors.

**Component Before**:
```tsx
import { useRouter } from 'next/navigation';

export function ReportsCard({ data }) {
  const router = useRouter();
  
  const handleClick = () => {
    router.push('/analytics');
  };
  
  return (
    <div>
      <h3>Reports</h3>
      {data.map(item => <div key={item.id}>{item.name}</div>)}
      <button onClick={handleClick}>View Reports</button>
    </div>
  );
}
```

**Component After**:
```tsx
// ✅ Router removed, button disabled
export function ReportsCard({ data }) {
  return (
    <div>
      <h3>Reports</h3>
      {data.map(item => <div key={item.id}>{item.name}</div>)}
      {/* Disabled until /analytics route exists */}
      <button disabled>View Reports</button>
    </div>
  );
}
```

### Test File Before Cleanup

```typescript
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event'; // ❌ No longer needed
import { describe, it, expect, vi } from 'vitest';
import { ReportsCard } from './ReportsCard';

// ❌ PROBLEM: Mock still exists for removed functionality
const mockPush = vi.fn();

vi.mock('next/navigation', () => ({
  useRouter: () => ({
    push: mockPush,
    replace: vi.fn(),
    back: vi.fn(),
    forward: vi.fn(),
    refresh: vi.fn(),
    prefetch: vi.fn(),
  }),
}));

// ❌ PROBLEM: Test cases for deleted behavior
describe('ReportsCard', () => {
  const mockData = [
    { id: '1', name: 'Report 1' },
    { id: '2', name: 'Report 2' },
  ];

  it('renders report list', () => {
    render(<ReportsCard data={mockData} />);
    
    expect(screen.getByText('Report 1')).toBeInTheDocument();
    expect(screen.getByText('Report 2')).toBeInTheDocument();
  });

  // ❌ PROBLEM: This test is for removed functionality
  it('navigates to analytics when button clicked', async () => {
    const user = userEvent.setup();
    render(<ReportsCard data={mockData} />);
    
    const button = screen.getByText('View Reports');
    await user.click(button);
    
    expect(mockPush).toHaveBeenCalledWith('/analytics');
  });

  it('renders button', () => {
    render(<ReportsCard data={mockData} />);
    
    expect(screen.getByText('View Reports')).toBeInTheDocument();
  });
});
```

### Test File After Cleanup

```typescript
import { render, screen } from '@testing-library/react';
import { describe, it, expect } from 'vitest';
import { ReportsCard } from './ReportsCard';

// ✅ No more router mock - not needed!

describe('ReportsCard', () => {
  const mockData = [
    { id: '1', name: 'Report 1' },
    { id: '2', name: 'Report 2' },
  ];

  it('renders report list', () => {
    render(<ReportsCard data={mockData} />);
    
    expect(screen.getByText('Report 1')).toBeInTheDocument();
    expect(screen.getByText('Report 2')).toBeInTheDocument();
  });

  // ✅ Test updated to verify current behavior
  it('renders disabled View Reports button', () => {
    render(<ReportsCard data={mockData} />);
    
    const button = screen.getByText('View Reports');
    expect(button).toBeInTheDocument();
    expect(button).toBeDisabled(); // ✅ Test the actual state
  });
});
```

**Lines of code**: Reduced from ~45 to ~25 (44% reduction)  
**Maintenance burden**: Significantly reduced  
**Clarity**: Tests now only verify what the component actually does

---

## Step-by-Step Cleanup Process

### Step 1: Identify Removed Functionality

After refactoring your component, list what was removed:
- ❌ `useRouter` hook
- ❌ `handleClick` function
- ❌ `onClick` prop on button

### Step 2: Remove Related Mocks

Delete mocks for dependencies no longer used:

```typescript
// ❌ DELETE THIS
const mockPush = vi.fn();

vi.mock('next/navigation', () => ({
  useRouter: () => ({
    push: mockPush,
    // ... rest of mock
  }),
}));
```

### Step 3: Remove Unused Imports

```typescript
// ❌ DELETE THESE
import userEvent from '@testing-library/user-event';
// (if no other tests use it)
```

### Step 4: Delete Obsolete Test Cases

```typescript
// ❌ DELETE THIS ENTIRE TEST
it('navigates to analytics when button clicked', async () => {
  // ...
});
```

### Step 5: Update Related Test Cases

```typescript
// ❌ Before: Generic test
it('renders button', () => {
  render(<ReportsCard data={mockData} />);
  expect(screen.getByText('View Reports')).toBeInTheDocument();
});

// ✅ After: Specific test for current behavior
it('renders disabled View Reports button', () => {
  render(<ReportsCard data={mockData} />);
  
  const button = screen.getByText('View Reports');
  expect(button).toBeInTheDocument();
  expect(button).toBeDisabled();
});
```

### Step 6: Simplify Component Mocks

If you mock UI components, remove props you no longer test:

```typescript
// ❌ Before: Mock includes unused props
vi.mock('@ui/components', () => ({
  Button: ({ 
    children, 
    onClick,    // ❌ Not tested
    disabled,
    size,       // ❌ Not tested
    variant,    // ❌ Not tested
  }) => (
    <button onClick={onClick} disabled={disabled}>
      {children}
    </button>
  ),
}));

// ✅ After: Only mock what you test
vi.mock('@ui/components', () => ({
  Button: ({ children, disabled }) => (
    <button disabled={disabled}>
      {children}
    </button>
  ),
}));
```

---

## Common Cleanup Scenarios

### Scenario 1: Removing Form Submission

**Before Refactor**: Component had form with validation and submission  
**After Refactor**: Simplified to read-only display

**Cleanup checklist**:
- [ ] Remove form submission handler mocks
- [ ] Remove validation library mocks
- [ ] Delete test cases for form submission
- [ ] Delete test cases for validation errors
- [ ] Update tests to verify read-only display

### Scenario 2: Removing API Calls

**Before Refactor**: Component fetched data via API  
**After Refactor**: Receives data as props (lifted state up)

**Cleanup checklist**:
- [ ] Remove API client mocks
- [ ] Remove loading state tests
- [ ] Remove error state tests
- [ ] Remove API call verification tests
- [ ] Update tests to pass data via props

### Scenario 3: Simplifying Event Handlers

**Before Refactor**: Complex onClick with multiple side effects  
**After Refactor**: Button disabled or removed

**Cleanup checklist**:
- [ ] Remove `userEvent` import if no longer needed
- [ ] Remove click simulation tests
- [ ] Remove side effect verification
- [ ] Add disabled state test if button remains
- [ ] Remove button tests entirely if button removed

---

## Red Flags: Signs Your Tests Need Cleanup

🚩 **Test file has more mocks than tests**  
→ Likely has unused mocks for removed functionality

🚩 **Test assertions fail but component works fine**  
→ Tests are checking old behavior

🚩 **Multiple unused imports flagged by linter**  
→ Dependencies removed from component but not tests

🚩 **Test coverage drops after component simplification**  
→ Old tests testing deleted functionality

🚩 **Mock functions never get called (`expect(mock).not.toHaveBeenCalled()` always passes)**  
→ Mocks are for removed functionality

🚩 **Test descriptions don't match component behavior**  
→ Tests outdated after refactoring

---

## Best Practices

### ✅ DO

- **Remove mocks immediately** when removing dependencies from components
- **Update test descriptions** to match new behavior
- **Simplify mocks** to only include tested props
- **Run tests after cleanup** to ensure nothing broke
- **Review test coverage** to verify you're testing actual behavior

### ❌ DON'T

- Don't leave unused mocks "just in case" they're needed later
- Don't keep old test cases thinking they might be useful again
- Don't add `@ts-ignore` or `eslint-disable` to silence warnings about unused code
- Don't mock more than you need to test
- Don't test implementation details that no longer exist

---

## Checklist: Post-Refactor Test Cleanup

After refactoring a component:

- [ ] List all functionality removed from component
- [ ] Remove mocks for deleted dependencies
- [ ] Delete test cases for removed behavior
- [ ] Update test descriptions to match new behavior
- [ ] Simplify component mocks to only tested props
- [ ] Remove unused imports (`userEvent`, mocked libraries, etc.)
- [ ] Run test suite and verify all tests pass
- [ ] Check test coverage hasn't dropped unexpectedly
- [ ] Verify no linter warnings about unused code
- [ ] Read through test file to ensure it's easy to understand

---

## Tools to Help

### ESLint Rules

```json
{
  "rules": {
    "no-unused-vars": "error",
    "@typescript-eslint/no-unused-vars": "error",
    "jest/no-disabled-tests": "warn",
    "jest/no-focused-tests": "error"
  }
}
```

### Git Diff Review

Before committing:
```bash
git diff --cached
```

Look for:
- Mocks in tests that aren't in component anymore
- Test cases for removed functionality
- Imports in tests not used anywhere

### Test Coverage

Run coverage and look for:
- Functions that are no longer covered (might indicate old tests)
- Lines marked as covered but not actually tested
- Overall coverage changes after refactor

```bash
npm test -- --coverage
```

---

## Real Impact

### Before Cleanup
- **43 lines** in test file
- **3 mock functions** (1 never called)
- **15 imports** (4 unused)
- **3 test cases** (1 tests deleted feature)
- **Maintenance time**: 10 minutes to understand

### After Cleanup
- **24 lines** in test file (44% reduction)
- **0 mock functions** (none needed)
- **8 imports** (all used)
- **2 test cases** (both test current behavior)
- **Maintenance time**: 2 minutes to understand

**Result**: Tests are clearer, faster, and easier to maintain.

---

## Related Documentation

- [Testing Semantic Behavior](./testing-semantic-behavior-not-implementation.md)
- [Code Quality Rules](../quick-reference/code-quality-rules.md)
- [React Testing Best Practices](../react/)

---

**Date**: January 2026  
**Topics**: Testing, Refactoring, Code Quality, Technical Debt
