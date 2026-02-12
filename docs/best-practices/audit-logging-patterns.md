# Audit Logging Patterns for Backend Systems

## Overview

Audit logging is critical for production systems to track **who did what, when, and why**. This document covers patterns for implementing robust audit trails using database triggers and session context.

## Why Audit Logging Matters

### Business Value
- **Compliance**: GDPR, CCPA, SOX require complete audit trails
- **Security**: Track suspicious activities and data breaches
- **Debugging**: Trace issues back to specific requests and users
- **Analytics**: Understand user behavior and system usage

### Real-World Scenario

**Without proper audit logging:**
```
Audit Log Entry:
- timestamp: 2024-02-10 10:30:00
- user_id: NULL ❌
- request_id: NULL ❌
- operation: UPDATE
- table: users
```

**Problem:** You know WHAT changed, but not WHO did it or HOW to trace the request.

**With proper audit logging:**
```
Audit Log Entry:
- timestamp: 2024-02-10 10:30:00
- user_id: "user_abc123" ✅
- request_id: "req_xyz789" ✅
- trace_id: "trace_def456" ✅
- operation: UPDATE
- table: users
```

**Benefit:** Full traceability - can identify the user, find the complete request log, and trace the entire request flow.

---

## Architecture Pattern

### Database Trigger Approach

Database triggers automatically capture all data changes without requiring application code to remember audit logging.

**Trigger Example (Conceptual):**
```sql
-- Trigger fires on INSERT/UPDATE/DELETE
CREATE TRIGGER audit_log_trigger
AFTER INSERT OR UPDATE OR DELETE ON sensitive_table
FOR EACH ROW EXECUTE FUNCTION log_audit_trail();
```

**Audit Trigger Function:**
```sql
CREATE FUNCTION log_audit_trail() RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO audit_log (
    user_id,      -- Who made the change?
    request_id,   -- Which request?
    trace_id,     -- Full distributed trace
    table_name,
    operation,
    old_data,
    new_data,
    timestamp
  ) VALUES (
    current_setting('app.user_id'),       -- Read from session variable
    current_setting('app.request_id'),    -- Read from session variable
    current_setting('app.trace_id'),      -- Read from session variable
    TG_TABLE_NAME,
    TG_OP,
    row_to_json(OLD),
    row_to_json(NEW),
    NOW()
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**Key Insight:** The trigger reads context from **PostgreSQL session variables**. If these aren't set, it logs NULL.

---

## Implementation Pattern

### ❌ Anti-Pattern: No Audit Context

```typescript
// BAD: Direct database operation without context
async function updateUserProfile(userId: string, data: any) {
  return await db
    .updateTable('users')
    .set(data)
    .where('id', '=', userId)
    .execute();
}
```

**Problem:**
- Audit trigger fires but captures NULL for `user_id`, `request_id`, `trace_id`
- No way to trace who made the change or from which request
- Compliance risk

### ✅ Best Practice: Set Audit Context Before Operations

```typescript
// GOOD: Wrap in transaction and set audit context
async function updateUserProfile(
  userId: string,
  data: any,
  context: { requestId: string, traceId: string, actorId: string }
) {
  return await db
    .transaction()
    .execute(async (trx) => {
      // Set session variables for audit trigger
      await setAuditContext(trx, {
        userId: context.actorId,
        requestId: context.requestId,
        traceId: context.traceId,
      });

      // Now perform the update
      return await trx
        .updateTable('users')
        .set(data)
        .where('id', '=', userId)
        .execute();
    });
}
```

**Benefits:**
- ✅ Audit log captures complete context
- ✅ Transaction ensures atomicity
- ✅ Session variables are transaction-scoped (isolated from other requests)

---

## Setting Database Context

### Context Setting Function

```typescript
interface AuditContext {
  userId?: string;
  requestId?: string;
  traceId?: string;
  orgId?: string;      // For multi-tenant systems
  reason?: string;     // Optional: why this change was made
}

