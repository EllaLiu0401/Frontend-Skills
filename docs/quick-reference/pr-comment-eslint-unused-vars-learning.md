# PR Learning: Unused Variables and Lint-Safe Cleanup

## What I Learned

When a reviewer suggests removing an `eslint-disable` for an unused variable, treat it as a code-quality improvement, not just a style preference.

The key is to keep behavior unchanged while making intent explicit and satisfying the project's lint rules.

## Practical Pattern

### Before

```ts
// eslint-disable-next-line @typescript-eslint/no-unused-vars
const { contextField, ...payload } = input ?? {};
```

### After

```ts
const { contextField, ...payload } = input ?? {};
void contextField;
```

## Main Logic

- Remove unnecessary lint suppression comments.
- Preserve existing behavior (the excluded field still does not enter `payload`).
- Mark the variable as intentionally unused in a lint-safe way.
- Prefer the smallest possible change (KISS), especially in shared factory/helper code.

## Review Checklist for Similar Comments

- Is this a functional bug or a code-quality cleanup?
- Will the change alter runtime behavior?
- Does the repository's ESLint config allow underscore-prefixed unused variables?
- If not, use an explicit no-op usage (for example, `void variable`).
- Run lint before commit to avoid hook failures.
