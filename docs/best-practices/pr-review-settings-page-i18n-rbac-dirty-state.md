# PR Review Learnings: i18n Wiring, RBAC-Aware Fetching, and Dirty State Guards

## Context

During a PR review for a settings page that allows editing an organisation's display name, several issues were caught: an i18n key defined but never wired to the UI, a `useEffect` that could overwrite in-flight user edits, an API call made for roles that lack permission, hardcoded user-facing strings, and an external network call held inside a database transaction.

---

## Lesson 1: Wire Every i18n Key to the UI

**Problem**: A translation key was added to the locale file and a test asserted its presence, but the actual component never rendered it.

```typescript
// locale file has:
// "name": { "slug": "Slug: {slug} (cannot be changed)" }

// ❌ Component renders the FormField without the hint
<FormField label={t('name.label')} error={nameError}>
  <Input ... />
</FormField>

// Test expects the hint to be visible
expect(screen.getByText('name.slug')).toBeInTheDocument(); // FAILS
```

**Fix**: Always check that every i18n key you add is actually consumed in the component.

```typescript
// ✅ Wire the hint prop to the FormField
<FormField
  label={t('name.label')}
  hint={data?.slug ? t('name.slug', { slug: data.slug }) : undefined}
  error={nameError}
>
  <Input ... />
</FormField>
```

**Takeaway**: When adding translation keys, trace the full path: locale file → component prop → rendered output → test assertion. A missing link at any step means the feature is broken.

### Checklist

- [ ] Key exists in locale JSON
- [ ] Component calls `t('key')` and passes result to a rendered prop
- [ ] Test verifies the text appears in the DOM
- [ ] Dynamic interpolation values (`{slug}`, `{count}`) are passed correctly

---

## Lesson 2: Guard useEffect Sync with a Dirty Flag

**Problem**: A `useEffect` syncs server data into local state. If the server cache refreshes while the user is mid-edit, their typed input gets silently overwritten.

```typescript
const [nameInput, setNameInput] = useState('');

// ❌ Unconditionally overwrites local state on every server data change
useEffect(() => {
  if (serverData?.name) setNameInput(serverData.name);
}, [serverData?.name]);
```

**Scenario**: User starts typing "New Name" → a background React Query refetch resolves → `useEffect` fires → input resets to the old server value → user's changes are lost with no warning.

**Fix**: Track whether the user has modified the input. Only sync from server when the input is clean.

```typescript
const [nameInput, setNameInput] = useState('');
const [isDirty, setIsDirty] = useState(false);

// ✅ Only sync server data when user hasn't started editing
useEffect(() => {
  if (serverData?.name && !isDirty) setNameInput(serverData.name);
}, [serverData?.name, isDirty]);

// Mark dirty on user input
const handleChange = (e) => {
  setNameInput(e.target.value);
  setIsDirty(true);
};

// Reset dirty flag after successful save
const handleSaveSuccess = () => {
  setIsDirty(false);
};
```

**Takeaway**: Any time you sync external/async data into controlled input state via `useEffect`, you must guard against overwriting in-flight user edits. The `isDirty` pattern is the simplest solution.

### When This Matters

- Forms that auto-refresh data in the background (React Query, SWR, polling)
- Collaborative editing where multiple sources can update the same value
- Settings pages where save is explicit (button click), not automatic

---

## Lesson 3: RBAC-Aware Data Fetching on Shared Pages

**Problem**: A profile page called an API endpoint restricted to admin/owner roles. Users with a lower role (e.g. analyst) hit a guaranteed 403 on every page load — the org name field silently failed, and the server logged unnecessary RBAC-denied errors.

```typescript
// ❌ Calls admin-only endpoint for ALL users with an orgId
const orgResult = orgId ? await getOrganisation() : undefined;
const orgName = orgResult?.data?.name;
```

**Fix**: Check the user's role before calling restricted endpoints.

```typescript
// ✅ Only call admin-restricted endpoint for users who have permission
const orgResult = orgId && isAdmin ? await getOrganisation() : undefined;
const orgName = orgResult?.data?.name;
```