async function setAuditContext(
  db: Transaction,
  context: AuditContext
): Promise<void> {
  // Set PostgreSQL session variables (transaction-scoped)
  await sql`
    SELECT
      set_config('app.user_id', ${context.userId ?? ''}, true),
      set_config('app.request_id', ${context.requestId ?? ''}, true),
      set_config('app.trace_id', ${context.traceId ?? ''}, true),
      set_config('app.org_id', ${context.orgId ?? ''}, true),
      set_config('app.reason', ${context.reason ?? ''}, true)
  `.execute(db);
}
```

**Key Points:**
- Use `set_config(name, value, is_local=true)` to make it transaction-scoped
- Empty strings convert to NULL in the trigger via `NULLIF()`
- Transaction isolation prevents context leaking between concurrent requests

---

## API Route Pattern

### Express.js / Fastify Pattern

```typescript
// Route handler
app.post('/api/users/:id', async (req, res) => {
  const userId = req.params.id;
  const updates = req.body;

  // Extract context from request
  const context = {
    actorId: req.user.id,           // From JWT/session
    requestId: req.id,              // Generated by framework
    traceId: req.headers['x-trace-id'] || generateTraceId(),
  };

  // Call service with context
  const result = await userService.updateProfile(userId, updates, context);

  return res.json(result);
});
```

### Service Layer Pattern

```typescript
// Service layer - business logic
class UserService {
  async updateProfile(
    userId: string,
    updates: ProfileUpdate,
    context: RequestContext
  ) {
    return await db.transaction().execute(async (trx) => {
      // STEP 1: Set audit context
      await setAuditContext(trx, {
        userId: context.actorId,
        requestId: context.requestId,
        traceId: context.traceId,
      });

      // STEP 2: Perform business logic
      const user = await this.userRepo.update(trx, userId, updates);

      // STEP 3: Trigger sends email, update cache, etc.
      await this.notifyUserOfChange(user);

      return user;
    });
  }
}
```

---

## Common Pitfalls

### 1. Forgetting to Use Transactions

```typescript
// ❌ BAD: No transaction = context not isolated
await setAuditContext(db, context);
await db.updateTable('users').set(data).execute();
```

**Problem:** In concurrent scenarios, another request might overwrite the session variables before your query runs.

```typescript
// ✅ GOOD: Transaction ensures isolation
await db.transaction().execute(async (trx) => {
  await setAuditContext(trx, context);
  await trx.updateTable('users').set(data).execute();
});
```

### 2. Inconsistent Patterns Across Codebase

```typescript
// Some routes set context:
await db.transaction().execute(async (trx) => {
  await setAuditContext(trx, context);
  await updateUser(trx, userId, data);
});

// Other routes don't:
await updateUser(db, userId, data);  // ❌ Missing audit context!
```

**Solution:** Enforce pattern via:
- Code review checklists
- Linting rules (detect `getDb()` usage in routes)
- Architecture documentation
- Team training

### 3. Not Handling System Operations

Some operations don't have a user context (e.g., cron jobs, system migrations).

```typescript
// ✅ Handle system operations gracefully
async function systemMigration() {
  await db.transaction().execute(async (trx) => {
    await setAuditContext(trx, {
      userId: 'SYSTEM',
      requestId: 'migration-2024-02-10',
      traceId: generateTraceId(),
      reason: 'Data migration for feature X',
    });

    // Perform migration
    await migrateData(trx);
  });
}
```

---

## Multi-Tenant Considerations

For multi-tenant systems, include `orgId` in audit context:

```typescript
interface AuditContext {
  orgId?: string;      // May be NULL for system tables
  userId: string;
  requestId: string;
  traceId: string;
}

// Tenant-scoped operation
await db.transaction().execute(async (trx) => {
  await setAuditContext(trx, {
    orgId: req.tenant.id,      // From tenant middleware
    userId: req.user.id,
    requestId: req.id,
    traceId: req.traceId,
  });

  await updateTenantData(trx, data);
});

