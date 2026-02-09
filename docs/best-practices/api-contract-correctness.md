# API Contract Correctness and Data Integrity

## Overview

When building APIs and data layers, it's critical that operations return accurate results that reflect what actually happened. This document covers common pitfalls and best practices for ensuring API contracts are truthful and reliable.

## The Problem: Silent Failures

### What is a Silent Failure?

A silent failure occurs when an operation reports success but didn't actually perform the intended action. This is particularly dangerous because:

1. **Breaks trust**: Callers believe the operation succeeded
2. **Masks bugs**: Issues go undetected in production
3. **Data inconsistency**: System state doesn't match what users expect
4. **Hard to debug**: By the time the issue is discovered, the context is lost

### Common Example

```typescript
// BAD: Returns success even when no rows were updated
async function updateUserPreferences(userId: string, preferences: object) {
  await db
    .updateTable('users')
    .set({ preferences })
    .where('id', '=', userId)
    .execute();  // Returns empty array if user doesn't exist

  return { success: true };  // Always returns success!
}
```

**Problem**: If the `userId` doesn't exist, the UPDATE affects zero rows, but the function still returns `{ success: true }`.

## Best Practices

### 1. Verify Operation Results

Always check that database operations actually affected rows:

```typescript
// GOOD: Throws error when user doesn't exist
async function updateUserPreferences(userId: string, preferences: object) {
  await db
    .updateTable('users')
    .set({ preferences })
    .where('id', '=', userId)
    .returning('id')  // Request the updated row
    .executeTakeFirstOrThrow();  // Throws NoResultError if no match

  return { success: true };  // Only returns if update succeeded
}
```

### 2. Use Appropriate Query Methods

Different query methods have different behaviors:

| Method | Use Case | Throws on Empty? |
|--------|----------|------------------|
| `.execute()` | Multiple results expected | No |
| `.executeTakeFirst()` | Single result, may be null | No |
| `.executeTakeFirstOrThrow()` | Single result required | Yes |
| `.returning('id')` | Need updated row data | Depends on method |

```typescript
// Reading data
const user = await db
  .selectFrom('users')
  .where('id', '=', userId)
  .executeTakeFirstOrThrow();  // Throws if user not found

// Updating data
const updated = await db
  .updateTable('users')
  .set({ lastLogin: new Date() })
  .where('id', '=', userId)
  .returning('id')
  .executeTakeFirstOrThrow();  // Throws if no rows updated

// Listing data
const users = await db
  .selectFrom('users')
  .limit(10)
  .execute();  // Returns empty array if no results (valid)
```

### 3. Match Patterns Across Codebase

Consistency matters. If your `findById` throws on missing records, your `update` should too:

```typescript
// Repository layer should be consistent
export const userRepository = {
  // Throws if not found
  async findById(id: string) {
    return db
      .selectFrom('users')
      .where('id', '=', id)
      .executeTakeFirstOrThrow();
  },

  // Should also throw if not found (consistent!)
  async update(id: string, data: object) {
    return db
      .updateTable('users')
      .set(data)
      .where('id', '=', id)
      .returning('id')
      .executeTakeFirstOrThrow();
  }
};
```

### 4. Document Behavior

Make error conditions explicit in documentation:

```typescript
/**
 * Update user Terms of Service acceptance
 *
 * @throws NoResultError if user does not exist or has been deleted
 */
export async function updateToSAcceptance(
  db: Database,
  userId: string,
  tosVersion: string
): Promise<void> {
  await db
    .updateTable('users')
    .set({ tosAcceptedAt: new Date(), tosVersion })
    .where('id', '=', userId)
    .where('deletedAt', 'is', null)
    .returning('id')
    .executeTakeFirstOrThrow();
}
```

## API Layer Implications

### Service Layer

The service layer should handle repository errors appropriately:

```typescript
// Service translates database errors to domain errors
export async function acceptTerms(userId: string, version: string) {
  try {
    await userRepository.updateToSAcceptance(userId, version);
    return { success: true };
  } catch (error) {
    if (error instanceof NoResultError) {
      throw new NotFoundError('User not found');
    }
    throw error;
  }
}
```

### API Response