**Takeaway**: When a page is accessible to multiple roles but fetches data from a role-restricted endpoint, always gate the call by role. Otherwise you generate:
- Silent failures (data just doesn't show, no error message to user)
- Wasted network requests (every page load triggers a 403)
- Noisy server logs (RBAC denials flood error monitoring)

### Pattern

```typescript
// Generic pattern for role-gated server calls
const shouldFetch = hasPermission(user, requiredRole);
const result = shouldFetch ? await fetchRestrictedData() : undefined;
```

---

## Lesson 4: Never Hardcode User-Facing Strings

**Problem**: A button label was hardcoded as a string literal instead of going through the i18n system.

```typescript
// ❌ Hardcoded string — breaks i18n, won't translate
<Button>{isPending ? <Loading /> : 'Save'}</Button>

// ✅ Use translation function
<Button>{isPending ? <Loading /> : t('name.save')}</Button>
```

**Takeaway**: ALL user-visible text must use the i18n system — including button labels, aria-labels, error messages, tooltips, and placeholder text. This is easy to miss on small labels like "Save", "Cancel", "OK".

### Quick Grep Audit

Before submitting a PR, search for common hardcoded patterns:

```bash
# Find likely hardcoded English strings in TSX files
rg "'(Save|Cancel|Delete|Submit|OK|Back|Close|Edit|Add|Remove)'" --type tsx
rg '"(Save|Cancel|Delete|Submit|OK|Back|Close|Edit|Add|Remove)"' --type tsx
```

---

## Lesson 5: Keep External Calls Outside Database Transactions

**Problem**: An external API call (e.g. updating a third-party identity provider) was made inside a database transaction. The transaction held row locks open for the duration of the network round-trip.

```typescript
// ❌ External call inside transaction — holds locks during network latency
await db.transaction().execute(async (trx) => {
  await setDbContext(trx, context);
  const updated = await repo.updateName(trx, orgId, name);

  // This can take 200ms–5s depending on external service
  await externalService.syncName(externalId, name);

  return updated;
});
```

**Fix**: Commit the DB transaction first, then make the external call as a best-effort sync.

```typescript
// ✅ DB transaction commits fast; external sync is best-effort after
const updated = await db.transaction().execute(async (trx) => {
  await setDbContext(trx, context);
  return await repo.updateName(trx, orgId, name);
});

// Best-effort sync — log warning on failure, don't roll back DB
try {
  await externalService.syncName(externalId, name);
} catch (err) {
  log.warn({ orgId, err }, 'External sync failed — DB is source of truth');
}
```

**Takeaway**: Database transactions should be as short as possible. Never include network calls to external services inside a transaction — it increases lock contention and creates inconsistency risks (external succeeds but DB commit fails).

### Rule of Thumb

| Inside Transaction | Outside Transaction |
|--------------------|---------------------|
| DB reads/writes | External API calls |
| Context setup | Email/notification sends |
| Constraint checks | Webhook dispatches |
| Audit log inserts | Cache invalidation |

---

## Lesson 6: Consistent Styling Within Components

**Problem**: New elements added to an existing component used a different CSS class for the same visual purpose (e.g. label color), creating visual inconsistency.

```typescript
// Existing labels use one class
<Text className="font-medium text-base-content/60">Name</Text>
<Text className="font-medium text-base-content/60">Email</Text>

// ❌ New label uses a different class for the same purpose
<Text className="font-medium text-secondary">Org Name</Text>
```

**Fix**: Always match the existing pattern within the same component.

**Takeaway**: Before adding new UI elements, scan the existing component for the pattern being used. Inconsistency creates visual bugs that are hard to catch in code review but obvious to users.

---

## Summary Checklist

When submitting a PR that involves forms, settings pages, or RBAC-gated features:

- [ ] **i18n completeness**: Every translation key in locale files is wired to a component prop
- [ ] **No hardcoded strings**: `rg` for common English words in TSX files
- [ ] **Dirty state guard**: useEffect syncs to controlled inputs are gated by an `isDirty` flag
- [ ] **RBAC-aware fetching**: Restricted endpoints are only called for users with the correct role
- [ ] **Transaction scope**: No external/network calls inside DB transactions
- [ ] **Style consistency**: New elements match existing patterns in the same component
- [ ] **Test alignment**: Tests assert behavior that the component actually implements

---

**Tags**: `#react` `#i18n` `#rbac` `#forms` `#useEffect` `#database` `#pr-review`

**Date Added**: 2026-03-31

**Difficulty**: Intermediate
