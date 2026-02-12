# PR Comment Learning: Flow Tests Should Verify API Behavior

## Context

A review comment pointed out that a "flow test" was writing data directly into the database and then asserting the same database fields.  
This is a weak flow test because it does not verify what a real client sees.

## Core Lesson

For flow tests, prefer validating externally observable behavior (API response or user-visible output), not internal state you just wrote in setup.

## Before vs After

**Before**
- Insert test data directly into DB.
- Assert DB fields directly.
- Add extra setup data that does not change the assertion outcome.

**After**
- Keep minimal setup data required for the scenario.
- Call the relevant API endpoint used by real clients.
- Assert response shape and values (including `null` states).
- Remove unrelated setup that adds noise.

## Main Logic

1. **Flow tests represent real usage paths**  
   They should test request -> handler -> service -> repository -> response, not only repository state.

2. **Avoid assertion tautology**  
   "I inserted X, then I read X from DB" mostly proves the test setup, not product behavior.

3. **Keep tests focused (KISS)**  
   If a second fixture does not strengthen the assertion, remove it.

4. **Assert contract, not implementation details**  
   In frontend-oriented or API-consumer scenarios, response contract quality matters more than internal storage checks.

## Practical Review Checklist

- Is this test named as a flow/integration test?
- Does it call an actual API endpoint or user-facing interface?
- Does it verify response/output contract?
- Is there any setup fixture that does not affect assertions?
- Can this be simplified without losing coverage?

## Reusable PR Reply Template

```md
Thanks for the catch — agreed.

**Before:**
- [Old behavior]
- [Why it was weak/noisy]

**After:**
- [What endpoint/path is now exercised]
- [What contract is now asserted]
- [What noise was removed]

**Main logic:**
For a flow test, we should verify externally observable behavior rather than only internal state written during setup. This keeps tests realistic, focused, and maintainable.
```

