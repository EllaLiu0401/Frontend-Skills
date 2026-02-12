# PR Comment Lessons: Redundant Function Wrappers

## Context
A reviewer questioned why a function was wrapped in `() => fn()` before passing it to another function.

## Core Lesson
If a function already matches the required signature, pass it directly instead of wrapping it.

## Before / After Thinking Pattern
- Before: "Wrap it just in case."
- After: "Use the simplest form that already satisfies the type contract."

## Why This Matters
- Avoids unnecessary indirection.
- Improves readability for future maintainers.
- Reduces chances of accidental behavior drift during refactors.
- Keeps code aligned with KISS: remove code that does not add value.

## Quick Decision Checklist
- Does the original function already match the expected parameter and return types?
- Does the wrapper change behavior (arguments, `this`, timing, error handling)?
- Is there a documented bug that requires the wrapper?

If answers are **yes / no / no**, prefer passing the function directly.

## Good PR Reply Structure
- State what changed (before vs after).
- Clarify whether behavior changed (usually no for this case).
- Explain the intent: simplify, improve clarity, keep typing explicit and correct.

## One-Line Rule
Do not add wrappers without a clear behavioral reason.
