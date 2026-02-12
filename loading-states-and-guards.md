# Loading States and Guard Patterns in Frontend

## Problem

Common issues when fetching data before user actions:

1. **Premature submission** - User submits form before data loads
2. **Stale fallback values** - Using hardcoded defaults when API fails
3. **Poor UX** - No visual feedback during loading
4. **Race conditions** - Multiple requests interfering

## Solution: Proper Loading State Management

---

## Pattern 1: Guard Pattern

### Problem
```typescript
// ❌ Bad: No guard, uses fallback
const { data } = useFetch('/api/config');

function handleSubmit() {
  const value = data?.value ?? 'FALLBACK'; // Defeats dynamic config!
  api.submit({ value });
}
```

**Issues**:
- Fallback defeats single source of truth
- Can submit with stale data
- No indication data is missing

### Solution
```typescript
// ✅ Good: Guard prevents submission without data
const { data } = useFetch('/api/config');

function handleSubmit() {
  if (!data?.value) return; // Guard: early exit
  api.submit({ value: data.value }); // Only actual data used
}

<button disabled={!data}>Submit</button>
```

**Benefits**:
- No fallback needed
- Impossible to submit without data
- Clear loading state

---

## Pattern 2: Disable Until Ready

### Problem
```typescript
// ❌ Bad: Button always enabled
const { data, isLoading } = useFetch('/api/config');

<button onClick={handleSubmit}>
  Submit
</button>
```

**Issues**:
- User can click before data loads
- No visual feedback
- Relies on guard alone (UX issue)

### Solution
```typescript
// ✅ Good: Disable while loading
const { data, isLoading } = useFetch('/api/config');

<button
  onClick={handleSubmit}
  disabled={isLoading || !data} // Disable until ready
>
  {isLoading ? 'Loading...' : 'Submit'}
</button>
```

**Benefits**:
- Visual feedback (button disabled)
- Prevents premature clicks
- Better UX

---

## Pattern 3: Multiple Loading States

### Problem
```typescript
// ❌ Bad: Single loading flag, unclear state
const [loading, setLoading] = useState(false);

async function loadData() {
  setLoading(true);
  await fetchUser();
  await fetchConfig();
  setLoading(false);
}
```

**Issues**:
- Can't distinguish between loading states
- Can't show specific loading UI
- No error handling per request

### Solution
```typescript
// ✅ Good: Separate loading states
const { data: user, isLoading: loadingUser } = useFetch('/api/user');
const { data: config, isLoading: loadingConfig } = useFetch('/api/config');

const isReady = !loadingUser && !loadingConfig && user && config;

<button disabled={!isReady}>
  {loadingUser && 'Loading user...'}
  {loadingConfig && 'Loading config...'}
  {isReady && 'Submit'}
</button>
```

**Benefits**:
- Specific loading messages
- Fine-grained control
- Better error handling

---

## Pattern 4: Loading Skeleton

### Problem
```typescript
// ❌ Bad: Blank screen or spinner
if (isLoading) return <Spinner />;
return <Content data={data} />;
```

**Issues**:
- Layout shift when content loads
- Poor perceived performance
- No context while loading

### Solution
```typescript
// ✅ Good: Skeleton maintains layout
if (isLoading) {
  return (
    <div>
      <Skeleton className="h-8 w-64 mb-4" /> {/* Title */}
      <Skeleton className="h-4 w-full mb-2" /> {/* Line */}
      <Skeleton className="h-4 w-3/4" />       {/* Line */}
    </div>
  );
}

return <Content data={data} />;
```

**Benefits**:
- No layout shift
- Better perceived performance
- Shows structure while loading

---

## Pattern 5: Error Boundaries

### Problem
```typescript
// ❌ Bad: No error handling
const { data } = useFetch('/api/user');

return <Profile user={data} />; // Crashes if error!
```

**Issues**:
- Unhandled errors crash app
- No user feedback
- No retry mechanism

