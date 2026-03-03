# PR Review — PostgreSQL Enums vs CHECK Constraints & When NOT to Index

> **TL;DR**: Use PostgreSQL enum types instead of `TEXT + CHECK` for fixed-value columns — they're type-safe, codegen-friendly, and match standard patterns. Don't add indexes on low-cardinality columns or small tables — it's premature optimization with zero benefit.

## Context

A migration was adding a new column with a fixed set of allowed values (e.g. `'optionA' | 'optionB'`). The initial implementation used a `TEXT` column with a `CHECK` constraint and added a standalone index. Code review flagged both decisions.

## Key Learnings

### 1) Prefer PG Enum over TEXT + CHECK for Fixed-Value Columns

**What was done:**

```sql
-- ❌ TEXT column + CHECK constraint
ALTER TABLE my_table ADD COLUMN status TEXT NOT NULL DEFAULT 'active';
ALTER TABLE my_table ADD CONSTRAINT chk_status CHECK (status IN ('active', 'inactive'));
```

**What the reviewer suggested:**

```sql
-- ✅ PostgreSQL enum type
CREATE TYPE my_status AS ENUM ('active', 'inactive');
ALTER TABLE my_table ADD COLUMN status my_status NOT NULL DEFAULT 'active';
```

**Why PG enums are better:**

| Aspect | TEXT + CHECK | PG Enum |
|--------|-------------|---------|
| **Type safety** | Constraint is separate from the column — easy to forget or get out of sync | The type itself defines valid values — impossible to insert invalid data |
| **Codegen** | ORMs/codegen tools see `TEXT` → generate `string` in TypeScript | ORMs/codegen tools see the enum → generate a union type like `'active' \| 'inactive'` |
| **Consistency** | Ad-hoc pattern — each column might use CHECK differently | Reusable named type — the enum can be referenced by multiple columns |
| **Discoverability** | Have to inspect constraints to know valid values | `\dT+` in psql shows all enum types and their values |

**When to use CHECK instead:** If the valid values are truly dynamic or the list is very long and frequently changing, CHECK or a foreign key to a lookup table may be more appropriate. Enums are best for stable, small value sets.

### 2) Don't Index Low-Cardinality Columns on Small Tables

**What was done:**

```sql
-- ❌ Index on a column with only 2 possible values, on a table with few hundred rows
CREATE INDEX idx_my_table_status ON my_table (status);
```

**Why the reviewer said "probably don't need this":**

1. **Small table = fast full scan**: If the table has hundreds or even a few thousand rows, PostgreSQL can scan the entire table in microseconds. An index adds no measurable performance gain.

2. **Low cardinality kills index effectiveness**: With only 2 possible values, each value matches ~50% of rows. PostgreSQL's query planner will likely **ignore the index** and do a sequential scan anyway, because reading half the table through a B-tree is actually slower than just scanning it directly.

3. **YAGNI (You Aren't Gonna Need It)**: Adding an index "just in case" is premature optimization. Indexes have costs — they slow down writes, consume disk space, and add complexity. If a performance problem appears later, adding an index is a one-line migration.

**Rule of thumb for indexing:**

```
Should I add an index?
├── Is this column used in WHERE/JOIN/ORDER BY frequently? 
│   ├── No  → Don't index
│   └── Yes → Does the table have many rows (10k+)?
│       ├── No  → Probably don't need it
│       └── Yes → Is the cardinality high (many distinct values)?
│           ├── No  (e.g. boolean, 2-3 enum values) → Usually not helpful
│           └── Yes (e.g. email, UUID, timestamp) → Index it ✅
```

### 3) Down Migrations: Drop Order Matters for Enums

When writing down/rollback migrations with PG enums, you **must** drop the column before dropping the enum type. PostgreSQL won't let you drop a type that's still referenced by a column.

```sql
-- ✅ Correct order
ALTER TABLE my_table DROP COLUMN status;
DROP TYPE IF EXISTS my_status;

-- ❌ Wrong order — will fail with "cannot drop type because other objects depend on it"
DROP TYPE IF EXISTS my_status;
ALTER TABLE my_table DROP COLUMN status;
```

### 4) Adding Values to Existing PG Enums

PostgreSQL supports adding values to an existing enum, but with caveats:

```sql
-- Add a new value
ALTER TYPE my_status ADD VALUE IF NOT EXISTS 'suspended';
```

**Important:** `ADD VALUE` cannot run inside a transaction in PostgreSQL. If your migration framework wraps each migration in a transaction, you may need to handle this specially (e.g. separate migration file, or disabling the transaction wrapper).

**Also:** PostgreSQL does **not** support removing individual enum values. If you need to remove a value, you'd have to create a new enum type, migrate the column, and drop the old type. This is why the down migration for `ADD VALUE` is typically a no-op.

## Summary

| Decision | Guideline |
|----------|-----------|
| Fixed-value column | Use PG enum, not TEXT + CHECK |
| Index on small table | Skip it — add later if needed |
| Index on low-cardinality column | Almost never useful |
| Down migration with enum | Drop column first, then drop type |
| Adding enum values | Use `ADD VALUE IF NOT EXISTS`, separate migration, no transaction |
