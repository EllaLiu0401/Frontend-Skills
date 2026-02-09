# Global Error Handling Architecture

## Overview

When building React applications with data fetching libraries like TanStack Query (React Query), establishing a **centralized, global error handling pipeline** is crucial for consistent user experience and maintainable code.

## The Anti-Pattern: Local Error Handling Everywhere

❌ **What NOT to do:**

```typescript
// Bad: Each component manages its own errors
function UserProfile() {
  const [error, setError] = useState<string | null>(null);

  const mutation = useMutation({
    mutationFn: updateProfile,
    onSuccess: () => {
      toast.success('Profile updated!');
    },
    onError: (err) => {
      setError(err.message);  // ❌ Local error state
    },
  });

  return (
    <div>
      {error && <Alert type="error">{error}</Alert>}  {/* ❌ Local error UI */}
      <form onSubmit={() => mutation.mutate(data)}>
        {/* form fields */}
      </form>
    </div>
  );
}
```

**Problems with this approach:**
1. **Inconsistent UX** - Different pages show errors differently
2. **Code duplication** - Error handling logic repeated everywhere
3. **Difficult to maintain** - Changes require updating multiple files
4. **Memory leaks risk** - Error state not always properly cleaned up
5. **Harder to test** - Each component needs error state testing

## The Better Pattern: Global Error Pipeline

✅ **Recommended Architecture:**

```
┌─────────────────────────────────────────────────────────┐
│                    Component Layer                       │
│  (useQuery, useMutation - NO local error handling)      │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                   Query/Mutation Cache                   │
│        (Global onError handler intercepts all)           │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                     Error Store                          │
│              (Global reactive state)                     │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│               Global Error Display Component             │
│            (Modal, Toast, or Banner)                     │
└─────────────────────────────────────────────────────────┘
```

## Implementation Example

### 1. Error Wrapper Class

```typescript
// lib/errors.ts
export class ApiErrorWrapper extends Error {
  constructor(
    readonly code: string,
    readonly userMessage: string,
    readonly details?: unknown
  ) {
    super(userMessage);
    this.name = 'ApiErrorWrapper';
  }
}
```

### 2. Query Client with Global Error Handler

```typescript
// providers/QueryProvider.tsx
import { QueryClient, QueryClientProvider, MutationCache, QueryCache } from '@tanstack/react-query';
import { ApiErrorWrapper } from '@/lib/errors';
import { setGlobalError } from '@/stores/errorStore';

function useGlobalErrorHandler() {
  return (error: unknown) => {
    console.error('[Query Error]', error);

    if (error instanceof ApiErrorWrapper) {
      // Send to global error store
      setGlobalError(error);
    } else {
      // Handle unexpected errors
      setGlobalError(new ApiErrorWrapper(
        'unknown_error',
        'An unexpected error occurred'
      ));
    }
  };
}

export function QueryProvider({ children }: { children: React.ReactNode }) {
  const handleError = useGlobalErrorHandler();

  const [queryClient] = useState(
    () => new QueryClient({
      queryCache: new QueryCache({
        onError: handleError,  // ✅ Global query error handler
      }),
      mutationCache: new MutationCache({
        onError: handleError,  // ✅ Global mutation error handler
      }),
      defaultOptions: {
        queries: {
          staleTime: 30000,
          retry: 2,
          refetchOnWindowFocus: false,
        },
      },
    })
  );

  return (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
}
```

### 3. Global Error Store

```typescript
// stores/errorStore.ts
import { create } from 'zustand';
import { ApiErrorWrapper } from '@/lib/errors';

interface ErrorState {
  currentError: ApiErrorWrapper | null;
  setError: (error: ApiErrorWrapper) => void;
  clearError: () => void;
}

export const useErrorStore = create<ErrorState>((set) => ({
  currentError: null,
  setError: (error) => set({ currentError: error }),
  clearError: () => set({ currentError: null }),
}));

// Convenience functions
export const setGlobalError = (error: ApiErrorWrapper) => {
  useErrorStore.getState().setError(error);
};

export const clearGlobalError = () => {
  useErrorStore.getState().clearError();
};
```

### 4. Global Error Display Component

```typescript
// components/GlobalErrorModal.tsx
import { useEffect } from 'react';
import { useErrorStore } from '@/stores/errorStore';

export function GlobalErrorModal() {
  const { currentError, clearError } = useErrorStore();
  const [localError, setLocalError] = useState<ApiErrorWrapper | null>(null);

  // Subscribe to error store
  useEffect(() => {
    if (currentError) {
      setLocalError(currentError);
      clearError(); // Clear immediately so next error can be received
    }
  }, [currentError, clearError]);

  if (!localError) return null;

  return (
    <Modal
      open={!!localError}
      onClose={() => setLocalError(null)}
      title="Error"
    >
      <div>
        <p className="font-medium">{localError.userMessage}</p>
        {process.env.NODE_ENV === 'development' && (
          <details className="mt-4">
            <summary>Debug Info</summary>
            <pre className="text-xs">
              {JSON.stringify(localError.details, null, 2)}
            </pre>
          </details>
        )}
      </div>
      <button onClick={() => setLocalError(null)}>Close</button>
    </Modal>
  );
}
```

### 5. App Provider Setup

```typescript
// app/providers.tsx
export function AppProviders({ children }: { children: React.ReactNode }) {
  return (
    <QueryProvider>
      <GlobalErrorModal />  {/* ✅ Single error display component */}
      {children}
    </QueryProvider>
  );
}
```

