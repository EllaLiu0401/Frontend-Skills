# UI-Behavior Synchronization

> **TL;DR**: Never ship interactive UI elements without working behavior. Use feature flags or hide incomplete features.

## The Problem

Exposing buttons, links, or other interactive elements to users before the underlying functionality is implemented creates a frustrating user experience and erodes trust.

## Anti-Patterns

### ❌ Empty Click Handlers

```tsx
// ❌ Bad: Button does nothing when clicked
export const Dashboard = () => {
  return (
    <div>
      <button onClick={() => {}}>
        Export Report
      </button>
      <button onClick={() => console.log('TODO')}>
        Share Dashboard
      </button>
    </div>
  );
};
```

**Problem**: Users see the buttons, click them, and nothing happens. No feedback, no action, no indication it's incomplete.

### ❌ Dead Links

```tsx
// ❌ Bad: Link navigates to non-existent route
<nav>
  <Link to="/dashboard">Dashboard</Link>
  <Link to="/settings">Settings</Link>  {/* Route doesn't exist yet */}
  <Link to="/analytics">Analytics</Link>  {/* Not implemented */}
</nav>
```

**Problem**: Clicking leads to 404 or broken page, breaking user flow.

### ❌ Non-Functional Forms

```tsx
// ❌ Bad: Form submits but does nothing
export const ContactForm = () => {
  return (
    <form onSubmit={(e) => e.preventDefault()}>
      <input type="email" placeholder="Your email" />
      <button type="submit">Subscribe</button>
    </form>
  );
};
```

**Problem**: User fills out form, clicks submit, and receives no confirmation or action.

## Correct Patterns

### ✅ Hide Incomplete Features

```tsx
// ✅ Good: Only show when functionality exists
export const Dashboard = () => {
  const exportFeatureEnabled = useFeatureFlag('export-reports');
  
  return (
    <div>
      {exportFeatureEnabled && (
        <button onClick={handleExportReport}>
          Export Report
        </button>
      )}
      
      {/* Don't show share button until implemented */}
      {false && (
        <button onClick={handleShare}>
          Share Dashboard
        </button>
      )}
    </div>
  );
};
```

### ✅ Disable with Clear Communication

```tsx
// ✅ Good: Disable and explain why
export const Dashboard = () => {
  return (
    <div>
      <button 
        onClick={handleExport}
        disabled={!hasData}
      >
        Export Report
      </button>
      {!hasData && (
        <p className="text-muted">
          Add data to enable export
        </p>
      )}
      
      <Tooltip content="Coming in v2.0">
        <button disabled>
          AI Insights
        </button>
      </Tooltip>
    </div>
  );
};
```

### ✅ Progressive Disclosure

```tsx
// ✅ Good: Build feature completely before exposing
export const Dashboard = () => {
  const { data, isLoading } = useDashboardData();
  const canExport = data && data.length > 0;
  
  const handleExport = async () => {
    try {
      await exportService.generateReport(data);
      toast.success('Report exported!');
    } catch (error) {
      toast.error('Export failed. Please try again.');
    }
  };
  
  return (
    <div>
      {canExport && (
        <button onClick={handleExport}>
          Export Report
        </button>
      )}
    </div>
  );
};
```

### ✅ Feature Flags for Gradual Rollout

```tsx
// ✅ Good: Control visibility with feature flags
import { useFeatureFlag } from '@/hooks/useFeatureFlag';

export const Dashboard = () => {
  const showAnalytics = useFeatureFlag('analytics-panel');
  const showExport = useFeatureFlag('export-feature');
  
  return (
    <div>
      {showAnalytics && (
        <Link to="/analytics">
          Analytics Dashboard
        </Link>
      )}
      
      {showExport && (
        <ExportButton onExport={handleExport} />
      )}
    </div>
  );
};
```

## Implementation Strategies

### Strategy 1: Feature Flags

```typescript
// utils/featureFlags.ts
const FEATURES = {
  'export-reports': process.env.REACT_APP_ENABLE_EXPORT === 'true',
  'ai-insights': false, // Not ready yet
  'team-collaboration': true,
} as const;

export const useFeatureFlag = (flag: keyof typeof FEATURES): boolean => {
  return FEATURES[flag];
};
```

