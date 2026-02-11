# React Rules of Hooks

## Overview

The Rules of Hooks are fundamental constraints in React that ensure hooks work correctly and maintain state consistently across renders. Violating these rules leads to runtime errors, lost state, and unpredictable behavior.

## The Two Rules

### Rule 1: Only Call Hooks at the Top Level

**❌ DON'T** call hooks inside:
- Conditionals (`if`, `switch`)
- Loops (`for`, `while`, `map`)
- Nested functions
- Early returns

**✅ DO** call hooks:
- At the top level of your component or custom hook
- In the same order on every render

### Rule 2: Only Call Hooks from React Functions

**✅ Call hooks from:**
- React function components
- Custom hooks (functions starting with `use`)

**❌ DON'T call hooks from:**
- Regular JavaScript functions
- Class components
- Event handlers
- Helper functions

## Why These Rules Matter

React relies on the **order of hook calls** to maintain state between renders. React doesn't use variable names to track state—it uses call order.

```javascript
// First render
useState(0)  // Hook 1 → State slot 1
useState('') // Hook 2 → State slot 2
useEffect()  // Hook 3 → Effect slot 1

// Second render (must be same order!)
useState(0)  // Hook 1 → State slot 1
useState('') // Hook 2 → State slot 2
useEffect()  // Hook 3 → Effect slot 1
```

If the order changes between renders, React loses track of which state belongs to which hook.

## Common Violations and Fixes

### ❌ Violation 1: Conditional Hook Calls

**Bad Example:**
```javascript
function useData(options) {
  if (options.useCache) {
    return useQuery(['data'], fetchData); // ❌ Conditional hook
  }

  return useMemo(() => localData, []); // ❌ Conditional hook
}
```

**Problem:** If `options.useCache` changes between renders, the hook call order changes, causing React to lose state.

**✅ Fix: Call Hooks Unconditionally, Control with Logic**

```javascript
function useData(options) {
  // ✅ Always call both hooks in the same order
  const queryResult = useQuery(['data'], fetchData, {
    enabled: options.useCache, // Control execution with enabled
  });

  const memoResult = useMemo(() => localData, []);

  // Return the appropriate result
  return options.useCache ? queryResult : memoResult;
}
```

### ❌ Violation 2: Hooks in Loops

**Bad Example:**
```javascript
function FormFields({ fields }) {
  return fields.map((field) => {
    const [value, setValue] = useState(''); // ❌ Hook in loop
    return <input value={value} onChange={e => setValue(e.target.value)} />;
  });
}
```

**Problem:** The number of hook calls changes if `fields` length changes.

**✅ Fix: Move Hook to Child Component**

```javascript
function FormField({ field }) {
  const [value, setValue] = useState(''); // ✅ Hook at top level of component
  return <input value={value} onChange={e => setValue(e.target.value)} />;
}

function FormFields({ fields }) {
  return fields.map((field) => (
    <FormField key={field.id} field={field} />
  ));
}
```

### ❌ Violation 3: Hooks After Early Return

**Bad Example:**
```javascript
function useUser(id) {
  if (!id) {
    return null; // ❌ Early return before hook
  }

  const user = useQuery(['user', id], () => fetchUser(id)); // Hook after return
  return user;
}
```

**Problem:** When `id` is falsy, the hook isn't called, changing the hook call order.

**✅ Fix: Call Hook Before Conditional Logic**

```javascript
function useUser(id) {
  // ✅ Always call hook, control with enabled option
  const user = useQuery(['user', id], () => fetchUser(id), {
    enabled: Boolean(id),
  });

  if (!id) {
    return null; // Return after hooks
  }

  return user;
}
```

### ❌ Violation 4: Multiple Conditional Hooks

**Bad Example:**
```javascript
function useDataFetcher(options) {
  if (options.type === 'query') {
    return useQuery(options.key, options.fetcher); // ❌ Conditional
  }

  if (options.type === 'mutation') {
    return useMutation(options.fetcher); // ❌ Conditional
  }

  return useCallback(options.fetcher, []); // ❌ Conditional
}
```

**✅ Fix: Call All Hooks Unconditionally**

```javascript
function useDataFetcher(options) {
  const isQuery = options.type === 'query';
  const isMutation = options.type === 'mutation';
  const isCallback = options.type === 'callback';

  // ✅ Always call all hooks in the same order
  const queryResult = useQuery(options.key, options.fetcher, {
    enabled: isQuery,
  });

  const mutationResult = useMutation(options.fetcher, {
    // Mutations don't execute unless explicitly called, so no need for enabled
  });

  const callbackResult = useCallback(options.fetcher, []);

  // Return the appropriate result
  if (isQuery) return queryResult;
  if (isMutation) return mutationResult;
  return callbackResult;
}
```

## Best Practices

### ✅ 1. Use ESLint Plugin

Install and configure `eslint-plugin-react-hooks`:

```json
{
  "plugins": ["react-hooks"],
  "rules": {
    "react-hooks/rules-of-hooks": "error",
    "react-hooks/exhaustive-deps": "warn"
  }
}
```

This catches most violations automatically.

### ✅ 2. Control Execution with Options

Many hooks support `enabled` or similar options to control execution:

```javascript
// React Query
const { data } = useQuery(key, fetcher, { enabled: condition });

// Custom hooks can implement similar patterns
function useCustomHook(options) {
  const [state, setState] = useState(null);

  useEffect(() => {
    if (!options.enabled) return; // Control execution inside hook

    // Do work
  }, [options.enabled]);

  return state;
}
```

### ✅ 3. Extract to Child Components

When you need different hooks based on conditions, split into separate components:

```javascript
// Instead of conditional hooks:
function Parent({ mode }) {
  if (mode === 'A') {
    const dataA = useHookA(); // ❌ Conditional
    return <ViewA data={dataA} />;
  }
  const dataB = useHookB(); // ❌ Conditional
  return <ViewB data={dataB} />;
}

// ✅ Split into components
function ComponentA() {
  const data = useHookA(); // ✅ Always called
  return <ViewA data={data} />;
}

function ComponentB() {
  const data = useHookB(); // ✅ Always called
  return <ViewB data={data} />;
}

function Parent({ mode }) {
  return mode === 'A' ? <ComponentA /> : <ComponentB />;
}
```

### ✅ 4. Use Null/Undefined Returns from Hooks

If you need to skip a hook's execution, use conditional logic inside the hook:

```javascript
function useConditionalData(shouldFetch) {
  const [data, setData] = useState(null); // ✅ Always called

  useEffect(() => {
    if (!shouldFetch) return; // Skip execution inside hook

    fetchData().then(setData);
  }, [shouldFetch]);

  return shouldFetch ? data : null;
}
```

## Real-World Example: Conditional Data Fetching

### ❌ Problematic Pattern

```javascript
function useUserData(userId) {
  // User changes userId from null to "123"
  if (!userId) {
    return { data: null, loading: false }; // ❌ Early return before hook
  }

  // Hook call order changes when userId becomes truthy
  const { data, loading } = useQuery(['user', userId], () => fetchUser(userId));
  return { data, loading };
}
```

### ✅ Correct Pattern

```javascript
function useUserData(userId) {
  // ✅ Always call hook in the same order
  const { data, loading } = useQuery(
    ['user', userId],
    () => fetchUser(userId),
    {
      enabled: Boolean(userId), // Control execution, not hook call
    }
  );

  // Conditional logic after hooks
  if (!userId) {
    return { data: null, loading: false };
  }

  return { data, loading };
}
```

## Common Pitfalls

### Pitfall 1: "It Works in Development"

Sometimes conditional hooks appear to work because:
- The condition doesn't change during testing
- Development mode is more forgiving
- The bug only manifests under specific state transitions

**Always follow the rules**, even if it seems to work.

### Pitfall 2: Custom Hooks with Different Hook Counts

```javascript
// ❌ Bad: Different number of hooks based on input
function useFeature(config) {
  if (config.advanced) {
    useEffect(() => { /* advanced logic */ }, []);
    useCallback(() => { /* advanced callback */ }, []);
    return advancedFeature;
  }
  return basicFeature;
}

// ✅ Good: Same hooks, different execution
function useFeature(config) {
  useEffect(() => {
    if (config.advanced) {
      /* advanced logic */
    }
  }, [config.advanced]);

  const callback = useCallback(() => {
    if (config.advanced) {
      /* advanced callback */
    }
  }, [config.advanced]);

  return config.advanced ? advancedFeature : basicFeature;
}
```

### Pitfall 3: Hooks in Callbacks

```javascript
function Component() {
  const handleClick = () => {
    const [count, setCount] = useState(0); // ❌ Hook in callback
    setCount(count + 1);
  };

  return <button onClick={handleClick}>Click</button>;
}

// ✅ Fix: Move state to component level
function Component() {
  const [count, setCount] = useState(0); // ✅ At top level

  const handleClick = () => {
    setCount(count + 1);
  };

  return <button onClick={handleClick}>Click</button>;
}
```

## Debugging Tips

### 1. Check Hook Call Order

When debugging unexpected behavior:
1. Add console.logs to see hook execution order
2. Verify the order is consistent across renders
3. Check if any conditionals affect hook calls

```javascript
function MyComponent({ condition }) {
  console.log('Render start');

  const [a] = useState(1);
  console.log('After useState(1)');

  const [b] = useState(2);
  console.log('After useState(2)');

  // Check if logs appear in same order every render
}
```

### 2. Use React DevTools

React DevTools shows:
- Current hooks and their values
- Hook call order
- Changes between renders

### 3. Enable Strict Mode

Strict Mode helps catch issues:

```javascript
<React.StrictMode>
  <App />
</React.StrictMode>
```

## Summary

✅ **Always:**
- Call hooks at the top level
- Call hooks in the same order every render
- Use ESLint plugin to catch violations
- Control execution with options/conditionals inside hooks

❌ **Never:**
- Call hooks conditionally
- Call hooks in loops
- Call hooks after early returns
- Call hooks in nested functions or callbacks

🎯 **Remember:** React uses call order, not variable names, to track hook state. Breaking the order breaks React's state management.

## Additional Resources

- [React Official Docs: Rules of Hooks](https://react.dev/reference/rules/rules-of-hooks)
- [ESLint Plugin: react-hooks](https://www.npmjs.com/package/eslint-plugin-react-hooks)
- [Why Do Hooks Rely on Call Order?](https://overreacted.io/why-do-hooks-rely-on-call-order/)