// System table operation (no orgId)
await db.transaction().execute(async (trx) => {
  await setAuditContext(trx, {
    orgId: undefined,          // NULL in audit log
    userId: req.user.id,
    requestId: req.id,
    traceId: req.traceId,
  });

  await updateSystemTable(trx, data);
});
```

---

## Audit Log Schema

```sql
CREATE TABLE audit_log (
  id UUID PRIMARY KEY,
  occurred_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  -- Context
  org_id UUID,                    -- NULL for system operations
  user_id UUID,                   -- Who made the change
  request_id TEXT,                -- Request identifier
  trace_id TEXT,                  -- Distributed trace ID

  -- Operation details
  table_name TEXT NOT NULL,
  row_id TEXT NOT NULL,           -- ID of affected row
  operation TEXT NOT NULL,        -- INSERT, UPDATE, DELETE

  -- Data snapshots
  old_data JSONB,                 -- Before state (NULL for INSERT)
  new_data JSONB NOT NULL,        -- After state

  -- Optional
  reason TEXT                     -- Why the change was made
);

-- Indexes for querying
CREATE INDEX idx_audit_org_time ON audit_log (org_id, occurred_at);
CREATE INDEX idx_audit_user ON audit_log (user_id, occurred_at);
CREATE INDEX idx_audit_entity ON audit_log (table_name, row_id, occurred_at);
CREATE INDEX idx_audit_request ON audit_log (request_id);
CREATE INDEX idx_audit_trace ON audit_log (trace_id);
```

---

## Code Review Checklist

When reviewing code that modifies data, check:

- [ ] Is the operation wrapped in a transaction?
- [ ] Is `setAuditContext()` called before data modifications?
- [ ] Are all required context fields provided (`userId`, `requestId`, `traceId`)?
- [ ] Is the pattern consistent with other routes?
- [ ] Are there any direct `db.updateTable()` / `db.insertInto()` calls without context?
- [ ] For system operations, is there a clear reason/identifier?

---

## Testing Audit Logs

```typescript
describe('User update with audit logging', () => {
  it('should capture audit context when updating user', async () => {
    const userId = 'user_123';
    const context = {
      actorId: 'admin_456',
      requestId: 'req_789',
      traceId: 'trace_abc',
    };

    // Perform update
    await userService.updateProfile(userId, { name: 'John' }, context);

    // Verify audit log
    const auditEntry = await db
      .selectFrom('audit_log')
      .where('table_name', '=', 'users')
      .where('row_id', '=', userId)
      .orderBy('occurred_at', 'desc')
      .selectAll()
      .executeTakeFirst();

    expect(auditEntry).toBeDefined();
    expect(auditEntry.user_id).toBe('admin_456');
    expect(auditEntry.request_id).toBe('req_789');
    expect(auditEntry.trace_id).toBe('trace_abc');
    expect(auditEntry.operation).toBe('UPDATE');
    expect(auditEntry.new_data).toMatchObject({ name: 'John' });
  });

  it('should log NULL when audit context is missing', async () => {
    // This test ensures we detect the problem
    await db.updateTable('users')
      .set({ name: 'Jane' })
      .where('id', '=', 'user_123')
      .execute();

    const auditEntry = await db
      .selectFrom('audit_log')
      .where('table_name', '=', 'users')
      .where('row_id', '=', 'user_123')
      .orderBy('occurred_at', 'desc')
      .selectAll()
      .executeTakeFirst();

    // This should fail - alerts us to missing audit context
    expect(auditEntry.user_id).not.toBeNull();
  });
});
```

---

## Key Takeaways

1. **Always use transactions** when setting audit context
2. **Set context BEFORE data modifications** so triggers can capture it
3. **Make it consistent** - use the same pattern across the entire codebase
4. **Don't forget system operations** - provide meaningful context even for automated tasks
5. **Test audit logging** - verify context is captured correctly
6. **Review for audit context** - make it part of your code review checklist

---

## References & Further Reading

- PostgreSQL Session Variables: https://www.postgresql.org/docs/current/functions-admin.html#FUNCTIONS-ADMIN-SET
- Database Triggers: https://www.postgresql.org/docs/current/trigger-definition.html
- Distributed Tracing: https://opentelemetry.io/docs/concepts/signals/traces/
- Audit Logging Best Practices: https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html

---

**Last Updated:** February 12, 2026
**Related Topics:** Database Transactions, Multi-Tenant Architecture, Compliance, Security