The API framework should translate domain errors to HTTP responses:

```typescript
// Fastify error handler
fastify.setErrorHandler((error, request, reply) => {
  if (error instanceof NotFoundError) {
    return reply.status(404).send({
      error: 'Not Found',
      message: error.message
    });
  }

  // Log unexpected errors
  logger.error(error);
  return reply.status(500).send({
    error: 'Internal Server Error',
    message: 'An unexpected error occurred'
  });
});
```

## Testing Considerations

### Test Both Success and Failure Cases

```typescript
describe('updateUserPreferences', () => {
  it('updates preferences when user exists', async () => {
    const userId = await createTestUser();

    const result = await updateUserPreferences(userId, { theme: 'dark' });

    expect(result.success).toBe(true);

    const user = await getUserById(userId);
    expect(user.preferences.theme).toBe('dark');
  });

  it('throws NotFoundError when user does not exist', async () => {
    const fakeUserId = 'non-existent-id';

    await expect(
      updateUserPreferences(fakeUserId, { theme: 'dark' })
    ).rejects.toThrow(NotFoundError);
  });
});
```

### Integration Tests Should Verify Database State

```typescript
it('does not create partial data on failure', async () => {
  const userId = 'non-existent';

  await expect(
    updateUserWithRelatedData(userId, {
      profile: {...},
      settings: {...}
    })
  ).rejects.toThrow();

  // Verify rollback - no orphaned data
  const orphanedProfile = await db
    .selectFrom('profiles')
    .where('userId', '=', userId)
    .executeTakeFirst();

  expect(orphanedProfile).toBeUndefined();
});
```

## Common Pitfalls

### 1. Assuming Success Without Verification

```typescript
// BAD
async function deleteUser(userId: string) {
  await db.deleteFrom('users').where('id', '=', userId).execute();
  return { deleted: true };  // Was anything actually deleted?
}

// GOOD
async function deleteUser(userId: string) {
  const result = await db
    .deleteFrom('users')
    .where('id', '=', userId)
    .returning('id')
    .executeTakeFirstOrThrow();  // Throws if user didn't exist

  return { deleted: true, userId: result.id };
}
```

### 2. Swallowing Errors

```typescript
// BAD
async function updateUser(userId: string, data: object) {
  try {
    await userRepository.update(userId, data);
  } catch (error) {
    console.log('Update failed');  // Silent failure!
  }
  return { success: true };
}

// GOOD
async function updateUser(userId: string, data: object) {
  await userRepository.update(userId, data);  // Let errors propagate
  return { success: true };
}
```

### 3. Inconsistent Error Handling

```typescript
// BAD: Inconsistent patterns
const user = await db.selectFrom('users')
  .where('id', '=', userId)
  .executeTakeFirstOrThrow();  // Throws on missing

await db.updateTable('users')
  .set({ lastLogin: new Date() })
  .where('id', '=', userId)
  .execute();  // Silently does nothing if missing

// GOOD: Consistent patterns
const user = await db.selectFrom('users')
  .where('id', '=', userId)
  .executeTakeFirstOrThrow();

await db.updateTable('users')
  .set({ lastLogin: new Date() })
  .where('id', '=', userId)
  .returning('id')
  .executeTakeFirstOrThrow();  // Also throws on missing
```

## Key Takeaways

1. **Operations should report accurate results** - Don't return success when nothing happened
2. **Use `.executeTakeFirstOrThrow()` for operations that require a match** - Updates, deletes, and single-record reads
3. **Be consistent** - If reads throw on missing records, writes should too
4. **Document error conditions** - Make it clear what exceptions callers should handle
5. **Test failure cases** - Don't just test the happy path
6. **Let errors propagate** - Don't swallow exceptions unless you have a good reason

## Related Patterns

- **Repository Pattern**: Encapsulates data access logic and provides consistent error handling
- **Error Boundaries**: Centralized error handling at application boundaries (HTTP handlers, etc.)
- **Idempotency**: Design operations to be safely retryable
- **Optimistic Locking**: Prevent concurrent modification issues with version checks

## Further Reading

- Database transaction best practices
- API error response standardization
- Domain-driven design error handling
- ACID properties and their implications
