# Prevent Infinite Loading on API Failure

## Why this matters

A common frontend bug is showing a loading spinner forever when a request fails but the UI only checks for "data is missing" instead of handling explicit error states.

This creates a dead-end UX: users cannot proceed, recover, or understand what happened.

## Core lesson

Always separate these states clearly:

1. **Loading**: request is in progress.
2. **Unauthorized**: user needs to re-authenticate (for example, HTTP 401).
3. **Other failure**: server/network/transient issues that should show a recoverable error UI.
4. **Success**: data is available.

If you merge loading and missing-data checks (for example, `isLoading || !data`) without handling errors first, non-auth failures can accidentally look like permanent loading.

## Recommended pattern (framework-agnostic)

- **Keep loading logic strict**: show spinner only while request is actively loading.
- **Handle auth failures explicitly**: redirect or re-authenticate only for unauthorized errors.
- **Handle non-auth failures with recovery**:
  - Show a user-friendly message.
  - Offer retry.
  - Do not trap users in spinner state.
- **Use centralized error formatting**:
  - Reuse shared error parsing/translation logic.
  - Avoid ad-hoc per-page message handling when a global pattern already exists.

## Before / After mindset

**Before**
- Page logic mixed loading and missing data.
- Only unauthorized errors were explicitly handled.
- Non-auth API failures could leave UI stuck in loading forever.

**After**
- Loading, unauthorized, and generic error states are explicitly separated.
- Unauthorized flow remains unchanged.
- Non-auth failures render a recoverable error state with retry.
- Error messages use shared error-handling utilities for consistency.

## KISS checklist

When fixing this class of issue:

- Make the smallest safe change in the affected UI flow.
- Preserve existing successful and auth redirect behavior.
- Avoid introducing new abstractions unless repeated patterns demand it.
- Verify lint/type checks and tests after the change.

