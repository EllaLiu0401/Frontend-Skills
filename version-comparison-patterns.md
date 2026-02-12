# Version Comparison Patterns

## Overview

When building systems that track versioned resources (documents, agreements, APIs, etc.), proper version comparison is critical for security and user experience.

---

## Anti-Pattern: Existence Check Only

### Problem

```typescript
// ❌ Bad: Only checks if user accepted at some point
function hasAccepted(user) {
  return user.acceptedAt !== null;
}

// Scenario:
// - User accepted v1.0 on Jan 1
// - System updated to v2.0 on Feb 1
// - hasAccepted() returns true (WRONG!)
// - User can access system without accepting v2.0
```

**Issues**:
- Doesn't verify current version
- Security vulnerability
- Compliance risk
- Users bypass re-acceptance

---

## Pattern: Version Match Validation

### Solution

```typescript
// ✅ Good: Checks both existence AND version match
function hasAcceptedCurrent(user) {
  return (
    user.acceptedAt !== null &&
    user.acceptedVersion === user.currentVersion
  );
}

// Scenario:
// - User accepted v1.0 on Jan 1
// - System updated to v2.0 on Feb 1
// - user.acceptedVersion = '1.0'
// - user.currentVersion = '2.0'
// - hasAcceptedCurrent() returns false (CORRECT!)
// - User must re-accept v2.0
```

**Benefits**:
- Enforces current version
- Automatic invalidation on updates
- Secure and compliant
- Clear acceptance state

---

## Implementation Examples

### Frontend Hook

```typescript
interface User {
  id: string;
  acceptedAt: Date | null;
  acceptedVersion: string | null;
  currentVersion: string;
}

export function useVersionCheck() {
  const { data: user, isLoading } = useFetch<User>('/api/user');

  const hasAcceptedCurrent = useMemo(() => {
    if (!user) return false;

    // Two conditions must be met:
    // 1. User has accepted at some point
    // 2. Their accepted version matches current version
    return (
      user.acceptedAt !== null &&
      user.acceptedVersion === user.currentVersion
    );
  }, [user]);

  return {
    hasAcceptedCurrent,
    isLoading,
    user,
  };
}
```

**Usage**:
```typescript
function ProtectedRoute({ children }) {
  const { hasAcceptedCurrent, isLoading } = useVersionCheck();

  if (isLoading) return <Loading />;

  if (!hasAcceptedCurrent) {
    return <Redirect to="/accept-terms" />;
  }

  return children;
}
```

---

### Backend Validation

```typescript
// Service layer validation
export async function validateUserAccess(userId: string): Promise<boolean> {
  const user = await userRepository.findById(userId);

  if (!user) return false;

  // Check both conditions
  return (
    user.tosAcceptedAt !== null &&
    user.tosVersion === config.currentTosVersion
  );
}

// Route guard
router.use(async (req, res, next) => {
  const hasAccess = await validateUserAccess(req.userId);

  if (!hasAccess) {
    return res.status(403).json({
      error: 'Must accept current terms of service',
      redirect: '/accept-terms',
    });
  }

  next();
});
```

---

## Semantic Versioning Comparison

### Simple Date-Based Versions

```typescript
// Format: YYYY-MM-DD
function compareVersions(v1: string, v2: string): number {
  return v1.localeCompare(v2); // Works for date strings
}

const isCurrentVersion = (acceptedVersion: string, currentVersion: string) => {
  return compareVersions(acceptedVersion, currentVersion) === 0;
};

// Example:
isCurrentVersion('2024-01-01', '2024-01-01'); // true
isCurrentVersion('2024-01-01', '2024-06-01'); // false
```

---

### Semantic Versioning (SemVer)

```typescript
// Format: MAJOR.MINOR.PATCH (e.g., 2.1.3)
import semver from 'semver';

function needsReAcceptance(
  acceptedVersion: string,
  currentVersion: string
): boolean {
  // Parse versions
  const accepted = semver.parse(acceptedVersion);
  const current = semver.parse(currentVersion);

  if (!accepted || !current) return true;

  // Only require re-acceptance for major version changes
  return accepted.major < current.major;
}

// Examples:
needsReAcceptance('1.5.0', '1.6.0'); // false (minor bump)
needsReAcceptance('1.5.0', '2.0.0'); // true (major bump)
needsReAcceptance('2.1.0', '2.1.1'); // false (patch bump)
```

---

### Custom Version Logic

