# Conditional Queries in React Query

## Overview

Conditional queries allow you to control **when** a query should execute, preventing unnecessary API calls and avoiding errors in scenarios where the query shouldn't run yet.

**Common Use Cases:**
- Don't fetch user data until authentication is confirmed
- Don't query dependent data until parent data is loaded
- Don't make API calls when required parameters are missing
- Skip queries based on feature flags or user permissions

---

## The Problem: Unconditional Queries

### Bad Pattern ❌

```tsx
function UserProfile() {
  const { user, isLoading: isAuthLoading } = useAuth();

  // ⚠️ This query runs immediately, even if user is null!
  const { data: profile } = useQuery({
    queryKey: ['profile'],
    queryFn: () => fetchUserProfile(), // Will fail with 401 if not authenticated
  });

  if (isAuthLoading) return <Loading />;
  if (!user) return <LoginPrompt />;

  return <div>{profile?.name}</div>;
}
```

**Problems:**
1. Query executes before we know if user is authenticated
2. Triggers 401 errors that might show error modals
3. Wastes API calls that will always fail
4. Poor user experience with error flashes

---

## The Solution: The `enabled` Option

React Query provides an `enabled` option to conditionally run queries.

### Good Pattern ✅

```tsx
function UserProfile() {
  const { user, isLoading: isAuthLoading } = useAuth();

  // ✅ Query only runs when user exists
  const { data: profile, isLoading: isProfileLoading } = useQuery({
    queryKey: ['profile'],
    queryFn: () => fetchUserProfile(),
    enabled: !!user, // Only run when user is truthy
  });

  if (isAuthLoading || (user && isProfileLoading)) {
    return <Loading />;
  }

  if (!user) return <LoginPrompt />;

  return <div>{profile?.name}</div>;
}
```

**Benefits:**
- No unnecessary API calls
- No 401 errors for unauthenticated users
- Cleaner error handling
- Better performance

---

## Understanding Query States with `enabled`

When a query is disabled (`enabled: false`):

```tsx
const { data, isLoading, isFetching, isError } = useQuery({
  queryKey: ['data'],
  queryFn: fetchData,
  enabled: false,
});

// Query states when disabled:
// - isLoading: false
// - isFetching: false
// - isError: false
// - data: undefined
// - status: 'pending' (but not actively fetching)
```

**Important:** Even when disabled, the query is initialized - it just won't fetch until `enabled` becomes `true`.

---

## Common Patterns

### 1. Dependent Queries (Sequential Data Fetching)

```tsx
function UserOrders() {
  // First query: Get user
  const { data: user } = useQuery({
    queryKey: ['user'],
    queryFn: fetchUser,
  });

  // Second query: Only run after user is loaded
  const { data: orders } = useQuery({
    queryKey: ['orders', user?.id],
    queryFn: () => fetchOrders(user.id),
    enabled: !!user?.id, // Wait for user.id to exist
  });

  return <OrderList orders={orders} />;
}
```

### 2. Conditional on Multiple Factors

```tsx
function SearchResults() {
  const { user } = useAuth();
  const [searchTerm, setSearchTerm] = useState('');
  const [hasConsent, setHasConsent] = useState(false);

  const { data: results } = useQuery({
    queryKey: ['search', searchTerm],
    queryFn: () => searchAPI(searchTerm),
    enabled:
      !!user &&                    // User is authenticated
      searchTerm.length > 2 &&     // Search term is valid
      hasConsent,                  // User gave consent
  });

  return <Results data={results} />;
}
```

### 3. Feature Flag Gating

```tsx
function NewFeature() {
  const { data: featureFlags } = useQuery({
    queryKey: ['features'],
    queryFn: fetchFeatureFlags,
  });

  const { data: betaData } = useQuery({
    queryKey: ['beta-feature'],
    queryFn: fetchBetaFeature,
    enabled: featureFlags?.betaEnabled === true,
  });

  return betaData ? <BetaComponent /> : <StandardComponent />;
}
```

### 4. Pagination with Validation

```tsx
function PaginatedList() {
  const [page, setPage] = useState(1);
  const [pageSize] = useState(20);

  const { data, isLoading } = useQuery({
    queryKey: ['items', page, pageSize],
    queryFn: () => fetchItems({ page, pageSize }),
    enabled: page > 0 && pageSize > 0, // Validate params
    keepPreviousData: true, // Smooth pagination
  });

  return <List items={data?.items} />;
}
```

