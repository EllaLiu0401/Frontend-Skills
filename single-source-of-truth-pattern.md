# Single Source of Truth Pattern for Version Management

## Problem

When building applications that require users to accept versioned agreements (terms, policies, etc.), common issues arise:

1. **Frontend hardcodes version** → Requires code changes and redeployment when version updates
2. **Backend only validates format** → Security vulnerability (users can bypass with arbitrary values)
3. **No version comparison** → Users who accepted old versions aren't prompted to re-accept

## Solution: Single Source of Truth Pattern

### Architecture Overview

```
Backend Config (Source of Truth)
    ↓
API Response (Exposes current version)
    ↓
Frontend Reads (No hardcoded values)
    ↓
Backend Validates (Enforces current version)
```

---

## Implementation

### 1. Backend: Configuration

**Bad** ❌:
```typescript
// Hardcoded in multiple places
const CURRENT_VERSION = '2024-01-01';
```

**Good** ✅:
```typescript
// config.ts - Single source of truth
export const config = {
  currentVersion: process.env.CURRENT_VERSION || '2024-01-01',
  // ... other config
};
```

**Benefits**:
- Change via environment variable
- No code changes needed
- One place to update

---

### 2. Backend: Service Layer Validation

**Bad** ❌:
```typescript
// Only validates format
if (!/^\d{4}-\d{2}-\d{2}$/.test(version)) {
  throw new ValidationError('Invalid format');
}
// Accepts any date!
```

**Good** ✅:
```typescript
// Validates both format AND matches current version
if (!/^\d{4}-\d{2}-\d{2}$/.test(version)) {
  throw new ValidationError('Invalid format');
}

if (version !== config.currentVersion) {
  throw new ValidationError(
    `Version mismatch: expected '${config.currentVersion}', got '${version}'`
  );
}
```

**Benefits**:
- Prevents bypassing with curl/API clients
- Enforces current version acceptance only
- Clear error messages

---

### 3. API Response: Expose Current Version

**Bad** ❌:
```typescript
// GET /api/user
return {
  id: user.id,
  acceptedVersion: user.acceptedVersion,
  // Frontend has no way to know current version!
};
```

**Good** ✅:
```typescript
// GET /api/user
return {
  id: user.id,
  acceptedVersion: user.acceptedVersion,
  currentVersion: config.currentVersion, // ✅ Expose current
};
```

**Benefits**:
- Frontend can compare versions
- Dynamic updates without code changes
- Single source of truth maintained

---

### 4. Frontend: Fetch and Use API Version

**Bad** ❌:
```typescript
// Hardcoded in frontend
const VERSION = '2024-01-01';

function handleSubmit() {
  api.acceptTerms({ version: VERSION }); // Stale when updated!
}
```

**Good** ✅:
```typescript
// Fetch from API
const { data: user } = useQuery(['user'], fetchUser);

function handleSubmit() {
  if (!user?.currentVersion) return; // Guard
  api.acceptTerms({ version: user.currentVersion }); // Always current
}

// Disable button until data loads
<button disabled={!user || isPending}>
  Accept
</button>
```

**Benefits**:
- No hardcoded versions
- Always uses current version
- Prevents submission before data loads

---

### 5. Frontend: Version Comparison Logic

**Bad** ❌:
```typescript
// Only checks if user accepted at some point
const hasAccepted = user.acceptedAt !== null;

// Problem: User who accepted v1 can access app when v2 is current!
```

**Good** ✅:
```typescript
// Checks if user accepted CURRENT version
const hasAccepted =
  user.acceptedAt !== null &&
  user.acceptedVersion === user.currentVersion;

// User must re-accept when version updates
```

**Benefits**:
- Enforces re-acceptance when version changes
- Prevents bypassing with old acceptances
- Automatic invalidation on updates

---

## Complete Flow Example

### Scenario: Version Update from v1 to v2

```
┌─────────────────────────────────────────────────────────┐
│ 1. Admin updates environment variable                   │
│    CURRENT_VERSION=2024-06-01 (was 2024-01-01)         │
│    → Restart service                                    │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│ 2. User visits app                                      │
│    GET /api/user                                        │
│    → { acceptedVersion: '2024-01-01',                  │
│        currentVersion: '2024-06-01' }                   │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│ 3. Frontend comparison                                  │
│    '2024-01-01' === '2024-06-01' → false               │
│    hasAccepted = false                                  │
│    → Redirect to acceptance page                        │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│ 4. User clicks accept                                   │
│    POST /api/accept-terms                               │
│    { version: '2024-06-01' } ← from API, not hardcoded │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│ 5. Backend validation                                   │
│    version === config.currentVersion                    │
│    '2024-06-01' === '2024-06-01' → ✅ Accept           │
└─────────────────────────────────────────────────────────┘
```

