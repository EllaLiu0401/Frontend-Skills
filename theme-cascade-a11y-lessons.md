# Theme Cascade and A11y Lessons

## What Happened

A visual/accessibility issue appeared in a page that was not directly edited.  
The root cause was global CSS and theme overrides affecting shared components in another screen.

## Key Lessons

1. Global CSS changes can impact unrelated pages
- If a component style class is shared, any global style change can affect all pages/stories using it.
- Failure locations in CI often show where the issue is detected first, not where it was introduced.

2. Nested theme containers create cascade traps
- In development tools and apps, theme wrappers can be nested (outer light, inner dark, etc.).
- Both light and dark selectors may match at once.
- If rules rely on selector order and `!important`, the wrong theme color can win.

3. Use token-driven theming, not selector battles
- Put theme differences in tokens (light token values vs dark token values).
- Keep component classes simple: they should consume tokens, not hardcoded theme-specific colors.
- This is more stable than repeating many light/dark override blocks.

4. Accessibility checks validate computed styles
- A11y tools evaluate the final rendered color pair (foreground/background), not your intent.
- Always verify final computed contrast ratio in both light and dark themes.

5. Prefer minimal, architecture-aligned fixes
- First patch quickly to stop regressions.
- Then refactor to a cleaner token model if cascade complexity remains.
- Remove duplicate overrides once token mapping is in place.

## Practical Rules to Follow

- Avoid global component color overrides with broad selectors.
- Avoid stacking many `!important` overrides across the file.
- Scope theme rules carefully, but prefer token mapping over deep selectors.
- Keep one source of truth for state colors (info/success/warning/error).
- Let children inherit color unless there is a clear reason not to.

## Verification Checklist (Before PR)

- Run accessibility tests in all theme/breakpoint combinations used by CI.
- Manually inspect contrast-critical components (alerts, badges, links, buttons).
- Confirm no regressions in both isolated component stories and aggregate demo pages.
- Hard refresh after CSS changes to avoid stale devtool/storybook cache confusion.

## Short Mental Model

Theme architecture should be:

`Primitives -> Semantic/Theme Tokens -> Component Classes`

When this chain is respected, cross-theme behavior is predictable and easier to maintain.
