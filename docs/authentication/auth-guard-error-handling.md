# Authentication Guard Error Handling: Avoiding Redirect Loops

## Problem

When implementing authentication guards that fetch user data, a common mistake is to redirect to the login page on **any** error. This causes infinite redirect loops when transient errors occur:

```
500 error → redirect to /login → auto-login → back to protected page → 500 error → ...
```

## The Issue

**❌ Bad: Redirect on all errors**

```typescript
const { data: user, error } = useQuery(['user'], fetchCurrentUser);

useEffect(() => {
  if (error) {
    router.push('/login'); // ⚠️ Problem: ALL errors trigger redirect
  }
}, [error, router]);
```

**Problems with this approach:**
- **500 Internal Server Error** → redirects unnecessarily
- **504 Gateway Timeout** → redirects unnecessarily
- **429 Rate Limit** → redirects unnecessarily
- **Network failures** → redirects unnecessarily
- User gets stuck in a loop if Auth0/OAuth auto-logs them back in

## The Solution

**✅ Good: Only redirect on 401 Unauthorized**

```typescript
const { data: user, error } = useQuery(['user'], fetchCurrentUser);

useEffect(() => {
  if (error?.statusCode === 401) {
    router.push('/login'); // ✓ Only unauthorized users redirected
  }
}, [error, router]);
```

**Why this works:**
- `401` = User is not authenticated → redirect to login ✓
- `500` = Server error → show error state, allow retry ✓
- `503` = Service unavailable → show error state, allow retry ✓
- Network issues → show error state, allow retry ✓

## Implementation Pattern

### 1. Return Error Object from Hooks

Your data-fetching hooks should return the full error object, not just a boolean:

```typescript
// ❌ Bad: Only boolean
interface UseAuthResult {
  user: User | null;
  isLoading: boolean;
  isError: boolean; // ⚠️ Can't check error type
}

// ✅ Good: Full error object
interface UseAuthResult {
  user: User | null;
  isLoading: boolean;
  error: ApiError | null; // ✓ Can check statusCode
}
```

### 2. Check Specific Status Codes

```typescript
function AuthGuard({ children }) {
  const { user, isLoading, error } = useCurrentUser();
  const router = useRouter();

  useEffect(() => {
    if (!isLoading && error?.statusCode === 401) {
      router.push('/login');
    }
  }, [isLoading, error, router]);

  if (isLoading) {
    return <LoadingSpinner />;
  }

  // Don't render protected content on auth errors
  if (error?.statusCode === 401) {
    return null; // Redirecting...
  }

  // Show error state for non-auth errors
  if (error) {
    return <ErrorState message="Failed to load user data" retry={refetch} />;
  }

  if (!user) {
    return <Unauthorized />;
  }

  return <>{children}</>;
}
```

### 3. Apply to Pages Outside Layout Guards

Pages that sit outside your main layout (e.g., onboarding flows) need their own auth checks:

```typescript
export default function OnboardingPage() {
  const { user, isLoading, error } = useCurrentUser();
  const router = useRouter();

  // Redirect only on 401
  useEffect(() => {
    if (!isLoading && error?.statusCode === 401) {
      router.push('/login');
    }
  }, [isLoading, error, router]);

  // Show loading while checking auth
  if (isLoading || !user) {
    return <LoadingSpinner />;
  }

  return <OnboardingForm />;
}
```

## Error Handling Strategy

| Status Code | Meaning | Action |
|------------|---------|--------|
| 401 | Unauthorized | Redirect to `/login` |
| 403 | Forbidden | Show "Access Denied" message |
| 500 | Server Error | Show error + retry button |
| 503 | Service Unavailable | Show error + retry button |
| Network Error | Connection Issue | Show error + retry button |

## Testing Considerations

When testing auth guards, verify behavior for different error types:

```typescript
describe('AuthGuard', () => {
  it('redirects to login on 401', () => {
    mockQuery.mockReturnValue({ error: { statusCode: 401 } });
    render(<AuthGuard><Content /></AuthGuard>);
    expect(mockRouter.push).toHaveBeenCalledWith('/login');
  });

  it('shows error state on 500', () => {
    mockQuery.mockReturnValue({ error: { statusCode: 500 } });
    const { getByText } = render(<AuthGuard><Content /></AuthGuard>);
    expect(getByText(/failed to load/i)).toBeInTheDocument();
    expect(mockRouter.push).not.toHaveBeenCalled();
  });

  it('shows error state on network failure', () => {
    mockQuery.mockReturnValue({ error: { message: 'Network Error' } });
    const { getByText } = render(<AuthGuard><Content /></AuthGuard>);
    expect(getByText(/failed to load/i)).toBeInTheDocument();
    expect(mockRouter.push).not.toHaveBeenCalled();
  });
});
```

## Common Mistakes

### 1. Using retry: true with redirect logic

```typescript
// ❌ Bad: Retry conflicts with redirect
const { error } = useQuery(['user'], fetchUser, {
  retry: 3, // ⚠️ Will retry 401s before redirecting
});

useEffect(() => {
  if (error?.statusCode === 401) {
    router.push('/login');
  }
}, [error]);
```

**Fix:** Use `retry: false` for auth queries

```typescript
// ✅ Good: No retry on auth failures
const { error } = useQuery(['user'], fetchUser, {
  retry: false, // ✓ Immediate redirect on 401
});
```

### 2. Not handling loading states

```typescript
// ❌ Bad: Redirect fires during loading
useEffect(() => {
  if (error?.statusCode === 401) {
    router.push('/login'); // ⚠️ Fires even when still loading
  }
}, [error]);

// ✅ Good: Wait for loading to complete
useEffect(() => {
  if (!isLoading && error?.statusCode === 401) {
    router.push('/login'); // ✓ Only after load completes
  }
}, [isLoading, error]);
```

### 3. Showing protected content during redirect

```typescript
// ❌ Bad: User sees protected content briefly
if (error?.statusCode === 401) {
  router.push('/login');
}
return <ProtectedContent />; // ⚠️ Flashes before redirect

// ✅ Good: Show loading state during redirect
if (error?.statusCode === 401) {
  router.push('/login');
  return <LoadingSpinner />; // ✓ No flash of content
}
```

## Real-World Scenarios

### Scenario 1: Temporary Backend Outage

**With bad pattern (redirect on all errors):**
1. Backend goes down (500 errors)
2. All users get redirected to login
3. Auth0 auto-logs them back in
4. They're redirected to dashboard
5. Backend still down → 500 → redirect again
6. **Infinite loop**

**With good pattern (401 only):**
1. Backend goes down (500 errors)
2. Users see "Service temporarily unavailable" message
3. They can retry or wait
4. Backend comes back online
5. **No redirect loop**

### Scenario 2: Rate Limiting

**With bad pattern:**
- App makes too many requests → 429 rate limit
- User gets logged out unnecessarily
- They have to log back in
- Bad UX

**With good pattern:**
- App makes too many requests → 429 rate limit
- User sees "Too many requests, please wait" message
- After rate limit window, they can retry
- No logout required

## Key Takeaways

1. ✅ **Only redirect on 401** - This is the only status that means "not authenticated"
2. ✅ **Return full error objects** - Allows checking specific status codes
3. ✅ **Show error states for transient failures** - Give users a way to retry
4. ✅ **Wait for loading to complete** - Check `!isLoading` before redirecting
5. ✅ **Use `retry: false` for auth queries** - Avoid unnecessary retries on 401
6. ✅ **Test different error types** - Verify behavior for 401, 500, network errors

## Related Patterns

- [Protecting Next.js App Router Routes](./protecting-nextjs-app-router-routes.md)
- [Global Error Handling Architecture](../error-handling/global-error-handling-architecture.md)
- [Conditional Queries with React Query](../state-management/conditional-queries-react-query.md)

## References

- HTTP Status Codes: [MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)
- React Query Error Handling: [TanStack Query Docs](https://tanstack.com/query/latest/docs/react/guides/query-functions#handling-and-throwing-errors)
