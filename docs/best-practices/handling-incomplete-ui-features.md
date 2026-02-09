# Handling Incomplete UI Features

## Overview

When building features incrementally, it's important to avoid confusing users with UI elements that don't work yet. This guide covers best practices for handling incomplete features in production applications.

## The Problem: Empty Click Handlers

**Bad Example:**

```tsx
<Button
  onClick={() => {
    // TODO: Implement notification panel later
  }}
>
  Notifications (3)
</Button>
```

**Why this is problematic:**

- User sees a badge with "3 notifications"
- User clicks the button expecting something to happen
- Nothing happens → User confusion and frustration
- Damages user trust and perceived product quality

## Solution Approaches

### Approach 1: Don't Render Until Ready (Recommended)

The simplest solution following KISS principle - don't show the feature at all.

```tsx
export function AppHeader() {
  // Only render when notifications API is ready
  const { data: notifications, isLoading } = useNotifications();

  return (
    <header>
      {/* Other header content */}

      {/* Only show notifications when feature is complete */}
      {notifications && <NotificationBell count={notifications.length} />}
    </header>
  );
}
```

**Benefits:**

- ✅ No user confusion
- ✅ Simplest implementation
- ✅ Easy to add back when ready
- ✅ No maintenance burden

**When to use:**

- Feature is planned for near future (1-2 sprints)
- Mock data would be misleading
- No critical UX flow depends on it

### Approach 2: Disable State with Tooltip

If the UI element needs to be visible for design consistency or user awareness:

```tsx
<Tooltip content="Coming soon in Sprint 2">
  <Button disabled onClick={undefined} aria-label="Notifications (coming soon)">
    <BellIcon />
  </Button>
</Tooltip>
```

**Benefits:**

- ✅ Sets clear expectations
- ✅ Maintains visual design consistency
- ✅ No user frustration from clicking

**When to use:**

- Feature needs to be visible in UI for consistency
- Users need awareness that feature exists
- Part of a complete navigation pattern

### Approach 3: Show Informative Message

For features users might actively look for:

```tsx
<Button
  onClick={() => {
    toast.info("Notifications feature launching in Sprint 2!");
  }}
>
  <BellIcon />
</Button>
```

**Benefits:**

- ✅ Provides feedback instead of silence
- ✅ Sets expectations for availability

**When to use:**

- Users might actively seek this feature
- You want to gather interest/engagement data
- Feature is definitely coming soon

## Code Review Checklist

When reviewing incomplete features:

- [ ] Does clicking do nothing? (Red flag)
- [ ] Is there mock data being displayed? (Can mislead users)
- [ ] Is the feature disabled with clear indication?
- [ ] Is there a tooltip/message explaining unavailability?
- [ ] Can this be removed entirely without breaking UX?

## Anti-Patterns to Avoid

### ❌ Mock Data with Non-Functional UI

```tsx
// Bad: Shows "3 notifications" but can't view them
const mockNotificationCount = 3;
<NotificationBell count={mockNotificationCount} onClick={() => {}} />;
```

### ❌ Silent Click Handlers

```tsx
// Bad: Click does nothing, no feedback
<button
  onClick={() => {
    /* TODO */
  }}
>
  View Details
</button>
```

### ❌ Fake Loading States

```tsx
// Bad: Pretends to load but goes nowhere
<button onClick={() => setLoading(true)}>Load More</button>
```

## Best Practices Summary

1. **Default to removal** - If it doesn't work, don't show it
2. **Be honest** - Don't fake functionality with mock data
3. **Set expectations** - If visible, clearly indicate it's not ready
4. **Provide feedback** - Never leave users confused by silent clicks
5. **Document plans** - Add clear TODO comments with ticket numbers

## Real-World Example

**Before (Problematic):**

```tsx
// Shows notification badge with empty click handler
<NotificationBell
  count={3} // mock data
  onClick={() => {
    // TODO: [V2-XX] Implement in Sprint 2
  }}
/>
```

**After (Fixed):**

```tsx
// Temporarily removed, will add back in Sprint 2
// TODO: [V2-XX] Add NotificationBell when API is ready

// Later, when implementing:
<NotificationBell
  count={realNotifications.length}
  onClick={() => openNotificationsPanel()}
/>
```

## Key Takeaway

**Empty onClick handlers are user-facing bugs, not just incomplete code.** Always consider the user experience when leaving features incomplete. When in doubt, don't show it until it works.
