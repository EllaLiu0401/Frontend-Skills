# Defensive Button `type` and ARIA Role Opt-Out

> **TL;DR**: Always add `type="button"` to non-submit buttons. When using CSS-only UI patterns, opt out of ARIA roles rather than leaving half-wired semantics.

## Context

During a PR review for tab sub-components, two defensive coding issues were caught:
1. A `<button>` element missing `type="button"`, which could cause form submission
2. Tab content panels with `role="tabpanel"` but no accessible name, creating unnamed landmarks

Both are easy to miss and hard to debug in production.

## Issues & Fixes

### 1) Missing `type="button"` on Non-Submit Buttons

**Symptom**: A button-based tab component renders `<button>` without an explicit `type` attribute.

```tsx
// ❌ Before: missing type attribute
<button
  role="tab"
  aria-selected={active}
>
  Settings
</button>

// ✅ After: explicit type="button"
<button
  type="button"
  role="tab"
  aria-selected={active}
>
  Settings
</button>
```

**Root Cause**: The HTML spec defaults `<button>` to `type="submit"` when no type is specified. Inside a `<form>`, clicking any untyped button triggers form submission — even if it's a tab, accordion toggle, or modal close button.

**Fix**: Add `type="button"` to the element. Place it before `{...rest}` spread so consumers can still override if needed.

**Prevention**:
- Treat `type="button"` as mandatory for every `<button>` that isn't a form submit
- Check the entire codebase for consistency — if `Button`, `IconButton`, and `MenuItem` all set it, any new button component should too
- Add this to component review checklists

---

### 2) ARIA Roles Without Accessible Names

**Symptom**: A panel component always renders `role="tabpanel"` but is used in CSS-only demos where no `aria-labelledby` or `aria-label` is provided.

```tsx
// ❌ Before: role="tabpanel" with no accessible name
<div role="tabpanel" className="tab-content">
  Tab content 1
</div>

// ✅ Option A: opt out of ARIA for CSS-only usage
<div role={undefined} className="tab-content">
  Tab content 1
</div>

// ✅ Option B: provide proper accessible name for ARIA usage
<div role="tabpanel" aria-labelledby="tab-1" className="tab-content">
  Tab content 1
</div>
```

**Root Cause**: The component was designed for the ARIA tab pattern (where `labelledBy` wires everything up), but was also used in CSS-only DaisyUI-style demos where the radio-input pattern doesn't have tab IDs to reference. The `role="tabpanel"` was always applied regardless of context.

**Fix**: In CSS-only demo stories, pass `role={undefined}` to opt out of ARIA semantics. This matches the approach already taken on the container (`role={undefined}` on the tablist).

**Prevention**:
- When a component hardcodes an ARIA role, consider whether all usage contexts provide the required accessible name
- If a component serves both ARIA and non-ARIA patterns, make the role configurable or document which props are required for a11y compliance
- Run axe checks locally before pushing: unnamed roles are caught immediately

---

## Reusable Rules

1. **Every `<button>` needs an explicit `type`**: If it's not submitting a form, it's `type="button"`. No exceptions.
2. **ARIA roles require accessible names**: `role="tabpanel"` needs `aria-labelledby` or `aria-label`. `role="tab"` gets its name from content. Check every role.
3. **Don't half-wire ARIA**: Either go full ARIA with proper wiring, or opt out entirely with `role={undefined}`. A half-wired ARIA tree is worse than no ARIA at all — it actively misleads assistive technology.
4. **Match your pattern to your structure**: CSS sibling-selector patterns (`:checked + .content`) need content inside the container. ARIA patterns need panels outside the tablist. Mixing structures breaks one or both.

## Checklist for Component PRs

Before submitting a component PR for review:

- [ ] Every `<button>` has explicit `type="button"` (unless it's a submit button)
- [ ] Every ARIA role has a corresponding accessible name
- [ ] CSS-only demo patterns opt out of ARIA roles (`role={undefined}`)
- [ ] ARIA wiring is complete: `aria-controls` ↔ `id`, `aria-labelledby` ↔ `id`
- [ ] Verify against existing component patterns in the codebase for consistency
- [ ] Run `pnpm lint:fix` (or equivalent) before pushing
- [ ] Run axe / accessibility checks on stories locally

## Related Documentation

- [ARIA Tab Pattern Pitfalls](../accessibility/aria-tab-pattern-pitfalls.md)
- [How to Respond to PR Review Comments](pr-review-how-to-respond.md)
- [Custom Dropdown ARIA Patterns](../accessibility/custom-dropdown-aria-patterns.md)

---

**Source**: PR Review — Tab sub-components  
**Date**: April 2026  
**Topics**: Accessibility, Defensive Coding, ARIA, Buttons, Code Review
