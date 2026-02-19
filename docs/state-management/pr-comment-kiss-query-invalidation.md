# PR Comment Learnings: KISS + Query Invalidation

## Context

A code review raised two concerns after a mutation flow was implemented with custom cache logic:

1. The success handler felt too complex.
2. A query key had previously been hardcoded and could drift if renamed later.

## What I Learned

- **Two comments can both be correct**: one can push for simplicity, another for maintainability.
- **Do not hardcode shared query keys**: export and reuse a single constant.
- **Prefer standard mutation patterns first**: invalidate the related query on success.
- **Avoid premature cache optimization**: custom optimistic cache writes should be used only when clearly needed.
- **Pick the simplest solution that preserves correctness** (KISS).

## Practical Decision Rule

When a mutation succeeds:

1. Reuse the shared query key constant.
2. Call query invalidation for that key.
3. Navigate/update UI flow as needed.
4. Add custom cache writes only if there is a proven issue (e.g., user-visible race condition) that invalidation cannot reasonably solve.

## Why This Works

- Keeps behavior predictable across features.
- Reduces maintenance risk and hidden coupling.
- Makes review feedback easier to address with a clear, repeatable pattern.

## How to Explain This in Review (Short Version)

"I simplified the success flow to the standard query-invalidation pattern and kept a shared query-key constant to avoid hardcoded drift. This keeps the implementation consistent, predictable, and easy to maintain."