---

## Creating Reusable Conditional Hooks

### Pattern: Hook with Optional `enabled` Parameter

```tsx
interface UseUserDataOptions {
  enabled?: boolean;
}

interface UseUserDataResult {
  data: User | null;
  isLoading: boolean;
  isError: boolean;
}

/**
 * Fetch current user data
 * @param options.enabled - Control when the query runs (default: true)
 */
function useUserData(options?: UseUserDataOptions): UseUserDataResult {
  const { enabled = true } = options ?? {};

  const { data, isLoading, error } = useQuery({
    queryKey: ['user', 'me'],
    queryFn: fetchCurrentUser,
    enabled,
  });

  // Return safe defaults when disabled
  if (!enabled) {
    return {
      data: null,
      isLoading: false,
      isError: false,
    };
  }

  return {
    data: data ?? null,
    isLoading,
    isError: !!error,
  };
}
```

**Usage:**

```tsx
// In a protected route component
function ProtectedRoute({ children }) {
  const { user, isLoading: isAuthLoading } = useAuth();

  // Only fetch user data after authentication is confirmed
  const { data: userData } = useUserData({
    enabled: !!user
  });

  if (isAuthLoading || (user && !userData)) {
    return <Loading />;
  }

  if (!user) {
    return <LoginRequired />;
  }

  return <>{children}</>;
}
```

---

## Best Practices

### ✅ DO

1. **Use `enabled` for dependent queries**
   ```tsx
   enabled: !!prerequisiteData
   ```

2. **Validate required parameters**
   ```tsx
   enabled: !!userId && userId.length > 0
   ```

3. **Return sensible defaults when disabled**
   ```tsx
   if (!enabled) {
     return { data: null, isLoading: false, isError: false };
   }
   ```

4. **Combine with error boundaries for auth failures**
   ```tsx
   // Let 401s propagate to global error handler
   // Use enabled to prevent them in the first place
   ```

5. **Document why queries are conditional**
   ```tsx
   // Only fetch after user accepts terms to avoid 403 errors
   enabled: hasAcceptedTerms
   ```

### ❌ DON'T

1. **Don't use `enabled` for data that should always load**
   ```tsx
   // Bad: Creates unnecessary loading states
   enabled: true
   ```

2. **Don't forget to handle the disabled state in your UI**
   ```tsx
   // Bad: Shows wrong loading state
   if (isLoading) return <Loading />; // isLoading is false when disabled!
   ```

3. **Don't use `enabled` as a replacement for proper error handling**
   ```tsx
   // Bad: Hiding real errors
   enabled: !hasError
   ```

4. **Don't make conditions too complex**
   ```tsx
   // Bad: Hard to debug
   enabled: !!a && (b || c) && !d && (e?.f?.g ?? h)

   // Good: Extract to named variable
   const shouldFetch = !!a && (b || c) && !d && (e?.f?.g ?? h);
   enabled: shouldFetch
   ```

---

## Real-World Example: AuthGuard Pattern

This pattern protects routes and prevents unnecessary API calls for unauthenticated users.

```tsx
// hooks/useTermsCheck.ts
interface UseTermsCheckOptions {
  enabled?: boolean;
}

function useTermsCheck(options?: UseTermsCheckOptions) {
  const { enabled = true } = options ?? {};

  const { data, isLoading, error } = useQuery({
    queryKey: ['user', 'terms-status'],
    queryFn: fetchTermsStatus,
    enabled,
    staleTime: 0, // Always fresh check for terms
  });

  if (!enabled) {
    return { hasAccepted: false, isLoading: false, isError: false };
  }

  return {
    hasAccepted: data?.hasAccepted ?? false,
    isLoading,
    isError: !!error,
  };
}

// components/AuthGuard.tsx
function AuthGuard({ children }) {
  const { user, isLoading: isAuthLoading } = useAuth();
  const router = useRouter();

  // Only check terms after auth is confirmed
  const { hasAccepted, isLoading: isCheckingTerms } = useTermsCheck({
    enabled: !!user
  });

  useEffect(() => {
    if (!isAuthLoading && user && !isCheckingTerms) {
      if (!hasAccepted) {
        router.push('/accept-terms');
      }
    }
  }, [user, isAuthLoading, hasAccepted, isCheckingTerms, router]);

  // Show loading while checking auth or terms
  if (isAuthLoading || (user && isCheckingTerms)) {
    return <Loading />;
  }

  // Not authenticated
  if (!user) {
    return <LoginRequired />;
  }

  // Terms not accepted (will redirect)
  if (!hasAccepted) {
    return <Loading />;
  }

  return <>{children}</>;
}
```

