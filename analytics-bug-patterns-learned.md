# Frontend Bug Patterns & Solutions Learned

**Date**: 2026-02-09
**Context**: Analytics Dashboard Implementation

---

## 1. Date Range Boundary Handling

### Problem: Off-by-One in Date Ranges

**Scenario**: API uses exclusive upper bound (`WHERE date >= from AND date < to`), but frontend sets `to = today at midnight`, excluding today's data.

**Root Cause**: Mismatched understanding of inclusive vs exclusive boundaries.

```typescript
// ❌ WRONG: Excludes today
const to = new Date();
to.setHours(0, 0, 0, 0);  // to = today 00:00:00

// SQL: WHERE date >= from AND date < to
// Result: Only gets past days, excludes today
```

**Solution**: Set upper bound to next day's midnight for inclusive range.

```typescript
// ✅ CORRECT: Includes today
const to = new Date();
to.setHours(0, 0, 0, 0);
to.setDate(to.getDate() + 1);  // to = tomorrow 00:00:00

// SQL: WHERE date >= from AND date < to
// Result: Includes today (from today 00:00 to tomorrow 00:00)
```

### Key Lessons

- **Know your boundary semantics**: `<` (exclusive) vs `<=` (inclusive)
- **Coordinate with backend**: Frontend and API must agree on date range interpretation
- **Test with "today" edge case**: Most bugs appear when querying current day
- **Prefer exclusive upper bounds**: Cleaner for datetime ranges (avoids `.999` milliseconds)

### General Pattern

```typescript
function getDateRange(days: number): { from: Date; to: Date } {
  // For "last N days INCLUDING today"
  const from = new Date();
  from.setHours(0, 0, 0, 0);
  from.setDate(from.getDate() - days + 1);  // N days ago, including today

  const to = new Date();
  to.setHours(0, 0, 0, 0);
  to.setDate(to.getDate() + 1);  // Tomorrow midnight (exclusive upper bound)

  return { from, to };
}
```

---

## 2. Sentinel Value Handling in Type Systems

### Problem: String Sentinels Break Type Guards

**Scenario**: Using `'Unknown'` string as a sentinel value, but truthy checks fail to detect it.

```typescript
// Data layer uses 'Unknown' for undefined dates
const aggregated = buckets.map(b => ({
  date: b.date ?? 'Unknown',  // Sentinel value
  count: b.count
}));

// ❌ WRONG: Truthy check passes for 'Unknown' string
const label = item.date
  ? new Date(item.date).toLocaleDateString()  // new Date('Unknown') = Invalid Date
  : fallback('unknown');
```

**Solution**: Explicitly check for sentinel values.

```typescript
// ✅ CORRECT: Check for sentinel before type coercion
const label = item.date && item.date !== 'Unknown'
  ? new Date(item.date).toLocaleDateString()
  : fallback('unknown');
```

### Alternative: Use TypeScript Discriminated Unions

```typescript
// Better type-safe approach
type AggregatedData =
  | { type: 'dated'; date: string; count: number }
  | { type: 'undated'; count: number };

// Type guard automatically works
if (item.type === 'dated') {
  return new Date(item.date).toLocaleDateString();
} else {
  return fallback('unknown');
}
```

### Key Lessons

- **Sentinel values are fragile**: String sentinels bypass type system
- **Explicit checks required**: `value !== 'SENTINEL'` before processing
- **Consider union types**: TypeScript discriminated unions are type-safe
- **Document sentinel values**: Comment why they exist and how to handle them

---

## 3. Race Condition Prevention in Async UI

### Problem: Out-of-Order Response Handling

**Scenario**: User rapidly changes filters, older requests complete after newer ones, overwriting UI with stale data.

```
User clicks "7 days"  → Request A (ID: 1)
User clicks "30 days" → Request B (ID: 2)
Request B completes   → UI shows 30 days ✅
Request A completes   → UI shows 7 days ❌ WRONG
```

