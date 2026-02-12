# PR Comment Lessons: Explicit Return Types at Data Boundaries

## Context
A reviewer suggested adding an explicit return type to a data-access function instead of relying only on inferred types.

## Core Lesson
This is usually not a runtime bug. It is a contract-clarity improvement.

Use explicit return types at module boundaries (especially exported data-access functions) when the function represents a stable contract for callers.

## Why This Matters
- Makes the function contract obvious to maintainers.
- Prevents accidental contract drift during refactors.
- Improves review quality by separating "behavior changes" from "type clarity changes."
- Keeps changes safe when done as a minimal annotation-only update.

## KISS Decision Rule
- If behavior is correct and only type clarity is missing, prefer a small explicit type update.
- Do not redesign surrounding architecture for this.
- Avoid introducing extra abstractions if a simple type alias is enough.

## Practical Pattern
- Define a focused output type (often with `Pick<...>` or an existing row/interface type).
- Annotate exported function return type with `Promise<Output | undefined>` (or the equivalent shape).
- Keep query/logic unchanged unless there is a proven behavior issue.

## Quick Review Checklist
- Did runtime behavior change? (Usually no)
- Is the explicit type aligned with selected/returned fields?
- Is the update minimal and local?
- Does it improve readability for future contributors?

## Good PR Reply Structure
- Clarify this is a type-contract improvement, not a behavior fix.
- State that runtime behavior is unchanged.
- Mention that the change follows KISS and avoids over-engineering.

## One-Line Rule
When the code works but the contract is implicit, add the smallest explicit type at the boundary.