### Solution
```typescript
// ✅ Good: Graceful error handling
const { data, isLoading, error } = useFetch('/api/user');

if (isLoading) return <Skeleton />;

if (error) {
  return (
    <Alert type="error">
      Failed to load user data.
      <button onClick={refetch}>Retry</button>
    </Alert>
  );
}

if (!data) return null; // Guard

return <Profile user={data} />;
```

**Benefits**:
- Graceful degradation
- User can retry
- Clear error message

---

## Pattern 6: Optimistic Updates

### Problem
```typescript
// ❌ Bad: Wait for server response
async function handleLike() {
  const result = await api.like(postId);
  setLikes(result.likes); // Delay!
}
```

**Issues**:
- Slow perceived performance
- Unresponsive UI
- No instant feedback

### Solution
```typescript
// ✅ Good: Optimistic update with rollback
async function handleLike() {
  const previousLikes = likes;

  // Optimistic update
  setLikes(likes + 1);

  try {
    await api.like(postId);
  } catch (error) {
    // Rollback on error
    setLikes(previousLikes);
    showError('Failed to like post');
  }
}
```

**Benefits**:
- Instant feedback
- Responsive UI
- Automatic rollback on error

---

## Pattern 7: Dependent Queries

### Problem
```typescript
// ❌ Bad: Race condition
const { data: user } = useFetch('/api/user');
const { data: orders } = useFetch(`/api/orders?userId=${user?.id}`);
// orders might load with undefined userId!
```

**Issues**:
- Invalid request with undefined
- Race condition
- Wasted requests

### Solution
```typescript
// ✅ Good: Conditional query
const { data: user, isLoading: loadingUser } = useFetch('/api/user');

const { data: orders, isLoading: loadingOrders } = useFetch(
  `/api/orders?userId=${user?.id}`,
  {
    enabled: !!user?.id, // Only fetch if userId exists
  }
);

const isReady = !loadingUser && !loadingOrders && user && orders;
```

**Benefits**:
- No invalid requests
- Proper sequencing
- Efficient loading

---

## Pattern 8: Parallel Loading

### Problem
```typescript
// ❌ Bad: Sequential loading
async function loadData() {
  const user = await fetchUser();
  const config = await fetchConfig();
  const settings = await fetchSettings();
  // Takes 300ms + 200ms + 150ms = 650ms total!
}
```

**Issues**:
- Slow (sequential)
- Longer perceived load time
- Blocked by slow requests

### Solution
```typescript
// ✅ Good: Parallel loading
async function loadData() {
  const [user, config, settings] = await Promise.all([
    fetchUser(),      // 300ms
    fetchConfig(),    // 200ms
    fetchSettings(),  // 150ms
  ]);
  // Takes max(300, 200, 150) = 300ms total!
}

// Or with React Query:
const { data: user } = useFetch('/api/user');
const { data: config } = useFetch('/api/config');
const { data: settings } = useFetch('/api/settings');
// All fetched in parallel automatically!
```

**Benefits**:
- Faster loading
- Better performance
- Better UX

---

## Complete Example: Form with Dynamic Data

```typescript
import { useState } from 'react';
import { useFetch } from './hooks';

interface Config {
  minAmount: number;
  maxAmount: number;
  currentVersion: string;
}

export function PaymentForm() {
  const [amount, setAmount] = useState('');

  // Fetch configuration from API
  const {
    data: config,
    isLoading,
    error,
    refetch,
  } = useFetch<Config>('/api/config');

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();

    // Guard: Don't submit without config
    if (!config) return;

    // Use dynamic config (not hardcoded)
    await api.submitPayment({
      amount: Number(amount),
      version: config.currentVersion,
    });
  }

  // Loading state
  if (isLoading) {
    return (
      <div className="space-y-4">
        <Skeleton className="h-10 w-full" />
        <Skeleton className="h-10 w-32" />
      </div>
    );
  }

  // Error state
  if (error) {
    return (
      <Alert type="error">
        Failed to load configuration.
        <button onClick={refetch}>Retry</button>
      </Alert>
    );
  }

  // Guard: Don't render form without data
  if (!config) return null;

  return (
    <form onSubmit={handleSubmit}>
      <label>
        Amount ($)
        <input
          type="number"
          value={amount}
          onChange={(e) => setAmount(e.target.value)}
          min={config.minAmount} // Dynamic min
          max={config.maxAmount} // Dynamic max
        />
      </label>

      <button
        type="submit"
        disabled={!amount || !config} // Disable without data
      >
        Submit Payment
      </button>

      <small>
        Valid range: ${config.minAmount} - ${config.maxAmount}
      </small>
    </form>
  );
}
```