### 6. Component Usage (Clean!)

```typescript
// components/UserProfile.tsx
function UserProfile() {
  // ✅ No local error state
  // ✅ No onError handler
  // ✅ Errors automatically handled globally

  const mutation = useMutation({
    mutationFn: updateProfile,
    onSuccess: () => {
      toast.success('Profile updated!');
      // Only handle SUCCESS locally - errors go to global handler
    },
    // ❌ NO onError here!
  });

  return (
    <form onSubmit={() => mutation.mutate(data)}>
      {/* ✅ No Alert component needed */}
      {/* ✅ Clean component focused on happy path */}
      <button type="submit" disabled={mutation.isPending}>
        {mutation.isPending ? 'Saving...' : 'Save'}
      </button>
    </form>
  );
}
```

## Benefits of Global Error Handling

### 1. Consistency
- All errors displayed in the same way
- Users get familiar with error patterns
- Easier to meet accessibility requirements

### 2. Maintainability
- Single place to update error display logic
- Easy to add features (error reporting, logging)
- Less code duplication

### 3. Separation of Concerns
- **Components**: Focus on happy path and business logic
- **Error Handler**: Focus on error display and recovery
- **Store**: Focus on state management

### 4. Flexibility
- Easy to switch between modal, toast, or banner
- Can add conditional logic (e.g., different display for certain errors)
- Can integrate with error reporting services (Sentry, LogRocket)

### 5. Testing
- Mock error store for component tests
- Test error display component separately
- Easier to test error scenarios

## When to Deviate from Global Handling

There are **rare cases** where local error handling makes sense:

### ✅ Acceptable Local Error Handling

```typescript
// Acceptable: Form validation errors that need inline display
function LoginForm() {
  const [validationErrors, setValidationErrors] = useState<Record<string, string>>({});

  const mutation = useMutation({
    mutationFn: login,
    onSuccess: () => router.push('/dashboard'),
    onError: (error) => {
      // ✅ OK: Validation errors need inline display next to fields
      if (error.code === 'validation_error') {
        setValidationErrors(error.fieldErrors);
        return; // Don't send to global handler
      }
      // All other errors will bubble to global handler automatically
    },
  });

  return (
    <form>
      <input name="email" />
      {validationErrors.email && <span className="error">{validationErrors.email}</span>}
      {/* ... */}
    </form>
  );
}
```

**Rule of thumb:** Only handle errors locally if:
1. You need **inline, field-specific** error messages
2. The error requires **immediate, contextual** user action
3. You still document why you're deviating from the pattern

## Common Mistakes to Avoid

### ❌ Mistake 1: Mixing Local and Global

```typescript
// ❌ BAD: Error will show in BOTH places
const mutation = useMutation({
  mutationFn: updateUser,
  onError: (err) => {
    setLocalError(err.message); // Shows in component
    // Also triggers global handler - duplicate error display!
  },
});
```

### ❌ Mistake 2: Not Clearing Error State

```typescript
// ❌ BAD: Old errors can reappear
function GlobalErrorModal() {
  const { currentError } = useErrorStore();
  // Missing: clearError() call after showing error
  // Result: Error shows again on next navigation
}
```

### ❌ Mistake 3: Leaking Technical Details

```typescript
// ❌ BAD: Showing technical error to user
onError: (err) => {
  setError(err.stack); // Never show stack traces to users!
}

// ✅ GOOD: User-friendly message
onError: (err) => {
  setGlobalError(new ApiErrorWrapper(
    err.code,
    'Unable to save your changes. Please try again.'
  ));
}
```

## Integration with Error Reporting

```typescript
// Enhanced global error handler with reporting
function useGlobalErrorHandler() {
  return (error: unknown) => {
    // 1. Log to console for development
    console.error('[Query Error]', error);

    // 2. Send to error reporting service
    if (process.env.NODE_ENV === 'production') {
      Sentry.captureException(error, {
        tags: { source: 'query-cache' },
      });
    }

    // 3. Display to user
    if (error instanceof ApiErrorWrapper) {
      setGlobalError(error);
    }
  };
}
```

## Architecture Checklist

When implementing global error handling:

- [ ] Single error store/state
- [ ] Global error handler in QueryCache/MutationCache
- [ ] Single error display component
- [ ] Components use ONLY `onSuccess` (no `onError`)
- [ ] Error wrapper class for type safety
- [ ] User-friendly messages (no technical details)
- [ ] Development mode shows debug info
- [ ] Integration with error reporting service
- [ ] Documented exceptions to the pattern
- [ ] Tests for error display component

## Key Takeaways

1. **Centralize error handling** - Don't scatter `onError` handlers throughout your app
2. **Global error pipeline** - QueryCache → Error Store → Display Component
3. **Components stay clean** - Focus on business logic, not error display
4. **Consistent UX** - Users see errors the same way everywhere
5. **Document exceptions** - If you deviate, explain why in comments

## Further Reading

- [TanStack Query Error Handling](https://tanstack.com/query/latest/docs/react/guides/mutations#error-handling)
- [React Error Boundaries](https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary)
- [Zustand for Error State](https://github.com/pmndrs/zustand)
- [User-Centered Error Messages](https://www.nngroup.com/articles/error-message-guidelines/)

---

**Date Added:** 2026-02-09
**Source:** Code review learning - Global error handling architecture pattern