---

## Security Benefits

### Before (Vulnerable)
```bash
# Attacker can bypass with old version
curl -X POST /api/accept-terms \
  -d '{"version":"2020-01-01"}'

→ ✅ Accepted (SECURITY ISSUE!)
```

### After (Secure)
```bash
# Backend rejects non-current versions
curl -X POST /api/accept-terms \
  -d '{"version":"2020-01-01"}'

→ ❌ 400 ValidationError: "Version mismatch"
```

---

## Anti-Patterns to Avoid

### ❌ Anti-Pattern 1: Fallback to Hardcoded Value
```typescript
const version = apiVersion ?? 'HARDCODED'; // BAD!
```
**Problem**: Defeats single source of truth

**Solution**: Disable submission until API data loads
```typescript
if (!apiVersion) return; // Guard
api.submit({ version: apiVersion });
```

---

### ❌ Anti-Pattern 2: Only Format Validation
```typescript
if (!/^\d{4}-\d{2}-\d{2}$/.test(version)) {
  throw new Error('Invalid format');
}
// Accepts any date!
```

**Solution**: Add version match validation
```typescript
if (version !== config.currentVersion) {
  throw new Error('Must accept current version');
}
```

---

### ❌ Anti-Pattern 3: Version in Multiple Places
```typescript
// config.ts
const VERSION = '2024-01-01';

// frontend.ts
const VERSION = '2024-01-01'; // Duplicate!

// service.ts
const VERSION = '2024-01-01'; // Duplicate!
```

**Solution**: Single config, all others read from it

---

## Testing Strategy

### Unit Tests
```typescript
describe('version validation', () => {
  it('rejects old version', () => {
    config.currentVersion = '2024-06-01';

    expect(() => validateVersion('2024-01-01'))
      .toThrow('Version mismatch');
  });

  it('accepts current version', () => {
    config.currentVersion = '2024-06-01';

    expect(() => validateVersion('2024-06-01'))
      .not.toThrow();
  });
});
```

### Integration Tests
```typescript
describe('acceptance flow', () => {
  it('returns current version in user API', async () => {
    const response = await api.get('/user');

    expect(response.data).toHaveProperty('currentVersion');
    expect(response.data.currentVersion).toMatch(/^\d{4}-\d{2}-\d{2}$/);
  });

  it('invalidates old acceptances', async () => {
    // User accepted v1
    await db.updateUser({ acceptedVersion: '2024-01-01' });

    // Config updated to v2
    config.currentVersion = '2024-06-01';

    // User should need to re-accept
    const user = await api.get('/user');
    expect(hasAcceptedCurrent(user)).toBe(false);
  });
});
```

---

## Checklist for Implementation

- [ ] Backend config as single source of truth
- [ ] Environment variable override support
- [ ] Service layer validates version match
- [ ] API response includes current version
- [ ] Frontend fetches version from API
- [ ] No hardcoded versions in frontend
- [ ] Frontend compares accepted vs current version
- [ ] Loading state prevents premature submission
- [ ] Security: Backend rejects non-current versions
- [ ] Tests cover version mismatch scenarios

---

## Key Takeaways

1. **Single Source of Truth**: Backend config is authoritative
2. **No Hardcoding**: Frontend reads from API dynamically
3. **Layered Validation**:
   - Format validation (schema layer)
   - Version match validation (service layer)
   - Frontend comparison (UX layer)
4. **Security in Depth**: Multiple layers prevent bypass
5. **Config-Driven**: Updates via environment variables only
6. **Automatic Invalidation**: Old acceptances require re-acceptance

---

## Related Patterns

- **Feature Flags**: Similar single-source pattern for feature toggles
- **API Versioning**: Version headers from backend config
- **Content Versioning**: CMS content with version tracking
- **Schema Versioning**: Database migration version management

---

## References

- [12-Factor App: Config](https://12factor.net/config)
- [OWASP: Input Validation](https://owasp.org/www-project-proactive-controls/v3/en/c5-validate-inputs)
- [Defense in Depth](https://en.wikipedia.org/wiki/Defense_in_depth_(computing))
