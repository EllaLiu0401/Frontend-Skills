# PR Comment Lessons: Derive Return Types from Source Models

## Context

A reviewer pointed out that a service return type was manually re-declared, even though the same shape already existed in model-layer types.

This creates a maintenance risk: if source model fields change, the manually duplicated return type can silently drift.

## Before

```typescript
// ❌ Manual duplication: easy to drift over time
export async function getProfile(userId: string): Promise<{
  id: string;
  email: string;
  displayName: string | null;
  avatarUrl: string | null;
  role: "admin" | "member";
  currentPolicyVersion: string;
}> {
  const user = await findUserById(userId);
  return {
    ...user,
    currentPolicyVersion: getCurrentPolicyVersion(),
  };
}
```

## After

```typescript
// ✅ Derive from source model, compose only service-specific fields
type UserProfileBase = Pick<
  UserModel,
  "id" | "email" | "displayName" | "avatarUrl" | "role"
>;

export async function getProfile(
  userId: string,
): Promise<UserProfileBase & { currentPolicyVersion: string }> {
  const user = await findUserById(userId);
  return {
    ...user,
    currentPolicyVersion: getCurrentPolicyVersion(),
  };
}
```

## Main Logic

- Keep field-level type ownership in one place (model/schema/repository type).
- In service or view-model layers, compose extra context fields rather than redefining base fields.
- Prefer type composition (`&`, `Pick`, `Omit`) over copy-paste type declarations.

## Why This Matters

- Prevents type drift when source fields evolve.
- Reduces duplicate maintenance work in reviews and refactors.
- Makes intent clearer: "base model + contextual fields."
- Improves safety with zero runtime behavior change.

## Quick Review Checklist

- Is this return type manually repeating fields that already exist elsewhere?
- Can I reference a source type with `Pick`/`Omit` instead of retyping everything?
- Am I only adding truly layer-specific fields in this function?
- Does this change affect runtime behavior, or only compile-time safety?

## One-Line Rule

Do not manually duplicate return shapes when an authoritative source type already exists.

---

**Tags**: `#typescript` `#code-review` `#single-source-of-truth` `#maintainability`

**Date Added**: 2026-02-12

**Difficulty**: Intermediate
