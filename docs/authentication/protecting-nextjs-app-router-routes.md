# Protecting Next.js App Router Routes with Auth0

**Date:** 2026-01-29  
**Context:** Security vulnerability analysis and fix  
**Tags:** `security`, `authentication`, `nextjs`, `auth0`, `server-components`

## Problem: Unprotected Route Group

A critical security vulnerability was discovered where an entire route group `(app)` was accessible without authentication. The layout component rendered the UI without checking if the user had a valid session.

### Symptoms

- All pages under `(app)/*` were accessible without login
- No authentication check in the shared layout
- Middleware only handled Auth0 routes, not route protection
- Client components gracefully degraded but didn't block access

### Why This is Critical

**Security Impact:**

- Unauthorized access to protected resources
- Potential data exposure
- Bypassed authorization checks

**Common Misconception:**
Many developers assume that `auth0.middleware()` automatically protects routes, but it does **not**. The middleware only:

- Handles Auth0 authentication routes (`/auth/login`, `/auth/callback`, etc.)
- Manages session cookies
- Supports rolling sessions

It does **not** redirect unauthenticated users from protected routes.

## Solution: Layout-Level Authentication

Protect the entire route group by adding server-side authentication checks in the layout component.

### Implementation

```typescript
// app/(app)/layout.tsx
import type { ReactNode } from 'react';
import { redirect } from 'next/navigation';
import { auth0 } from '@/lib/auth0-client';
import { AppLayout } from '@/components/layout/AppLayout';

interface AppGroupLayoutProps {
  readonly children: ReactNode;
}

export default async function AppGroupLayout({ children }: AppGroupLayoutProps) {
  // Server-side authentication check
  const session = await auth0.getSession();

  // Redirect to login if no valid session
  if (!session?.user) {
    redirect('/login');
  }

  return <AppLayout>{children}</AppLayout>;
}
```

### Why This Approach is Correct

