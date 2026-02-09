# React Async State Race Conditions

## Problem

When a function updates state and immediately uses that state, it receives **stale values** because React state updates are asynchronous.

### Example Scenario

```typescript
const [startDate, setStartDate] = useState(new Date());
const [endDate, setEndDate] = useState(new Date());

const fetchData = () => {
  // Uses current state values
  api.getData({ from: startDate, to: endDate });
};

const handlePresetClick = (days: number) => {
  const newStart = new Date();
  const newEnd = new Date();
  newEnd.setDate(newStart.getDate() + days);

  setStartDate(newStart); // State update is async
  setEndDate(newEnd); // State update is async
  fetchData(); // ❌ Uses OLD startDate/endDate values!
};
```

### Why This Happens

1. `setStartDate()` and `setEndDate()` schedule state updates but don't execute immediately
2. `fetchData()` executes synchronously using the current (stale) state values
3. State updates apply after the function completes
4. User sees data fetched with wrong date range

## Solution Pattern: Pass Values Directly

Instead of relying on state, pass the new values as parameters.

### Pattern 1: Accept Override Parameters

```typescript
interface FetchOptions {
  from?: Date;
  to?: Date;
}

const fetchData = (overrides?: FetchOptions) => {
  const finalFrom = overrides?.from ?? startDate;
  const finalTo = overrides?.to ?? endDate;

  api.getData({ from: finalFrom, to: finalTo });
};

const handlePresetClick = (days: number) => {
  const newStart = new Date();
  const newEnd = new Date();
  newEnd.setDate(newStart.getDate() + days);

  setStartDate(newStart);
  setEndDate(newEnd);
  fetchData({ from: newStart, to: newEnd }); // ✅ Uses new values directly
};
```

### Pattern 2: Compute Values Just-in-Time

```typescript
const handlePresetClick = (days: number) => {
  const newStart = new Date();
  const newEnd = new Date();
  newEnd.setDate(newStart.getDate() + days);

  // Update state for UI
  setStartDate(newStart);
  setEndDate(newEnd);

  // Fetch with computed values immediately
  api.getData({ from: newStart, to: newEnd }); // ✅ No race condition
};
```

### Pattern 3: Use useEffect for State-Dependent Side Effects

```typescript
const [startDate, setStartDate] = useState(new Date());
const [endDate, setEndDate] = useState(new Date());
const [shouldRefetch, setShouldRefetch] = useState(false);

useEffect(() => {
  if (shouldRefetch) {
    api.getData({ from: startDate, to: endDate }); // ✅ Uses updated state
    setShouldRefetch(false);
  }
}, [startDate, endDate, shouldRefetch]);

const handlePresetClick = (days: number) => {
  const newStart = new Date();
  const newEnd = new Date();
  newEnd.setDate(newStart.getDate() + days);

  setStartDate(newStart);
  setEndDate(newEnd);
  setShouldRefetch(true); // Trigger effect after state updates
};
```

## Anti-Patterns to Avoid

### ❌ Don't: Use State Immediately After Setting

```typescript
setCount(5);
console.log(count); // Still old value!
```

### ❌ Don't: Chain Multiple State-Dependent Calls

```typescript
const handleAction = () => {
  updateState();
  doSomethingWithState(); // Uses old state
  doSomethingElse(); // Uses old state
};
```

### ❌ Don't: Use setTimeout as a "Fix"

```typescript
setStartDate(newDate);
setTimeout(() => fetchData(), 0); // ❌ Brittle, timing-dependent
```

## When to Use Each Pattern

| Pattern                      | Use When                                 | Example                        |
| ---------------------------- | ---------------------------------------- | ------------------------------ |
| **Override Parameters**      | Multiple functions need same flexibility | Form submissions, API calls    |
| **Just-in-Time Computation** | One-off actions with clear values        | Preset buttons, quick actions  |
| **useEffect**                | Side effect depends on state changes     | Auto-save, syncing, validation |

## Key Takeaways

1. **React state updates are asynchronous** - never assume state is updated immediately
2. **Pass values directly** when you need them right away
3. **Use useEffect** when side effects should respond to state changes
4. **Avoid setTimeout hacks** - they're fragile and hide the real issue
5. **Design APIs to accept overrides** for maximum flexibility

## Related Patterns

- [UI Behavior Synchronization](../react/ui-behavior-synchronization.md)
- [useEffect Dependency Array Pitfalls](../react/useeffect-dependency-array-pitfalls.md)

## References

- [React Docs: State as a Snapshot](https://react.dev/learn/state-as-a-snapshot)
- [React Docs: Queueing a Series of State Updates](https://react.dev/learn/queueing-a-series-of-state-updates)
