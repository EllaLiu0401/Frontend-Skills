# Database Repository Patterns and Best Practices

## Overview

When building applications with database persistence, certain patterns emerge as critical for data integrity, security, and maintainability. This document covers essential patterns learned from production code reviews, focusing on soft deletes, timestamp handling, and defensive programming.

---

## Table of Contents

1. [Soft Delete Pattern](#soft-delete-pattern)
2. [Timestamp Handling](#timestamp-handling)
3. [Repository Consistency](#repository-consistency)
4. [Defense in Depth](#defense-in-depth)
5. [Code Review Checklist](#code-review-checklist)

---

## Soft Delete Pattern

### What is Soft Delete?

Soft delete is a pattern where records are never physically deleted from the database. Instead, a `deleted_at` (or `deletedAt`) timestamp column marks when a record was "deleted". This provides:

- **Data Recovery**: Accidentally deleted data can be restored
- **Audit Trail**: History of what was deleted and when
- **Referential Integrity**: Foreign key relationships remain intact
- **Compliance**: Some regulations require data retention

### The Critical Bug Pattern

**❌ WRONG - Missing soft delete filter:**

```typescript
// This can update soft-deleted users!
export async function updateUser(db: Database, userId: string, data: UpdateData) {
  await db
    .updateTable('users')
    .set(data)
    .where('id', '=', userId)
    .execute();
}
```

**✅ CORRECT - With soft delete filter:**

```typescript
export async function updateUser(db: Database, userId: string, data: UpdateData) {
  await db
    .updateTable('users')
    .set(data)
    .where('id', '=', userId)
    .where('deletedAt', 'is', null)  // Critical: exclude soft-deleted records
    .execute();
}
```

### Why This Matters

Without the `deletedAt IS NULL` filter:
- Soft-deleted records can be accidentally modified
- Business logic may act on "deleted" data
- Data integrity violations can occur
- Audit trails become unreliable

### Rule of Thumb

**Every query that reads or modifies data MUST filter out soft-deleted records by default.**

Exceptions only when explicitly fetching deleted records (e.g., admin recovery features).

### Pattern Examples

**SELECT queries:**
```typescript
// Find active user
export async function findById(db: Database, id: string) {
  return await db
    .selectFrom('users')
    .selectAll()
    .where('id', '=', id)
    .where('deletedAt', 'is', null)  // Always filter soft-deleted
    .executeTakeFirst();
}

// List active users
export async function listUsers(db: Database) {
  return await db
    .selectFrom('users')
    .selectAll()
    .where('deletedAt', 'is', null)  // Always filter soft-deleted
    .execute();
}
```

**UPDATE queries:**
```typescript
export async function updateEmail(db: Database, userId: string, email: string) {
  await db
    .updateTable('users')
    .set({ email })
    .where('id', '=', userId)
    .where('deletedAt', 'is', null)  // Prevent updating deleted users
    .execute();
}
```

**Soft delete implementation:**
```typescript
export async function softDelete(db: Database, userId: string) {
  await db
    .updateTable('users')
    .set({ deletedAt: new Date() })
    .where('id', '=', userId)
    .where('deletedAt', 'is', null)  // Prevent double-deletion
    .execute();
}
```

---

## Timestamp Handling

### Application-Side vs Database-Side Timestamps

When recording timestamps, you have two choices:

1. **Application-side**: `new Date()` in your code
2. **Database-side**: `NOW()`, `CURRENT_TIMESTAMP`, etc.

### The Problem with Application-Side Timestamps

```typescript
// ❌ Problematic: Application-side timestamp
await db
  .updateTable('users')
  .set({
    tosAcceptedAt: new Date(),  // Uses server's clock
    tosVersion: '2024-01-15'
  })
  .execute();
```

**Issues:**
- **Clock Skew**: In distributed systems, app servers may have different times
- **Time Zones**: Application server timezone vs database timezone mismatches
- **Precision**: JavaScript Date has millisecond precision, databases may differ
- **Race Conditions**: Timestamp generated before transaction commits

### Best Practice: Use Database Timestamps

```typescript
// ✅ Better: Database-side timestamp
import { sql } from 'kysely'; // or your query builder

await db
  .updateTable('users')
  .set({
    tosAcceptedAt: sql`NOW()`,  // Database generates the timestamp
    tosVersion: '2024-01-15'
  })
  .execute();
```

**Benefits:**
- **Consistency**: All timestamps use the same clock (database server)
- **Accuracy**: Timestamp reflects when the transaction commits
- **Time Zone Safety**: Database handles timezone conversions
- **Atomic**: Timestamp generation is part of the transaction

### When to Use Each

| Use Case | Timestamp Type | Reason |
|----------|---------------|---------|
| Record creation/modification | Database-side | Consistency, accuracy |
| Scheduling future events | Application-side | Need to control exact time |
| Audit logs | Database-side | Trustworthy audit trail |
| User input (birthdate, etc.) | Application-side | User-provided data |
| Soft deletes | Database-side | Must be consistent with DB state |

### Implementation Patterns

**With Kysely (TypeScript):**
```typescript
import { sql } from 'kysely';

// Insert with database timestamp
await db
  .insertInto('users')
  .values({
    id: userId,
    email: 'user@example.com',
    createdAt: sql`NOW()`
  })
  .execute();

// Update with database timestamp
await db
  .updateTable('users')
  .set({ lastLoginAt: sql`NOW()` })
  .where('id', '=', userId)
  .execute();
```

**With Knex.js:**
```typescript
await knex('users')
  .where('id', userId)
  .update({
    updated_at: knex.fn.now()
  });
```

**With Prisma:**
```typescript
// Prisma handles this automatically with @updatedAt
// For manual timestamps:
await prisma.$executeRaw`
  UPDATE users
  SET updated_at = NOW()
  WHERE id = ${userId}
`;
```

**In Migrations:**
```sql
-- Always use database functions for defaults
CREATE TABLE users (
  id UUID PRIMARY KEY,
  email TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at TIMESTAMPTZ NULL
);
```

---

## Repository Consistency

### The Importance of Consistent Patterns

When building repositories (data access layer), consistency is crucial:

1. **Predictability**: Developers know what to expect
2. **Safety**: Less likely to miss critical filters
3. **Maintainability**: Easier to update all repositories at once
4. **Code Review**: Inconsistencies become obvious red flags

### Common Repository Methods Pattern

```typescript
// Standard repository interface
export interface UserRepository {
  // Always filters soft-deleted
  findById(id: string): Promise<User | null>;

  // Always filters soft-deleted
  findByEmail(email: string): Promise<User | null>;

  // Always filters soft-deleted
  list(options: ListOptions): Promise<User[]>;

  // Creates new record
  create(data: CreateUserData): Promise<User>;

  // Updates existing, filters soft-deleted
  update(id: string, data: UpdateUserData): Promise<User>;

  // Soft deletes (sets deletedAt)
  delete(id: string): Promise<void>;

  // Optional: restore soft-deleted records
  restore(id: string): Promise<void>;

  // Optional: include deleted records
  findByIdIncludingDeleted(id: string): Promise<User | null>;
}
```

### Implementation Example

```typescript
// users-repository.ts
import { sql } from 'kysely';
import type { Database } from './database';

/**
 * Find user by ID (excludes soft-deleted)
 */
export async function findById(db: Database, userId: string) {
  return await db
    .selectFrom('users')
    .select(['id', 'email', 'name', 'createdAt', 'updatedAt'])
    .where('id', '=', userId)
    .where('deletedAt', 'is', null)
    .executeTakeFirst();
}

/**
 * Update user profile
 */
export async function updateProfile(
  db: Database,
  userId: string,
  data: { name?: string; email?: string }
) {
  await db
    .updateTable('users')
    .set({
      ...data,
      updatedAt: sql`NOW()`  // Database-side timestamp
    })
    .where('id', '=', userId)
    .where('deletedAt', 'is', null)  // Exclude soft-deleted
    .execute();
}

/**
 * Soft delete user
 */
export async function softDelete(db: Database, userId: string) {
  await db
    .updateTable('users')
    .set({
      deletedAt: sql`NOW()`  // Database-side timestamp
    })
    .where('id', '=', userId)
    .where('deletedAt', 'is', null)  // Prevent double-deletion
    .execute();
}

/**
 * List active users with pagination
 */
export async function list(
  db: Database,
  options: { limit: number; offset: number }
) {
  return await db
    .selectFrom('users')
    .select(['id', 'email', 'name', 'createdAt'])
    .where('deletedAt', 'is', null)  // Always filter
    .orderBy('createdAt', 'desc')
    .limit(options.limit)
    .offset(options.offset)
    .execute();
}
```

### Anti-Pattern: Inconsistent Filtering

```typescript
// ❌ BAD: Inconsistent repository
export async function findById(db: Database, id: string) {
  return await db
    .selectFrom('users')
    .where('id', '=', id)
    .where('deletedAt', 'is', null)  // Has filter ✓
    .executeTakeFirst();
}

export async function findByEmail(db: Database, email: string) {
  return await db
    .selectFrom('users')
    .where('email', '=', email)
    // Missing deletedAt filter! ✗
    .executeTakeFirst();
}

export async function updateProfile(db: Database, id: string, data: any) {
  await db
    .updateTable('users')
    .set(data)
    .where('id', '=', id)
    // Missing deletedAt filter! ✗
    .execute();
}
```

This inconsistency creates bugs and confusion.

---

## Defense in Depth

### Multiple Layers of Protection

Good database security uses multiple layers:

1. **Application Layer**: Queries filter soft-deleted records
2. **Database Layer**: Row-Level Security (RLS) policies
3. **Transaction Layer**: Proper isolation levels
4. **Schema Layer**: Constraints and triggers

### Example: Multi-Tenant Application

```typescript
// Layer 1: Application-level filtering
export async function getUserData(
  db: Database,
  orgId: string,
  userId: string
) {
  return await db
    .selectFrom('users')
    .selectAll()
    .where('orgId', '=', orgId)         // Tenant isolation
    .where('id', '=', userId)
    .where('deletedAt', 'is', null)     // Soft delete filter
    .executeTakeFirst();
}

// Layer 2: Database RLS Policy (PostgreSQL)
/*
CREATE POLICY tenant_isolation ON users
  USING (org_id = current_setting('app.current_org_id')::uuid);

CREATE POLICY soft_delete_filter ON users
  USING (deleted_at IS NULL);
*/

// Layer 3: Set context at transaction start
export async function withOrgContext<T>(
  db: Database,
  orgId: string,
  callback: (db: Database) => Promise<T>
): Promise<T> {
  return await db.transaction().execute(async (trx) => {
    // Set org context for RLS
    await trx.executeQuery(
      sql`SET LOCAL app.current_org_id = ${orgId}`.compile(trx)
    );

    return await callback(trx);
  });
}
```

### Why Multiple Layers?

- **Application bugs**: If you forget the filter in code, RLS catches it
- **Direct database access**: RLS protects against SQL injection or admin errors
- **Migration safety**: Constraints prevent invalid data states
- **Audit compliance**: Multiple layers satisfy security requirements

---

## Code Review Checklist

### When Reviewing Database Code

Use this checklist for any pull request that touches database queries:

#### ✅ Soft Delete Compliance

- [ ] All SELECT queries include `WHERE deleted_at IS NULL`
- [ ] All UPDATE queries include `WHERE deleted_at IS NULL`
- [ ] DELETE operations are actually soft deletes (UPDATE SET deleted_at)
- [ ] Hard deletes are only used when explicitly required (e.g., GDPR purge)

#### ✅ Timestamp Handling

- [ ] Timestamps use database functions (`NOW()`, `CURRENT_TIMESTAMP`)
- [ ] No `new Date()` for audit timestamps
- [ ] Timestamp columns use appropriate types (`TIMESTAMPTZ` in Postgres)
- [ ] Time zones are handled consistently

#### ✅ Repository Consistency

- [ ] All repository methods follow the same patterns
- [ ] Soft delete filtering is consistent across all methods
- [ ] Similar operations across different repositories use the same approach
- [ ] No copy-paste errors between similar functions

#### ✅ Security & Tenancy

- [ ] Multi-tenant queries include `org_id` or tenant filter
- [ ] No user input directly in SQL (use parameterized queries)
- [ ] Sensitive data is properly filtered
- [ ] Authorization checks before data access

#### ✅ Testing

- [ ] Tests verify soft-deleted records are excluded
- [ ] Tests verify cross-tenant isolation (if multi-tenant)
- [ ] Tests verify timestamp consistency
- [ ] Tests verify edge cases (double-delete, etc.)

### Example Test Cases

```typescript
describe('UserRepository', () => {
  it('findById should exclude soft-deleted users', async () => {
    // Create a user
    const user = await createUser({ email: 'test@example.com' });

    // Verify we can find it
    const found = await userRepository.findById(db, user.id);
    expect(found).toBeDefined();

    // Soft delete the user
    await userRepository.softDelete(db, user.id);

    // Should not find it anymore
    const notFound = await userRepository.findById(db, user.id);
    expect(notFound).toBeNull();
  });

  it('update should not affect soft-deleted users', async () => {
    const user = await createUser({ email: 'test@example.com' });
    await userRepository.softDelete(db, user.id);

    // Try to update soft-deleted user
    await userRepository.updateProfile(db, user.id, { name: 'New Name' });

    // Check the database directly (bypassing repository filters)
    const raw = await db
      .selectFrom('users')
      .selectAll()
      .where('id', '=', user.id)
      .executeTakeFirst();

    // Name should NOT have changed
    expect(raw?.name).not.toBe('New Name');
  });

  it('uses database timestamp for deletedAt', async () => {
    const beforeDelete = new Date();
    const user = await createUser({ email: 'test@example.com' });

    await userRepository.softDelete(db, user.id);

    const raw = await db
      .selectFrom('users')
      .selectAll()
      .where('id', '=', user.id)
      .executeTakeFirst();

    expect(raw?.deletedAt).toBeDefined();
    expect(raw?.deletedAt).toBeInstanceOf(Date);
    // Timestamp should be close to now (within reasonable time)
    const diff = Date.now() - raw!.deletedAt!.getTime();
    expect(diff).toBeLessThan(5000); // Within 5 seconds
  });
});
```

---

## Real-World Example: The Bug That Was Caught

### The Original Code

```typescript
export async function updateToSAcceptance(
  db: Db,
  userId: string,
  tosVersion: string
): Promise<void> {
  await db
    .updateTable('users')
    .set({
      tosAcceptedAt: new Date(),  // ❌ Application-side timestamp
      tosVersion,
    })
    .where('id', '=', userId)  // ❌ Missing soft delete filter
    .execute();
}
```

### The Problems

1. **Missing soft delete filter**: A soft-deleted user could have their ToS acceptance updated
2. **Application-side timestamp**: Clock skew issues in distributed systems

### The Fixed Code

```typescript
import { sql } from 'kysely';

export async function updateToSAcceptance(
  db: Db,
  userId: string,
  tosVersion: string
): Promise<void> {
  await db
    .updateTable('users')
    .set({
      tosAcceptedAt: sql`NOW()`,  // ✅ Database-side timestamp
      tosVersion,
    })
    .where('id', '=', userId)
    .where('deletedAt', 'is', null)  // ✅ Exclude soft-deleted users
    .execute();
}
```

### Why This Matters

In production:
- User deletes their account (soft delete)
- Some background job tries to update their ToS acceptance
- Without the fix: Updates happen on deleted account
- With the fix: Update is silently skipped (correct behavior)

---

## Summary

### Key Takeaways

1. **Always filter soft-deleted records** in SELECT and UPDATE queries
2. **Use database-side timestamps** for consistency and accuracy
3. **Be consistent across all repositories** to prevent bugs
4. **Implement defense in depth** with multiple security layers
5. **Test soft delete behavior** explicitly in your test suite

### The Golden Rule

> Every database query that reads or modifies data must explicitly handle soft deletes and use database-side timestamps for audit fields.

### Quick Reference

```typescript
// ✅ The correct pattern for repository methods
export async function repositoryMethod(db: Database, id: string) {
  return await db
    .selectFrom('table_name')
    .selectAll()
    .where('id', '=', id)
    .where('deletedAt', 'is', null)  // Always include this
    .execute();
}

// ✅ The correct pattern for timestamps
await db
  .updateTable('table_name')
  .set({
    updated_at: sql`NOW()`  // Use database function
  })
  .execute();
```

### Further Reading

- [Soft Delete Pattern on Martin Fowler's blog](https://martinfowler.com/eaaCatalog/softDelete.html)
- [PostgreSQL Row-Level Security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
- [Database Timestamp Best Practices](https://wiki.postgresql.org/wiki/Don%27t_Do_This#Don.27t_use_timestamp_.28without_time_zone.29)

---

**Document Version**: 1.0
**Last Updated**: 2024-02-09
**Applies To**: TypeScript, Node.js, SQL databases (Postgres, MySQL, etc.)
