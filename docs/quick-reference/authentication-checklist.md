# Authentication Security Checklist

Quick reference for implementing and reviewing authentication in Next.js applications.

## Pre-Implementation Checklist

### Understanding Your Auth Provider

- [ ] Understand what your middleware actually does (e.g., Auth0 middleware doesn't auto-protect routes)
- [ ] Read official documentation for your auth provider + framework combination
- [ ] Identify which routes need protection
- [ ] Choose protection strategy (layout-level vs page-level)

## Implementation Checklist

### Server-Side Protection

- [ ] **CRITICAL:** Use server-side authentication checks only
- [ ] Protect at layout level for route groups
- [ ] Use `async` Server Components with `getSession()` or equivalent
- [ ] Redirect unauthenticated users to login
- [ ] Never rely on client-side checks alone

### Code Quality

- [ ] No authentication logic duplication across pages
- [ ] TypeScript types are correct (no `any`)
- [ ] Proper error handling for auth failures
- [ ] Clear comments explaining security approach

## Testing Checklist

### Must-Have Tests

- [ ] Test: Unauthenticated users are redirected
- [ ] Test: Authenticated users can access protected routes
- [ ] Test: Invalid/expired sessions are rejected
- [ ] All tests pass in CI

### Testing Principles

- [ ] Keep tests simple and focused
- [ ] Test security behavior, not implementation
- [ ] Mock external auth providers (Auth0, etc.)
- [ ] Don't over-test - focus on high-value scenarios

## Code Review Checklist

### Security Review

- [ ] All protected routes have server-side auth checks
- [ ] No authentication bypass paths
- [ ] Middleware configuration is correct
- [ ] Session validation is proper (check for null/undefined)
- [ ] Redirect paths are correct

### Common Vulnerabilities

- [ ] **Check:** Client-only authentication (easily bypassed)
- [ ] **Check:** Missing auth checks in new routes
- [ ] **Check:** Assuming middleware protects all routes
- [ ] **Check:** Missing null checks on session/user objects
- [ ] **Check:** Overly complex middleware logic

## Anti-Patterns to Avoid

### ❌ DON'T

```typescript
// DON'T: Client-side only check
'use client';
export function ProtectedPage() {
  const { user } = useUser();
  if (!user) return <Redirect to="/login" />;
  // This can be bypassed!
}

// DON'T: Duplicate checks everywhere
export async function Page1() {
  const session = await auth0.getSession();
  if (!session?.user) redirect('/login');
  // Repeated in every page...
}

// DON'T: Complex middleware for route protection
export async function middleware(request) {
  // Checking every route in middleware is inefficient
  const session = await getSession();
  if (!session && request.nextUrl.pathname.startsWith('/app')) {
    return redirect('/login');
  }
}
```

### ✅ DO

```typescript
// DO: Server-side check in layout
export default async function AppLayout({ children }) {
  const session = await auth0.getSession();

  if (!session?.user) {
    redirect('/login');
  }

  return <AppLayout>{children}</AppLayout>;
}

// DO: Simple tests for critical behavior
it('redirects when no session', async () => {
  mockGetSession.mockResolvedValue(null);
  await AppLayout({ children: <div>Test</div> });
  expect(redirect).toHaveBeenCalledWith('/login');
});
```

## Quick Reference: Auth0 + Next.js App Router

### What Auth0 Middleware Does

✅ Handles `/auth/login`, `/auth/callback`, `/auth/logout`  
✅ Manages session cookies  
✅ Supports rolling sessions  
❌ Does NOT redirect unauthenticated users  
❌ Does NOT protect routes automatically

### Correct Protection Pattern

```typescript
// 1. Layout: Server-side auth check
export default async function Layout({ children }) {
  const session = await auth0.getSession();
  if (!session?.user) redirect('/login');
  return <>{children}</>;
}

// 2. Test: Verify redirect behavior
it('redirects unauthenticated users', async () => {
  mockGetSession.mockResolvedValue(null);
  await Layout({ children: <div>Test</div> });
  expect(redirect).toHaveBeenCalledWith('/login');
});

// 3. Middleware: Only Auth0 routes
export async function middleware(request) {
  return await auth0.middleware(request);
}
```

## Resources

- [Protecting Next.js App Router Routes (Full Guide)](../authentication/protecting-nextjs-app-router-routes.md)
- [Testing Best Practices](../testing/README.md)
- [Auth0 Next.js Documentation](https://auth0.github.io/nextjs-auth0/)

## Version History

- **2026-01-29:** Initial checklist based on security vulnerability fix