According to [Auth0 official documentation](https://developer.auth0.com/resources/guides/web-app/nextjs/basic-authentication):

> **For protecting pages in the App Router:**
>
> - Use `getSession()` in server components to retrieve user authentication information
> - Apply protection at the page level **or layout level**

**Advantages:**

1. ✅ **Single point of protection** - All routes in the group are protected
2. ✅ **Server-side validation** - Cannot be bypassed by client-side manipulation
3. ✅ **Performance** - Only checks on actual page navigation, not on assets
4. ✅ **Type-safe** - Full TypeScript support with async Server Components
5. ✅ **Maintainable** - No need to repeat auth checks in every page

## Alternative Approaches (Not Recommended)

### ❌ Approach 1: Using `withMiddlewareAuthRequired()`

```typescript
// DON'T: Known issues in App Router
import { withMiddlewareAuthRequired } from "@auth0/nextjs-auth0/middleware";

export default withMiddlewareAuthRequired();
```

**Problems:**

- Not recommended for App Router
- Multiple known bugs (see Auth0 Community discussions)
- Doesn't support custom redirect paths
- Overrides custom logic

### ❌ Approach 2: Middleware-Level Checks

```typescript
// DON'T: Poor performance
export async function middleware(request: NextRequest) {
  const session = await auth0.getSession();
  if (!session && request.nextUrl.pathname.startsWith("/dashboard")) {
    return NextResponse.redirect(new URL("/login", request.url));
  }
  return await auth0.middleware(request);
}
```

**Problems:**

- Runs on every request (including static assets)
- Hard to maintain path lists
- Performance overhead
- Matcher configuration complexity

### ❌ Approach 3: Per-Page Checks

```typescript
// DON'T: Code duplication
export default async function DashboardPage() {
  const session = await auth0.getSession();
  if (!session?.user) redirect("/login");

  // ... rest of page
}
```

**Problems:**

- Violates DRY principle
- Easy to forget in new pages
- Maintenance burden
- Inconsistent implementation

## Testing Strategy

Write simple, high-value tests for authentication logic.

### Test File Structure

```typescript
// app/(app)/layout.test.tsx
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { redirect } from 'next/navigation';
import AppGroupLayout from './layout';

// Mock next/navigation
vi.mock('next/navigation', () => ({
  redirect: vi.fn(),
}));

// Mock Auth0 Client
vi.mock('@/lib/auth0-client', () => ({
  auth0: {
    getSession: vi.fn(),
  },
}));

// Mock AppLayout component
vi.mock('@/components/layout/AppLayout', () => ({
  AppLayout: ({ children }: { children: React.ReactNode }) => (
    <div data-testid="app-layout">{children}</div>
  ),
}));

const { auth0 } = await import('@/lib/auth0-client');

describe('AppGroupLayout', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  describe('Authentication', () => {
    it('should redirect to login when session is null', async () => {
      vi.mocked(auth0.getSession).mockResolvedValue(null);

      await AppGroupLayout({ children: <div>Test</div> });

      expect(redirect).toHaveBeenCalledWith('/login');
    });

    it('should redirect to login when session has no user', async () => {
      vi.mocked(auth0.getSession).mockResolvedValue({ user: null });

      await AppGroupLayout({ children: <div>Test</div> });

      expect(redirect).toHaveBeenCalledWith('/login');
    });

    it('should render when user is authenticated', async () => {
      const mockSession = {
        user: { email: 'test@example.com', sub: 'auth0|123' },
      };
      vi.mocked(auth0.getSession).mockResolvedValue(mockSession);

      const result = await AppGroupLayout({ children: <div>Test</div> });

      expect(redirect).not.toHaveBeenCalled();
      expect(result).toBeTruthy();
    });
  });
});
```

### Testing Principles

**Keep tests simple and valuable:**

- ✅ Test security-critical behavior
- ✅ Cover authentication scenarios
- ✅ Prevent regression
- ❌ Don't test implementation details
- ❌ Avoid over-mocking

**Why these tests matter:**

1. **Prevent security regressions** - Ensures auth checks aren't accidentally removed
2. **Document behavior** - Tests show expected authentication flow
3. **Fast feedback** - Catches issues before deployment

## Auth0 Middleware Configuration

Your middleware/proxy should focus on Auth0 routes, not route protection:

```typescript
// middleware.ts (Next.js 15) or proxy.ts (Next.js 16)
import type { NextRequest } from "next/server";
import { auth0 } from "@/lib/auth0-client";

export async function middleware(request: NextRequest) {
  // Only handles Auth0 routes and session management
  return await auth0.middleware(request);
}

export const config = {
  // Match all routes except static assets
  matcher: [
    "/((?!_next/static|_next/image|favicon.ico|sitemap.xml|robots.txt).*)",
  ],
};
```

**What this middleware does:**

- ✅ Handles `/auth/login`, `/auth/callback`, `/auth/logout`
- ✅ Manages session cookies
- ✅ Supports rolling sessions
- ❌ Does **NOT** redirect unauthenticated users
- ❌ Does **NOT** protect routes automatically

## Key Takeaways

### 🎯 Best Practices

1. **Server-side checks only** - Never rely on client-side auth alone
2. **Layout-level protection** - Protect route groups at the layout level
3. **Single source of truth** - One auth check for multiple routes
4. **Test critical paths** - Write simple tests for security logic
5. **Understand middleware** - Know what `auth0.middleware()` actually does

### ⚠️ Common Pitfalls

1. **Assuming middleware protects routes** - It doesn't
2. **Client-only checks** - Can be bypassed
3. **Over-engineering** - Complex middleware logic is unnecessary
4. **Missing tests** - Security logic should be tested
5. **Per-page duplication** - Use layouts instead

## References

- [Auth0 Next.js SDK Documentation](https://auth0.github.io/nextjs-auth0/)
- [Next.js App Router Authentication Guide](https://developer.auth0.com/resources/guides/web-app/nextjs/basic-authentication)
- [Next.js Server Components](https://nextjs.org/docs/app/building-your-application/rendering/server-components)

## Related Patterns

- [Error Handling in Server Components](../error-handling/README.md)
- [Testing Server Components](../testing/README.md)
- [TypeScript Best Practices](../typescript/README.md)