### Strategy 2: Permission-Based Visibility

```tsx
// Only show features user has access to
export const AdminPanel = () => {
  const { user } = useAuth();
  const canManageUsers = user.permissions.includes('manage-users');
  
  return (
    <div>
      {canManageUsers && (
        <button onClick={handleManageUsers}>
          Manage Users
        </button>
      )}
    </div>
  );
};
```

### Strategy 3: Loading States

```tsx
// Show loading state instead of disabled button
export const DataExporter = () => {
  const { data, isLoading } = useData();
  
  if (isLoading) {
    return <Skeleton.Button />;
  }
  
  if (!data || data.length === 0) {
    return null; // Don't show export if no data
  }
  
  return (
    <button onClick={handleExport}>
      Export {data.length} items
    </button>
  );
};
```

## Testing Checklist

Before shipping interactive UI:

- [ ] Every button has a working `onClick` handler
- [ ] Every link navigates to an existing route
- [ ] Every form submission has backend integration or proper handling
- [ ] Loading states are shown during async operations
- [ ] Error states are handled and communicated to users
- [ ] Success feedback is provided after actions complete
- [ ] Disabled states have clear explanations
- [ ] Incomplete features are hidden or behind feature flags

## Common Scenarios

### Scenario 1: Building Feature Incrementally

```tsx
// Phase 1: Hide feature completely
const SHOW_EXPORT = false;

export const Dashboard = () => (
  <div>
    {SHOW_EXPORT && <ExportButton />}  {/* Hidden during development */}
  </div>
);

// Phase 2: Enable in development only
const SHOW_EXPORT = process.env.NODE_ENV === 'development';

// Phase 3: Enable for specific users (beta)
const SHOW_EXPORT = user.betaFeatures?.includes('export');

// Phase 4: Enable for everyone
const SHOW_EXPORT = true;
```

### Scenario 2: Navigation Menu

```tsx
// ❌ Bad: Show all links even if pages don't exist
<nav>
  <Link to="/home">Home</Link>
  <Link to="/profile">Profile</Link>
  <Link to="/settings">Settings</Link>
</nav>

// ✅ Good: Only show implemented pages
const AVAILABLE_ROUTES = ['/home', '/profile'];

<nav>
  <Link to="/home">Home</Link>
  <Link to="/profile">Profile</Link>
  {AVAILABLE_ROUTES.includes('/settings') && (
    <Link to="/settings">Settings</Link>
  )}
</nav>
```

### Scenario 3: Action Buttons

```tsx
// ❌ Bad: Button with empty handler
<button onClick={() => {}}>Delete</button>

// ✅ Good: Button with full implementation
<button 
  onClick={() => {
    if (confirm('Are you sure?')) {
      deleteItem(id)
        .then(() => toast.success('Deleted'))
        .catch(() => toast.error('Failed to delete'));
    }
  }}
>
  Delete
</button>

// ✅ Alternative: Hide until implemented
{deleteEnabled && (
  <button onClick={handleDelete}>
    Delete
  </button>
)}
```

## Anti-Pattern Detection

Use ESLint to catch empty handlers:

```javascript
// .eslintrc.js
module.exports = {
  rules: {
    'react/jsx-no-bind': ['warn', {
      allowArrowFunctions: true,
      allowEmptyFunctions: false, // Warn on () => {}
    }],
  },
};
```

## Best Practices Summary

1. **Never ship empty handlers**: Every interactive element needs working behavior
2. **Hide incomplete features**: Use feature flags or conditional rendering
3. **Disable with explanation**: If shown but not ready, explain why it's disabled
4. **Provide feedback**: Always confirm actions with loading, success, or error states
5. **Test interactivity**: Click every button, submit every form before shipping
6. **Progressive disclosure**: Build complete features, then expose them
7. **Use feature flags**: Control visibility without code changes

## Related Resources

- [Feature Flag Patterns](../api-integration/)
- [Error Handling UI](../error-handling/)
- [PR-0161: UI Triggers Without Behavior](../best-practices/pr-0161-dashboard-foundation.md)

---

**Key Takeaway**: If users can see it and click it, it must work. Otherwise, hide it.