```typescript
interface Version {
  major: number;
  minor: number;
  patch: number;
}

function parseVersion(version: string): Version {
  const [major, minor, patch] = version.split('.').map(Number);
  return { major, minor, patch };
}

function requiresReAcceptance(
  accepted: string,
  current: string
): boolean {
  const acceptedVer = parseVersion(accepted);
  const currentVer = parseVersion(current);

  // Business rule: Re-acceptance needed for major or minor changes
  return (
    acceptedVer.major < currentVer.major ||
    acceptedVer.minor < currentVer.minor
  );
}

// Examples:
requiresReAcceptance('1.0.0', '1.0.1'); // false (patch only)
requiresReAcceptance('1.0.0', '1.1.0'); // true (minor change)
requiresReAcceptance('1.5.0', '2.0.0'); // true (major change)
```

---

## Edge Cases to Handle

### 1. Null/Undefined Versions

```typescript
function hasAcceptedCurrent(user: User): boolean {
  // Handle missing data
  if (!user.acceptedAt || !user.acceptedVersion) {
    return false; // Never accepted
  }

  if (!user.currentVersion) {
    throw new Error('Current version not configured'); // System error
  }

  return user.acceptedVersion === user.currentVersion;
}
```

---

### 2. Future Versions

```typescript
function hasAcceptedCurrent(user: User): boolean {
  const accepted = parseVersion(user.acceptedVersion);
  const current = parseVersion(user.currentVersion);

  // User might have accepted a future version in testing
  // Treat as current if equal or newer
  return accepted.major >= current.major &&
         accepted.minor >= current.minor;
}
```

---

### 3. Migration from Old System

```typescript
function hasAcceptedCurrent(user: User): boolean {
  // Legacy users have no version (accepted before versioning)
  if (user.acceptedAt && !user.acceptedVersion) {
    // Treat as not accepted - require re-acceptance
    return false;
  }

  return (
    user.acceptedAt !== null &&
    user.acceptedVersion === user.currentVersion
  );
}
```

---

## Testing Version Comparison

```typescript
describe('version comparison', () => {
  it('accepts exact version match', () => {
    const user = {
      acceptedAt: new Date(),
      acceptedVersion: '2024-01-01',
      currentVersion: '2024-01-01',
    };

    expect(hasAcceptedCurrent(user)).toBe(true);
  });

  it('rejects old version', () => {
    const user = {
      acceptedAt: new Date('2024-01-01'),
      acceptedVersion: '2024-01-01',
      currentVersion: '2024-06-01', // Updated
    };

    expect(hasAcceptedCurrent(user)).toBe(false);
  });

  it('rejects no acceptance', () => {
    const user = {
      acceptedAt: null,
      acceptedVersion: null,
      currentVersion: '2024-01-01',
    };

    expect(hasAcceptedCurrent(user)).toBe(false);
  });

  it('handles missing current version', () => {
    const user = {
      acceptedAt: new Date(),
      acceptedVersion: '2024-01-01',
      currentVersion: undefined,
    };

    expect(() => hasAcceptedCurrent(user)).toThrow();
  });
});
```

---

## UI Patterns

### Display Version Status

```typescript
function VersionStatus({ user }: { user: User }) {
  if (!user.acceptedAt) {
    return (
      <Alert type="warning">
        You haven't accepted the terms yet.
        <Link to="/accept-terms">Accept Now</Link>
      </Alert>
    );
  }

  const isOutdated = user.acceptedVersion !== user.currentVersion;

  if (isOutdated) {
    return (
      <Alert type="warning">
        Terms updated. You accepted v{user.acceptedVersion}, but current is v{user.currentVersion}.
        <Link to="/accept-terms">Review Changes</Link>
      </Alert>
    );
  }

  return (
    <Alert type="success">
      Terms accepted (v{user.acceptedVersion})
    </Alert>
  );
}
```

---

### Changelog Display

```typescript
function TermsPage() {
  const { user } = useVersionCheck();

  // Show changelog if user has old version
  const showChangelog = user?.acceptedVersion !== user?.currentVersion;

  return (
    <div>
      <h1>Terms of Service (v{user?.currentVersion})</h1>

      {showChangelog && user?.acceptedVersion && (
        <Alert type="info">
          You previously accepted v{user.acceptedVersion}.
          <details>
            <summary>What's Changed?</summary>
            <ChangelogSince version={user.acceptedVersion} />
          </details>
        </Alert>
      )}

      <TermsContent version={user?.currentVersion} />

      <AcceptButton />
    </div>
  );
}
```

---

## Database Schema

