## User Object Typing and Safe Access

### Why this matters

When a user object comes from an SDK, it often already has a strong type. Treating it as a generic
`Record<string, unknown>` weakens type safety and triggers unnecessary runtime guards.

### Key takeaways

- **Prefer SDK-provided types.** If the SDK exports a `User` (or similar) type, use it instead of
  `Record<string, unknown>` to keep properties properly typed.
- **Only add runtime guards when the type is genuinely unsafe.** If the type says `string`,
  avoid redundant `typeof` checks or `??` fallbacks that lint rules will flag as unnecessary.
- **Handle optional fields deliberately.** For optional strings, use clear fallbacks like
  `value ?? ''` or a translation fallback, but avoid extra conversion if the value is already
  typed as `string | undefined`.
- **Align tests with stricter types.** Update mocks to match the SDK type shape (e.g., required
  properties), and avoid template-literal expressions that violate lint rules.
- **Let lint rules guide correctness.** Rules like `no-unnecessary-condition`,
  `restrict-template-expressions`, and `no-unnecessary-type-conversion` usually signal real
  type mismatches or redundant logic.

### Practical checklist

- Use the SDK’s `User` type for props and state.
- Avoid `Record<string, unknown>` unless the source is truly untyped.
- Add guards only where the type allows `undefined` or non-string values.
- Keep test fixtures in sync with the real type definitions.
