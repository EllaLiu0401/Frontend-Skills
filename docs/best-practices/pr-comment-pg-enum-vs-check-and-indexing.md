# PR Comment — PostgreSQL Enums vs CHECK Constraints & When NOT to Index

> **TL;DR**: Use PostgreSQL enum types instead of `TEXT + CHECK` for fixed-value columns — they're type-safe, codegen-friendly, and match standard patterns. Don't add indexes on low-cardinality columns or small tables — it's premature optimization with zero benefit.

## Context

A database migration added a new column with a fixed set of allowed values. The initial implementation used a `TEXT` column with a `CHECK` constraint and added a standalone B-tree index. The reviewer flagged both decisions during PR review.

## Issues & Fixes

### 1) Prefer PG Enum over TEXT + CHECK for Fixed-Value Columns

**Symptom**: The column was defined as `TEXT` with a separate `CHECK` constraint to restrict values. This works, but doesn't follow the team's established pattern and produces weaker types in codegen output.

```sql
-- ❌ Before: TEXT column + CHECK constraint
ALTER TABLE my_table ADD COLUMN status TEXT NOT NULL DEFAULT 'active';
ALTER TABLE my_table ADD CONSTRAINT chk_status
  CHECK (status IN ('active', 'inactive'));
```

```sql
-- ✅ After: PostgreSQL enum type
CREATE TYPE my_status AS ENUM ('active', 'inactive');
ALTER TABLE my_table ADD COLUMN status my_status NOT NULL DEFAULT 'active';
```

**Why PG enums are better:**

| Aspect | TEXT + CHECK | PG Enum |
|--------|-------------|---------|
| Type safety | Constraint is separate — easy to forget or desync | Type itself defines valid values |
| Codegen output | ORM sees `TEXT` → generates `string` | ORM sees enum → generates `'active' \| 'inactive'` |
| Consistency | Ad-hoc pattern, each column may differ | Reusable named type across columns |
| Discoverability | Must inspect constraints to find valid values | `\dT+` in psql lists all enums |

**Lesson**: Always check how the existing codebase handles similar patterns before introducing a new one. If every other fixed-value column uses PG enums, follow that convention — even if the planning doc says otherwise (docs can be wrong; code is the reality).

### 2) Don't Index Low-Cardinality Columns on Small Tables

**Symptom**: A B-tree index was added on a column with only 2 possible values, on a table expected to have at most a few hundred rows.

```sql
-- ❌ Before: unnecessary index
CREATE INDEX idx_my_table_status ON my_table (status);

-- ✅ After: no index
-- (removed entirely)
```

**Why this index is useless:**

1. **Small table = fast full scan** — PostgreSQL can scan a few hundred rows in microseconds. An index adds no measurable benefit.

2. **Low cardinality kills index effectiveness** — With only 2 values, each value matches ~50% of rows. The query planner will likely ignore the index and do a sequential scan anyway, because traversing a B-tree to read half the table is slower than just scanning it directly.

3. **No query pattern needs it** — The column is read alongside the row via primary key lookup (`WHERE id = ?`), not used as a standalone filter (`WHERE status = ?`).

**Decision tree for indexing:**

```
Should I add an index?
├── Is this column used in WHERE / JOIN / ORDER BY frequently?
│   ├── No  → Don't index
│   └── Yes → Does the table have many rows (10k+)?
│       ├── No  → Probably don't need it
│       └── Yes → Is the cardinality high (many distinct values)?
│           ├── No  (boolean, 2-3 enum values) → Usually not helpful
│           └── Yes (email, UUID, timestamp)    → Index it ✅
```

**Lesson**: Indexing is not free — it slows down writes, consumes disk, and adds maintenance overhead. Only index when there's a real query pattern that benefits from it. Adding an index later is a one-line migration; removing premature optimization is harder because people start depending on it.

### 3) Down Migration Order Matters for Enums

When rolling back a PG enum column, you **must** drop the column before dropping the type. PostgreSQL won't let you drop a type that's still referenced.

```sql
-- ✅ Correct order
ALTER TABLE my_table DROP COLUMN status;
DROP TYPE IF EXISTS my_status;

-- ❌ Wrong order — fails with "cannot drop type: other objects depend on it"
DROP TYPE IF EXISTS my_status;
ALTER TABLE my_table DROP COLUMN status;
```

### 4) Adding Values to Existing PG Enums

```sql
ALTER TYPE my_status ADD VALUE IF NOT EXISTS 'suspended';
```

**Caveats:**
- `ADD VALUE` **cannot** run inside a transaction in PostgreSQL — keep it in a standalone migration
- PostgreSQL does **not** support removing individual enum values — the down migration is typically a no-op with a comment explaining why
- If you need to remove a value, you must create a new enum, migrate the column, and drop the old type

## How I Replied to the PR Comments

### Reply template structure

```
[Warm acknowledgment] — [action taken]! ✅

**Before:** [brief description]
(code block)

**After:** [brief description]
(code block)

[1-2 sentences: why the change is correct]
```

### Actual replies

**To "Switch this to a PG enum":**

> Great call — updated! ✅
>
> **Before:** `TEXT` column + `CHECK` constraint
>
> **After:** PG enum type
>
> This aligns with the existing codebase pattern and gives us a proper TypeScript union type via codegen instead of a plain `string`.

**To "Probably don't need an index at this point?":**

> Makes sense — removed! ✅
>
> **Before:** Standalone index on the column
>
> **After:** No index.
>
> Two reasons: (1) we don't query by this column directly, (2) cardinality is too low for an index to help. Easy to add later if needed.

### Tone tips for PR replies

- **Start positive**: "Great call", "Good catch", "Makes sense", "Thanks for flagging"
- **Show understanding**: Don't just say "fixed" — explain *why* the reviewer is correct
- **Be concise**: Before/After + one reason is enough. Reviewers are busy
- **Use ✅ emoji**: Signals the comment is resolved at a glance

## Key Takeaways

1. **Follow existing patterns** — check how the codebase handles the same problem before inventing a new approach
2. **PG enum > TEXT + CHECK** — for stable, small value sets (2-10 values)
3. **Don't index prematurely** — small table + low cardinality + no query pattern = no index
4. **YAGNI** — it's easier to add an index later than to remove one that people depend on
5. **Down migrations matter** — always test `up → down` reversibility; order of operations counts