### User Table
```sql
CREATE TABLE users (
  id UUID PRIMARY KEY,
  email VARCHAR(255) NOT NULL,

  -- Version tracking
  tos_accepted_at TIMESTAMP,
  tos_version VARCHAR(50), -- e.g., '2024-01-01' or '2.1.0'

  -- Audit
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Index for version queries
CREATE INDEX idx_users_tos_version ON users(tos_version);
```

### Acceptance History Table (Optional)
```sql
CREATE TABLE tos_acceptances (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  version VARCHAR(50) NOT NULL,
  accepted_at TIMESTAMP DEFAULT NOW(),
  ip_address INET,
  user_agent TEXT
);

-- Query acceptance history
SELECT * FROM tos_acceptances
WHERE user_id = 'user-123'
ORDER BY accepted_at DESC;
```

---

## Compliance & Audit

### Log Acceptance Events

```typescript
async function recordAcceptance(params: {
  userId: string;
  version: string;
  ipAddress: string;
  userAgent: string;
}) {
  // Update user record
  await db.updateTable('users')
    .set({
      tosAcceptedAt: new Date(),
      tosVersion: params.version,
    })
    .where('id', '=', params.userId)
    .execute();

  // Log in audit table
  await db.insertInto('tos_acceptances')
    .values({
      userId: params.userId,
      version: params.version,
      acceptedAt: new Date(),
      ipAddress: params.ipAddress,
      userAgent: params.userAgent,
    })
    .execute();
}
```

---

### Generate Compliance Report

```typescript
async function getComplianceReport(version: string) {
  const stats = await db
    .selectFrom('users')
    .select([
      db.fn.count('id').as('totalUsers'),
      db.fn.count('tosAcceptedAt').as('acceptedCount'),
      db.fn
        .count('id')
        .filterWhere('tosVersion', '=', version)
        .as('currentVersionCount'),
    ])
    .executeTakeFirst();

  return {
    totalUsers: stats.totalUsers,
    acceptanceRate: (stats.acceptedCount / stats.totalUsers) * 100,
    currentVersionRate: (stats.currentVersionCount / stats.totalUsers) * 100,
  };
}

// Example output:
// {
//   totalUsers: 10000,
//   acceptanceRate: 95.5,
//   currentVersionRate: 87.2
// }
```

---

## Best Practices

### ✅ DO

- Check both existence AND version match
- Store version with acceptance timestamp
- Validate version on backend
- Show clear version status in UI
- Log all acceptance events for audit
- Handle version updates gracefully
- Test edge cases (null, future versions)

### ❌ DON'T

- Check only if user accepted (without version)
- Allow access with outdated versions
- Hardcode version in frontend only
- Skip backend validation
- Lose acceptance history
- Force re-acceptance on patch updates (if using SemVer)

---

## Checklist

- [ ] Version stored with acceptance
- [ ] Comparison checks both existence AND match
- [ ] Backend validates current version
- [ ] Frontend checks version before allowing access
- [ ] UI shows version status
- [ ] Changelog displayed for updates
- [ ] Acceptance events logged for audit
- [ ] Tests cover version mismatch scenarios
- [ ] Edge cases handled (null, migration)
- [ ] Compliance reports available

---

## Common Mistakes

### Mistake 1: Incomplete Check
```typescript
// ❌ Only checks acceptedAt
const hasAccepted = !!user.acceptedAt;

// ✅ Check both
const hasAccepted = !!user.acceptedAt && user.version === currentVersion;
```

### Mistake 2: No Backend Validation
```typescript
// ❌ Frontend only
if (user.version !== currentVersion) redirect('/terms');

// ✅ Backend enforces
router.use((req, res, next) => {
  if (user.version !== config.currentVersion) {
    return res.status(403).json({ error: 'Must accept current terms' });
  }
  next();
});
```

### Mistake 3: Lost History
```typescript
// ❌ Overwrites old version
UPDATE users SET version = 'v2' WHERE id = 'user-123';

// ✅ Keeps history
INSERT INTO tos_acceptances (user_id, version) VALUES ('user-123', 'v2');
UPDATE users SET version = 'v2' WHERE id = 'user-123';
```

---

## Related Patterns

- **Feature Flags**: Similar version-based rollout
- **API Versioning**: Version negotiation in headers
- **Database Migrations**: Version-based schema changes
- **Content Versioning**: CMS content with versions

---

## Further Reading

- [Semantic Versioning](https://semver.org/)
- [GDPR Consent Requirements](https://gdpr-info.eu/)
- [Version Comparison Algorithms](https://en.wikipedia.org/wiki/Software_versioning)