**Naive Approach (Doesn't Work)**:
```typescript
// ❌ WRONG: No request tracking
const handleFilterChange = (filter) => {
  const result = await fetchData(filter);
  setState(result);  // Any response overwrites state
};
```

### Solution: Request ID Pattern

```typescript
const requestIdRef = useRef(0);

const handleFilterChange = (filter) => {
  // Increment and capture request ID
  requestIdRef.current += 1;
  const currentRequestId = requestIdRef.current;

  startTransition(async () => {
    const result = await fetchData(filter);

    // Ignore stale responses
    if (currentRequestId !== requestIdRef.current) {
      return;  // A newer request has started, discard this
    }

    setState(result);  // Safe to update
  });
};
```

### How It Works

1. Each request gets incrementing ID (1, 2, 3...)
2. Each request captures its ID in closure
3. On response, check if ID still matches current
4. If mismatch, a newer request exists → discard

### Alternative: AbortController

```typescript
const abortControllerRef = useRef<AbortController | null>(null);

const handleFilterChange = (filter) => {
  // Cancel previous request
  abortControllerRef.current?.abort();

  const controller = new AbortController();
  abortControllerRef.current = controller;

  try {
    const result = await fetchData(filter, { signal: controller.signal });
    setState(result);
  } catch (err) {
    if (err.name === 'AbortError') return;  // Cancelled, ignore
    throw err;
  }
};
```

### Comparison

| Approach | Pros | Cons |
|----------|------|------|
| Request ID | Simple, no network cancellation needed | Request still runs |
| AbortController | Actually cancels network request | More complex, requires signal support |

### Key Lessons

- **Race conditions are real**: Any async operation with user input is vulnerable
- **useRef for tracking**: Mutable ref persists across renders
- **Closure captures**: Each async callback captures its own request ID
- **Debouncing ≠ Race protection**: Debounce delays, doesn't prevent races

---

## 4. Duration Formatting UX Considerations

### Problem: Misleading Zero Values

**Scenario**: 45-second duration displays as "0:00", looks like zero duration.

```typescript
// ❌ WRONG: All sub-minute durations show as 0:00
function formatDuration(seconds: number): string {
  const hours = Math.floor(seconds / 3600);
  const minutes = Math.floor((seconds % 3600) / 60);
  return `${hours}:${minutes.toString().padStart(2, '0')}`;
}

formatDuration(45);  // "0:00" - confusing!
```

**Solution**: Show seconds for short durations.

```typescript
// ✅ CORRECT: Show seconds for clarity
function formatDuration(seconds: number): string {
  if (seconds < 60) {
    return `0:${Math.floor(seconds).toString().padStart(2, '0')}`;
  }

  const hours = Math.floor(seconds / 3600);
  const minutes = Math.floor((seconds % 3600) / 60);
  return `${hours}:${minutes.toString().padStart(2, '0')}`;
}

formatDuration(45);   // "0:45" ✅
formatDuration(90);   // "1:30" ✅
```

### Design Considerations

**Option 1**: Seconds for short durations only
- Pro: Consistent format for longer durations
- Pro: Minimal UI changes
- Con: Format changes at 60-second boundary

**Option 2**: Always show seconds
- Pro: Always accurate
- Con: Verbose ("1:30:00" vs "1:30")
- Con: Wider UI space needed

**Option 3**: Different format for short durations
- "45s" vs "1:30" (units change)
- Pro: Very clear
- Con: Inconsistent format

### Key Lessons

- **Zero is ambiguous**: "0:00" could mean "no data" or "very short"
- **Context matters**: Analytics dashboard needs precision
- **Consistency vs Clarity**: Sometimes breaking format consistency improves UX
- **Test edge cases**: 0s, 59s, 60s, 61s all behave differently

---

## 5. API Response Type Architecture Patterns

### Context: Server Components vs Server Actions

**Two valid patterns emerged**:

#### Pattern A: Direct API Client (Server Components)
```typescript
// Server Component (SSR)
const result: ApiResponse<Data> = await apiClient.getData();

if (result.error) {
  // Handle ApiResponse shape: { data?: T, error?: ApiError }
}
```

#### Pattern B: Server Action Wrapper (Client → Server)
```typescript
// Server Action
export async function getData(): Promise<ActionResult<Data>> {
  const result = await apiClient.getData();
  return toActionResult(result);  // Convert ApiResponse → ActionResult
}

// Client Component
const result = await getData();
if (!result.ok) {
  // Handle ActionResult shape: { ok: boolean, data?: T, error?: ActionError }
}
```

### Architectural Trade-offs

| Aspect | Direct API (Pattern A) | Server Action (Pattern B) |
|--------|------------------------|---------------------------|
| Performance | ⚡ Faster (no wrapper) | Slightly slower |
| Consistency | Different response types | Uniform ActionResult |
| Use Case | SSR initial data | Client-side refetch |
| Complexity | Simple | Extra abstraction layer |

### Key Lessons

- **Both patterns can coexist**: Not necessarily an error
- **Consistency vs Performance**: Sometimes worth trading consistency for speed
- **Document decisions**: Comment why pattern differs
- **Discriminators matter**: `ok` boolean vs `error` presence affects type guards

**When to use each**:
- **Pattern A**: Server Component initial data, performance critical
- **Pattern B**: Client-side actions, mutations, uniform error handling

---

## 6. General Debugging Principles Applied

### 1. Understand Boundary Semantics
- Always clarify inclusive vs exclusive bounds
- Test with edge cases (today, now, zero)

### 2. Trace Data Flow
- Follow data from API → transform → UI
- Check each layer's assumptions

### 3. Type Safety Isn't Magic
- Runtime sentinel values bypass TypeScript
- Explicit checks still needed

### 4. Test Async Timing
- Race conditions need deliberate testing
- Rapid user actions expose timing bugs

### 5. UX Over Technical Purity
- "0:00" is technically correct but confusing
- User clarity > format consistency

---

## Quick Reference: Common Pitfalls

| Pattern | Wrong ❌ | Right ✅ |
|---------|----------|----------|
| Date ranges | `to = today 00:00` with `< to` | `to = tomorrow 00:00` |
| Sentinel values | `if (value)` | `if (value && value !== 'SENTINEL')` |
| Async races | No tracking | Request ID or AbortController |
| Zero display | "0:00" for 45s | "0:45" or "45s" |
| Type guards | Trust discriminator alone | Validate sentinel values |

---

## Testing Checklist

When implementing similar features:

- [ ] Test with current day/time data
- [ ] Test rapid filter changes (race conditions)
- [ ] Test edge values (0, 1, boundary-1, boundary, boundary+1)
- [ ] Verify inclusive/exclusive semantics with backend
- [ ] Check sentinel value handling in UI layer
- [ ] Validate UX clarity for edge case displays

---

## 7. Data Aggregation with Non-Unique Display Names

### Problem: Silent Data Corruption from Premature Name Mapping

**Date Added**: 2026-02-11

**Scenario**: Aggregating analytics data by display names instead of stable IDs causes distinct entities with duplicate names to be incorrectly merged.

```typescript
// Database schema (simplified)
interface Entity {
  id: string;           // Unique, stable identifier (UUIDv7)
  name: string;         // Display name, NOT unique
  // Schema only enforces uniqueness on provider_id, not name
}

// Example data
[
  { id: 'uuid-123', name: 'Sales Bot', calls: 10, cost: 5 },
  { id: 'uuid-456', name: 'Sales Bot', calls: 20, cost: 10 },  // Different entity, same name!
  { id: 'uuid-789', name: 'Support Bot', calls: 15, cost: 7 }
]
```

### ❌ Wrong Approach: Aggregate by Display Name

```typescript
function aggregateData(buckets: Bucket[]): ChartData[] {
  // STEP 1: Map to display names FIRST
  const rawData = buckets.map(bucket => ({
    name: mapIdToName(bucket.entityId),  // Convert ID → Name
    count: bucket.count,
    cost: bucket.cost
  }));

  // STEP 2: Aggregate by name
  const aggregatedMap = rawData.reduce((acc, item) => {
    const key = item.name;  // Using name as aggregation key
    if (acc[key]) {
      acc[key].count += item.count;  // Merging different entities!
      acc[key].cost += item.cost;
    } else {
      acc[key] = { ...item };
    }
    return acc;
  }, {});

  return Object.values(aggregatedMap);
}

// Result (WRONG):
// [
//   { name: 'Sales Bot', count: 30, cost: 15 },     ← Two entities merged!
//   { name: 'Support Bot', count: 15, cost: 7 }
// ]
```

**Why This Is Dangerous**:
- **Silent corruption**: No errors thrown, data looks plausible
- **User confusion**: Analytics show fewer entities than actually exist
- **Incorrect totals**: Merged metrics lose per-entity accuracy
- **Hard to detect**: Only discovered when comparing raw data vs aggregated charts

---

### ✅ Correct Approach: Two-Stage Aggregation

```typescript
function aggregateData(
  buckets: Bucket[],
  groupByDimension: string,
  nameMap?: Map<string, string>
): ChartData[] {
  // STEP 1: Map to aggregation key (use stable ID for entity dimensions)
  const rawData = buckets.map(bucket => {
    const aggregationKey =
      groupByDimension === 'entityId'
        ? bucket.entityId ?? '__no_entity__'  // Aggregate by ID
        : formatDisplayValue(bucket.value);    // Other dimensions can use display value

    return {
      aggregationKey,       // For grouping
      originalValue: bucket.value,  // For display mapping later
      count: bucket.count,
      cost: bucket.cost
    };
  });

  // STEP 2: Aggregate by stable key
  const aggregatedMap = rawData.reduce((acc, item) => {
    const key = item.aggregationKey;  // ID, not name
    if (acc[key]) {
      acc[key].count += item.count;
      acc[key].cost += item.cost;
    } else {
      acc[key] = { ...item };
    }
    return acc;
  }, {});

  // STEP 3: Map to display names AFTER aggregation
  return Object.values(aggregatedMap)
    .map(item => ({
      name: mapToDisplayName(item.originalValue, groupByDimension, nameMap),
      count: item.count,
      cost: item.cost
    }))
    .sort((a, b) => b.count - a.count);
}

// Result (CORRECT):
// [
//   { name: 'Sales Bot', count: 20, cost: 10 },    ← uuid-456
//   { name: 'Support Bot', count: 15, cost: 7 },   ← uuid-789
//   { name: 'Sales Bot', count: 10, cost: 5 }      ← uuid-123
// ]
// Charts can handle duplicate labels (each gets distinct color/tooltip)
```

---

### Key Architectural Decisions

#### When to Use Each Strategy

| Dimension Type | Aggregation Key | Example |
|---------------|-----------------|---------|
| **Entity references** | Stable ID | User IDs, Assistant IDs, Product IDs |
| **Enums/statuses** | Display value | Status ('completed', 'failed'), Direction ('inbound', 'outbound') |
| **Dates** | Date value | Date strings ('2024-01-15') |

**Rule of thumb**: If the database field can have duplicates, aggregate by ID.

#### Why Charts Accept Duplicate Labels

Modern charting libraries handle duplicate labels correctly:
- Each data point gets unique color based on array index
- Tooltips show individual values
- Legend can show duplicates (or be hidden)
- User sees: "Two different entities have same name"

**Better than**:
- Merging data (incorrect)
- Appending IDs to labels: "Sales Bot (uuid-123)" (ugly)
- Forcing unique names (UX constraint)

---

### Testing Strategy

#### Test Case 1: Distinct Entities with Duplicate Names

```typescript
it('keeps distinct entities separate even with duplicate names', () => {
  const nameMap = new Map([
    ['entity-123', 'Sales Bot'],
    ['entity-456', 'Sales Bot'],  // Same name, different entity
    ['entity-789', 'Support Bot']
  ]);

  const buckets = [
    { entityId: 'entity-123', count: 10, cost: 5 },
    { entityId: 'entity-456', count: 20, cost: 10 },
    { entityId: 'entity-789', count: 15, cost: 7 }
  ];

  const result = aggregateData(buckets, 'entityId', nameMap);

  // Should have 3 separate buckets (not 2)
  expect(result).toHaveLength(3);

  // Both "Sales Bot" entries should exist
  const salesBots = result.filter(item => item.name === 'Sales Bot');
  expect(salesBots).toHaveLength(2);
  expect(salesBots[0].count).toBe(20);  // entity-456
  expect(salesBots[1].count).toBe(10);  // entity-123
});
```

#### Test Case 2: Other Dimensions Still Merge

```typescript
it('aggregates by display value for non-entity dimensions', () => {
  const buckets = [
    { status: 'completed', count: 10 },
    { status: 'completed', count: 20 }
  ];

  const result = aggregateData(buckets, 'status');

  // Should merge into 1 bucket
  expect(result).toHaveLength(1);
  expect(result[0]).toEqual({
    name: 'Completed',
    count: 30
  });
});
```

---

### Database Design Considerations

#### Why Names Aren't Always Unique

**Valid reasons for non-unique display names**:
1. **User flexibility**: Users can name things however they want
2. **Multi-tenancy**: Different orgs can use same names
3. **Historical reasons**: Legacy data before uniqueness was enforced
4. **External IDs are unique**: External provider IDs are unique, local names aren't

**Example Schema**:
```sql
CREATE TABLE entities (
  org_id UUID NOT NULL,
  id UUID NOT NULL,
  name VARCHAR(100) NOT NULL,        -- Display name, can duplicate
  provider_id VARCHAR(50),            -- External ID, unique per org
  PRIMARY KEY (org_id, id),
  UNIQUE (org_id, provider_id)       -- External ID is unique
);
-- Note: No UNIQUE constraint on name!
```

#### When to Enforce Uniqueness

**Add unique constraint if**:
- Name is used as lookup key (not just display)
- Duplicate names would confuse users
- Business logic requires uniqueness

**Skip unique constraint if**:
- Name is purely display label
- IDs are used for all operations
- Users expect naming flexibility

---

### General Lessons Learned

#### 1. Stable Identifiers vs Display Values

| Property | Stable ID | Display Name |
|----------|-----------|--------------|
| Purpose | Database relationships | User presentation |
| Uniqueness | Guaranteed by schema | Not guaranteed |
| Changes | Never (immutable) | Can change |
| Aggregation | ✅ Safe | ⚠️ Dangerous |

#### 2. Data Integrity Principles

**Golden Rule**: Aggregate by stable identifiers, map to display values last.

```typescript
// General pattern
function aggregateData<T>(
  items: T[],
  getStableKey: (item: T) => string,
  getDisplayName: (item: T) => string
): AggregatedItem[] {
  // 1. Group by stable key
  const grouped = groupBy(items, getStableKey);

  // 2. Aggregate within groups
  const aggregated = Object.entries(grouped).map(([key, group]) => ({
    key,
    ...aggregateMetrics(group),
    firstItem: group[0]  // Keep original for display mapping
  }));

  // 3. Map to display names
  return aggregated.map(item => ({
    name: getDisplayName(item.firstItem),
    ...item.metrics
  }));
}
```

#### 3. Type Safety Considerations

```typescript
// Make aggregation dimension explicit in type system
type AggregationDimension =
  | 'entityId'      // Aggregate by ID
  | 'status'        // Aggregate by value
  | 'date';         // Aggregate by value

function shouldAggregateById(dimension: AggregationDimension): boolean {
  return dimension === 'entityId';  // Expand as needed
}
```

#### 4. Documentation Requirements

**Always document**:
- Which fields are unique vs non-unique
- Why aggregation uses IDs (not names)
- How charts handle duplicate labels

```typescript
/**
 * Aggregate analytics buckets for chart display.
 *
 * IMPORTANT: For entity dimensions (like entityId), aggregates by stable ID
 * to prevent merging distinct entities with duplicate display names.
 * Maps to display names only after aggregation.
 *
 * @param buckets - Raw analytics data grouped by dimension
 * @param dimension - Which field to group by ('entityId', 'status', etc.)
 * @param nameMap - Optional ID→Name mapping for entity dimensions
 * @returns Aggregated chart data (may have duplicate names if entities share names)
 */
```

---

### Quick Reference

| Situation | Aggregate By | Map Names |
|-----------|--------------|-----------|
| User IDs, Assistant IDs, Product IDs | Stable UUID | After aggregation |
| Status, Type, Direction | Display value directly | During aggregation |
| Dates, Time buckets | Date value | N/A (dates are display) |
| Any non-unique field | ID or unique key | After aggregation |

---

### Testing Checklist

When implementing data aggregation:

- [ ] Identify all entity reference fields (IDs)
- [ ] Check database schema for uniqueness constraints
- [ ] Test with duplicate display names
- [ ] Verify non-entity dimensions merge correctly
- [ ] Document aggregation strategy in code comments
- [ ] Add test cases for duplicate name scenarios

---

**Key Takeaway**: Aggregating by display names is a silent data corruption bug. Always aggregate by stable IDs for entity references, map to names last. Test with duplicate names to catch this early.

---

**Overall Key Takeaway**: Most bugs occur at boundaries (dates, async timing, zero values, non-unique identifiers). Always test edges, not just happy paths.