**Why this works:**
1. No API call until `user` exists → No 401 errors
2. Clear separation of auth and terms checking
3. Proper loading states for each phase
4. Reusable hook that can be used elsewhere

---

## Performance Implications

### Query Lifecycle with `enabled`

```tsx
// Initial mount: enabled = false
const query1 = useQuery({
  queryKey: ['data'],
  queryFn: fetchData,
  enabled: false,
});
// Status: 'pending', but no network request

// Condition becomes true: enabled = true
const query2 = useQuery({
  queryKey: ['data'],
  queryFn: fetchData,
  enabled: true,
});
// Status: 'pending' → 'loading' → 'success'
// Network request fires

// Condition becomes false again: enabled = false
const query3 = useQuery({
  queryKey: ['data'],
  queryFn: fetchData,
  enabled: false,
});
// Status: remains 'success', data persists
// No new network request
```

**Key Insight:** When `enabled` changes from `true` to `false`, cached data remains accessible. The query just stops refetching.

---

## TypeScript Tips

### Type-Safe Conditional Queries

```tsx
interface User {
  id: string;
  name: string;
}

interface Order {
  id: string;
  userId: string;
  total: number;
}

function useUserOrders(userId: string | undefined) {
  return useQuery<Order[], Error>({
    queryKey: ['orders', userId],
    queryFn: () => {
      // TypeScript knows userId might be undefined
      if (!userId) {
        throw new Error('User ID is required');
      }
      return fetchOrders(userId);
    },
    enabled: !!userId, // Type guard
  });
}
```

### With Zod Validation

```tsx
import { z } from 'zod';

const UserIdSchema = z.string().uuid();

function useUserOrders(userId: string | undefined) {
  const isValidUserId = userId && UserIdSchema.safeParse(userId).success;

  return useQuery({
    queryKey: ['orders', userId],
    queryFn: () => fetchOrders(userId!), // Safe because of enabled
    enabled: isValidUserId,
  });
}
```

---

## Debugging Tips

### 1. Log Query State

```tsx
const query = useQuery({
  queryKey: ['data'],
  queryFn: fetchData,
  enabled: condition,
});

useEffect(() => {
  console.log('Query State:', {
    enabled: condition,
    status: query.status,
    isLoading: query.isLoading,
    isFetching: query.isFetching,
    data: query.data,
  });
}, [condition, query.status]);
```

### 2. Use React Query Devtools

```tsx
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <YourApp />
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

The devtools show:
- Whether queries are enabled/disabled
- When queries are fetching
- Cached data and staleness
- Query dependencies

### 3. Test Different States

```tsx
// Test helper to force query states
function TestWrapper({ enabled, children }) {
  return (
    <QueryClientProvider client={testQueryClient}>
      {children}
    </QueryClientProvider>
  );
}

it('should not fetch when disabled', () => {
  const { result } = renderHook(
    () => useData({ enabled: false }),
    { wrapper: TestWrapper }
  );

  expect(result.current.isLoading).toBe(false);
  expect(fetchDataSpy).not.toHaveBeenCalled();
});
```

---

## Summary

**Key Takeaways:**

1. Use `enabled` to control when queries execute
2. Prevent unnecessary API calls and errors
3. Essential for authentication flows and dependent queries
4. Return sensible defaults when queries are disabled
5. Document why queries are conditional
6. Test both enabled and disabled states

**The Golden Rule:**
> If your query depends on runtime conditions (auth, user input, feature flags), use the `enabled` option.

---

## Further Reading

- [React Query: Dependent Queries](https://tanstack.com/query/latest/docs/react/guides/dependent-queries)
- [React Query: Disabling Queries](https://tanstack.com/query/latest/docs/react/guides/disabling-queries)
- [React Query: Query Keys](https://tanstack.com/query/latest/docs/react/guides/query-keys)
