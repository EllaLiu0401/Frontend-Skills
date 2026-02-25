# Vitest Mock Completeness — Keep Mocks in Sync with Imports

## What Happened

All tests in a component test file failed with a single error:

```
[vitest] No "someExport" export is defined on the "some-module" mock.
Did you forget to return it from "vi.mock"?
```

The component had been updated to import a new function from an already-mocked module, but the `vi.mock()` block in the test file was never updated to include the new export.

## Root Cause

When you use `vi.mock('some-module', () => ({ ... }))` with an explicit factory, Vitest replaces **the entire module** with only what the factory returns. Any export the real module provides but the factory omits simply does not exist. If the component under test calls that missing export, Vitest throws at runtime and **every** test in the file fails — not just tests that exercise the new code path.

## Key Lessons

1. **Explicit mock factories are exhaustive replacements**
   - `vi.mock('mod', () => ({ a: vi.fn() }))` means the mock module has *only* `a`. If the real module also exports `b` and the component calls `b`, every test crashes.
   - This is different from Jest's `jest.mock` with `__esModule: true` where missing keys silently return `undefined`.

2. **One missing mock export breaks all tests, not just some**
   - Because the error is thrown during component render (module resolution time), it poisons every `render()` call. The error message points to the call site inside the component, not the test.

3. **Use `importOriginal` when you only need to override a few exports**
   - The safest pattern for partial mocks:

   ```typescript
   vi.mock('some-module', async (importOriginal) => {
     const actual = await importOriginal();
     return {
       ...actual,
       onlyThingToMock: vi.fn(),
     };
   });
   ```

   - This way, any new export added to the real module is automatically available in tests without updating the mock.

4. **Explicit factories are still fine when you want total control**
   - For modules where you want every call captured (e.g., API client actions), listing each export explicitly is intentional. Just remember to update the factory whenever the component starts using a new export.

## Practical Rules to Follow

- **When adding an import to a component**, search the test file for `vi.mock('that-module')` and add the new export to the factory.
- **Prefer `importOriginal` + spread** for large modules where you only mock a few things. This is more resilient to future changes.
- **Keep explicit factories** for small, focused modules (e.g., a set of API actions) where you want to be deliberate about what's mocked.
- **Read the error message carefully** — Vitest tells you exactly which export is missing and which module mock is incomplete.

## Verification Checklist (Before PR)

- If you added a new import to a component, check if the module is mocked in the test file.
- If you added a new export to a shared module, grep for `vi.mock('that-module')` across all test files.
- Run the full test file (not just the new test) to catch mock-completeness issues early.

## Short Mental Model

```
Component adds new import ─► Is the module mocked in the test?
                                │
                         Yes ───┤──► Is the new export in the mock factory?
                                │         │
                         No  ───┘    No ──► Add it (or switch to importOriginal + spread)
                                     Yes ─► You're good
```
