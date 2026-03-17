# Multi-Brand Proxy: Security and Brand Resolution Patterns

## Problem

In a single-deployment multi-brand architecture (one app serving multiple domains like `app.brand-a.com` and `app.brand-b.com`), the proxy/middleware must resolve which brand to display. Naive implementations can introduce security issues where users see the wrong brand — or worse, get permanently stuck on the wrong brand.

---

## Lesson 1: Dev-Only Overrides Must Be Guarded

### Problem

During local development, you often need a way to switch brands without separate domains (e.g., `?brand=other-brand` query param). A common mistake is forgetting to restrict this to local dev only.

```typescript
// ❌ Bad: Override runs on ALL hosts
function resolveBrand(request: Request): Brand {
  const host = request.headers.get('host') ?? '';
  if (host.includes('brand-b')) return 'brand-b';

  // Comment says "localhost only" but no actual check!
  const override = request.nextUrl.searchParams.get('brand');
  if (override === 'brand-a' || override === 'brand-b') return override;

  return 'brand-a';
}
```

**Impact**: On production, `app.brand-a.com?brand=brand-b` sets a persistent wrong-brand cookie. Users see the wrong branding on a legitimate domain.

### Solution

```typescript
// ✅ Good: Guard dev overrides with a hostname check
function resolveBrand(request: Request): Brand {
  const host = request.headers.get('host') ?? '';
  if (host.includes('brand-b')) return 'brand-b';

  const isLocalDev = host.includes('localhost') || host.includes('127.0.0.1');
  if (isLocalDev) {
    const override = request.nextUrl.searchParams.get('brand');
    if (override === 'brand-a' || override === 'brand-b') return override;
  }

  return getDefaultBrand(host);
}
```

**Rule**: If a feature is meant for local dev only, the code must enforce it — comments alone are not sufficient.

---

## Lesson 2: Cookie Fallbacks Can Create Self-Perpetuating Bugs

### Problem

A common pattern is to persist brand in a cookie so it survives navigation. But if the cookie is read back on production (where hostname should be the authority), a stale wrong-brand cookie creates a **perpetual loop**.

```typescript
// ❌ Bad: Cookie fallback runs on production too
function resolveBrand(request: Request): Brand {
  const host = request.headers.get('host') ?? '';
  if (host.includes('brand-b')) return 'brand-b';

  // Dev-only override (properly guarded)
  if (isLocalDev(host)) { /* ... */ }

  // This runs on production! A stale cookie overrides hostname.
  const cookieBrand = request.cookies.get('brand')?.value;
  if (cookieBrand === 'brand-b') return 'brand-b';

  return getDefaultBrand(host);
}
```

**The vicious cycle**:
1. User has a stale `brand=brand-b` cookie on `app.brand-a.com`
2. `resolveBrand()` reads cookie → returns `brand-b`
3. Downstream code sets `x-brand: brand-b` header for server rendering
4. Response writes `brand=brand-b` cookie with fresh 1-year expiry
5. Next request → same thing → **user is permanently stuck**

The only escape is manually clearing cookies — terrible UX.

### Solution

The cookie is only needed on localhost (where hostname can't distinguish brands). On production, hostname is the single source of truth.

```typescript
// ✅ Good: Cookie only consulted on localhost
function resolveBrand(request: Request): Brand {
  const host = request.headers.get('host') ?? '';
  if (host.includes('brand-b')) return 'brand-b';

  const isLocalDev = host.includes('localhost') || host.includes('127.0.0.1');
  if (isLocalDev) {
    const override = request.nextUrl.searchParams.get('brand');
    if (override === 'brand-a' || override === 'brand-b') return override;

    const cookieBrand = request.cookies.get('brand')?.value;
    if (cookieBrand === 'brand-b') return 'brand-b';
  }

  return getDefaultBrand(host);
}
```

**Rule**: If a value is derived from an authoritative source (hostname), don't let a less authoritative source (cookie) override it on the same environment. Cookies should only fill gaps where the authority is absent (e.g., localhost).

---

## Lesson 3: i18n Strings Need Brand Interpolation Too

### Problem

When making a page brand-aware, it's easy to update visual elements (logos, colors) but forget text content buried in i18n files.

```json
// ❌ Bad: Hardcoded brand name in translation
{
  "acceptTerms": {
    "title": "Welcome to Brand A"
  }
}
```

BookingBoost users see: Brand B logo + "Welcome to Brand A" title — mixed branding on a public-facing auth flow.

### Solution

Use interpolation in i18n keys, same as any other brand-aware string.

```json
// ✅ Good: Parameterised brand name
{
  "acceptTerms": {
    "title": "Welcome to {brandName}"
  }
}
```

```tsx
// Pass brand name from config
{t('title', { brandName: brandConfig.name })}
```

**Rule**: When adding multi-brand support, audit ALL user-facing strings — not just visual elements. Search i18n files for hardcoded brand names.

---

## Lesson 4: Proxy Matcher vs. Auth Exemption Are Different Concerns

### Problem

Confusing "which routes run through the proxy" with "which routes require authentication" leads to incorrect test expectations.

```typescript
// Proxy matcher: which routes the middleware processes
export const config = {
  matcher: ['/((?!healthz|monitoring|_next/static).*)'],
};

// Auth exemption: which routes skip org/auth checks
const PATHS_WITHOUT_ORG = ['/login', '/accept-terms', '/onboarding'];
```

`/login` should run through the proxy (so the brand cookie gets set) but should NOT require authentication. These are orthogonal concerns.

```typescript
// ❌ Bad test: confuses matcher with auth
it('excludes login from proxy', () => {
  expect(matcherRegex.test('/login')).toBe(false); // Wrong!
});

// ✅ Good test: login goes through proxy for brand detection
it('matches login so proxy can set brand cookie', () => {
  expect(matcherRegex.test('/login')).toBe(true);
});
```

**Rule**: Route matching (which middleware runs) and auth policy (who can access) are separate layers. A route can be matched by middleware without requiring authentication.

---

## General Principles

| Principle | Description |
|---|---|
| **Hostname is authority on production** | On production domains, the hostname determines the brand. No override mechanism should be available. |
| **Dev conveniences must be guarded** | Query param overrides, cookie fallbacks — anything for dev flexibility must check `isLocalDev`. |
| **Comments are not guards** | "// localhost only" means nothing without an `if` statement. Code must enforce invariants. |
| **Audit text, not just visuals** | Multi-brand means i18n strings, page titles, meta tags — not just logos and colors. |
| **Cookie writes amplify reads** | If you read a cookie to decide brand, and then write it back, a wrong value becomes permanent. Think about the full read-write cycle. |
| **Separate matching from authorization** | Middleware routing and access control are orthogonal. Don't conflate them in code or tests. |