---

## React Query Example

```typescript
import { useQuery, useMutation } from '@tanstack/react-query';

export function UserProfile() {
  // Fetch user data
  const {
    data: user,
    isLoading,
    error,
  } = useQuery({
    queryKey: ['user'],
    queryFn: fetchUser,
    staleTime: 30_000, // Cache for 30s
  });

  // Update mutation
  const updateMutation = useMutation({
    mutationFn: updateUser,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['user'] });
    },
  });

  async function handleUpdate(data: UpdateData) {
    // Guard
    if (!user) return;

    await updateMutation.mutateAsync({
      userId: user.id,
      ...data,
    });
  }

  if (isLoading) return <Skeleton />;
  if (error) return <ErrorMessage error={error} />;
  if (!user) return null; // Guard

  return (
    <form onSubmit={(e) => {
      e.preventDefault();
      handleUpdate(formData);
    }}>
      {/* Form fields */}

      <button
        type="submit"
        disabled={!user || updateMutation.isPending}
      >
        {updateMutation.isPending ? 'Saving...' : 'Save'}
      </button>
    </form>
  );
}
```

---

## Best Practices Summary

### ✅ DO

- Use guards to prevent actions without data
- Disable buttons while loading
- Show specific loading states
- Handle errors gracefully
- Load independent data in parallel
- Use optimistic updates for better UX
- Show skeletons for layout stability

### ❌ DON'T

- Use fallback values that defeat dynamic config
- Allow submission before data loads
- Show blank screens during loading
- Ignore errors
- Load dependent data sequentially
- Block UI without visual feedback

---

## Checklist

- [ ] All API calls have loading states
- [ ] Buttons disabled while data loading
- [ ] Guards prevent actions without data
- [ ] No hardcoded fallback values
- [ ] Error states handled gracefully
- [ ] Loading UI maintains layout (skeleton)
- [ ] Independent queries load in parallel
- [ ] Dependent queries properly sequenced
- [ ] Optimistic updates where appropriate
- [ ] Clear loading indicators

---

## Common Pitfalls

### Pitfall 1: Forgotten Guard
```typescript
// ❌ Missing guard
const { data } = useFetch('/api/config');
api.submit({ version: data.version }); // Crash if data is undefined!
```

### Pitfall 2: Stale Fallback
```typescript
// ❌ Defeats dynamic config
const value = data?.value ?? 'HARDCODED';
```

### Pitfall 3: No Loading State
```typescript
// ❌ No feedback while loading
<button onClick={handleSubmit}>Submit</button>
```

### Pitfall 4: Layout Shift
```typescript
// ❌ Causes layout shift
if (loading) return <Spinner />; // Different height than content!
```

---

## Tools & Libraries

- **React Query** - Powerful data fetching with caching
- **SWR** - React hooks for data fetching
- **Suspense** - React's built-in loading state
- **React Loading Skeleton** - Skeleton screens
- **React Error Boundary** - Error handling

---

## Related Patterns

- **Optimistic UI** - Update UI before server confirms
- **Skeleton Screens** - Show structure while loading
- **Progressive Enhancement** - Work without JS
- **Resilient Web Design** - Graceful degradation

---

## Further Reading

- [React Query Documentation](https://tanstack.com/query/latest)
- [Optimistic UI Patterns](https://www.smashingmagazine.com/2016/11/true-lies-of-optimistic-user-interfaces/)
- [Skeleton Screen Best Practices](https://uxdesign.cc/what-you-should-know-about-skeleton-screens-a820c45a571a)
