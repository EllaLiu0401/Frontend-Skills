# ARIA Tab Pattern: Common Pitfalls

> **TL;DR**: Tabs are deceptively simple — most a11y bugs come from mismatched ARIA ownership, unnamed tabpanels, or mixing CSS-only patterns with ARIA semantics.

## 1. Tablist Can Only Own Tab Elements

Per [WAI-ARIA 1.2 §tablist](https://www.w3.org/TR/wai-aria-1.2/#tablist), the **required owned elements** of `role="tablist"` are exclusively elements with `role="tab"`. Placing a `role="tabpanel"` inside a tablist is an ownership violation.

```tsx
// ❌ Wrong: tabpanel is a child of tablist
<div role="tablist">
  <button role="tab">Tab 1</button>
  <button role="tab">Tab 2</button>
  <div role="tabpanel">Panel 1</div>  // violation
  <div role="tabpanel">Panel 2</div>  // violation
</div>

// ✅ Correct: tabpanels are siblings of the tablist
<div role="tablist">
  <button role="tab" aria-controls="panel-1">Tab 1</button>
  <button role="tab" aria-controls="panel-2">Tab 2</button>
</div>
<div role="tabpanel" id="panel-1" aria-labelledby="tab-1">Panel 1</div>
<div role="tabpanel" id="panel-2" aria-labelledby="tab-2">Panel 2</div>
```

**Impact**: Screen readers (NVDA, JAWS, VoiceOver) use the ARIA ownership model to navigate tab interfaces. Invalid ownership causes panels to be announced incorrectly or skipped entirely.

---

## 2. Every Tabpanel Must Have an Accessible Name

If you use `role="tabpanel"`, it **must** have an accessible name — either `aria-labelledby` pointing to its controlling tab, or `aria-label`. An unnamed tabpanel fails the axe rule "ARIA roles must have accessible names."

```tsx
// ❌ Wrong: unnamed tabpanel
<div role="tabpanel" className="tab-content">
  Content here
</div>

// ✅ Option A: aria-labelledby (preferred — connects tab and panel)
<div role="tabpanel" id="panel-1" aria-labelledby="tab-1">
  Content here
</div>

// ✅ Option B: aria-label (when no tab element to reference)
<div role="tabpanel" aria-label="Settings">
  Content here
</div>
```

---

## 3. CSS-Only Tab Patterns vs. ARIA Tabs

Some CSS frameworks (e.g., DaisyUI) use a **CSS-only tab pattern** where radio inputs and content panels are siblings inside the same container. The visibility toggle relies on `input:checked + .tab-content` CSS selectors.

This CSS pattern is structurally incompatible with ARIA tablist semantics:

| Concern | ARIA Tab Pattern | CSS-Only Pattern |
|---|---|---|
| Container role | `tablist` | None (just a div) |
| Tab trigger | `<button role="tab">` | `<input type="radio">` |
| Panel location | Sibling of tablist | Inside same container (for CSS selectors) |
| Visibility | Managed via JS state | Managed via `:checked` selector |

**Rule**: When using a CSS-only tab pattern, **opt out of ARIA semantics entirely**. Don't add `role="tablist"` to the container or `role="tabpanel"` to the content — it creates an invalid ARIA tree.

```tsx
// ❌ Wrong: mixing CSS-only pattern with ARIA roles
<div role="tablist">
  <input type="radio" name="tabs" className="tab" />
  <div role="tabpanel" className="tab-content">Content</div>
</div>

// ✅ Correct: opt out of ARIA for CSS-only pattern
<div>
  <input type="radio" name="tabs" className="tab" aria-label="Tab 1" />
  <div className="tab-content">Content</div>
</div>
```

---

## 4. ARIA Wiring Checklist for Button-Based Tabs

When building interactive (JS-managed) tabs with proper ARIA:

```tsx
// Complete ARIA wiring example
<div role="tablist">
  <button
    role="tab"
    type="button"
    id="tab-1"
    aria-selected={activeTab === 'tab-1'}
    aria-controls="panel-1"
    tabIndex={activeTab === 'tab-1' ? 0 : -1}
  >
    Tab 1
  </button>
</div>

<div
  role="tabpanel"
  id="panel-1"
  aria-labelledby="tab-1"
  tabIndex={0}
  hidden={activeTab !== 'tab-1'}
>
  Panel 1 content
</div>
```

### Required attributes:

| Element | Attribute | Purpose |
|---|---|---|
| Tab container | `role="tablist"` | Declares the tab group |
| Tab button | `role="tab"` | Declares a tab trigger |
| Tab button | `type="button"` | Prevents form submission |
| Tab button | `aria-selected="true/false"` | Active state |
| Tab button | `aria-controls="panel-id"` | Links tab to its panel |
| Tab button | `tabIndex={0 or -1}` | Roving tabindex |
| Tab panel | `role="tabpanel"` | Declares a panel |
| Tab panel | `aria-labelledby="tab-id"` | Accessible name |
| Tab panel | `tabIndex={0}` | Makes panel focusable (for empty panels) |

---

## 5. Key Takeaway: Two Valid Patterns, Don't Mix Them

| Pattern | When to use | ARIA roles? |
|---|---|---|
| **JS-managed tabs** | Interactive apps, SPAs | Yes — full ARIA wiring |
| **CSS-only tabs** | Static demos, framework patterns | No — opt out of ARIA |

The biggest mistake is using ARIA roles from one pattern while structuring DOM for the other. Pick one and be consistent.

---

**Source**: PR Review — Tab sub-components with a11y support  
**Date**: April 2026  
**Topics**: ARIA, Accessibility, Tabs, WAI-ARIA 1.2, axe-core
