# PR Comment Lesson: CTA Button Spacing & Visual Hierarchy

## Context

A reviewer pointed out that a primary call-to-action (CTA) button ("Accept & Continue") sat too close to a secondary action ("Log Out"), which could pull the user's focus away from the intended primary action.

## Core Lesson

**Primary CTA buttons need generous spacing** to stand out from surrounding elements — especially from secondary or destructive actions. Proximity implies relationship; insufficient spacing makes actions feel equal in importance.

## The Gestalt Principle at Play

The **Law of Proximity** states that elements placed close together are perceived as a group. When a primary button and a secondary link are visually grouped, users may hesitate or misclick.

## Before / After Thinking Pattern

- Before: "Both actions are in the form area, standard gap is fine."
- After: "The primary CTA needs visual breathing room to reinforce its importance. Increase spacing above it to separate it from secondary actions."

## Practical Pattern

```tsx
// ❌ Before: checkbox and submit button share the same tight container
<form className="flex flex-col gap-4">
  <div className="flex flex-col gap-3">
    <label>
      <input type="checkbox" /> I agree to the terms
    </label>
    <button type="submit">Accept & Continue</button>
  </div>
</form>
<a href="/logout">Log Out</a>

// ✅ After: submit button is a direct child of the form with larger gap
<form className="flex flex-col gap-6">
  <label>
    <input type="checkbox" /> I agree to the terms
  </label>
  <button type="submit">Accept & Continue</button>
</form>
<a href="/logout">Log Out</a>
```

Key changes:

1. **Remove unnecessary wrapper** around checkbox + button — flatten the structure.
2. **Increase the gap** (e.g., `gap-4` → `gap-6`) so the CTA has more space above it.
3. The outer layout already provides separation between the form and secondary links.

## Why This Matters

- **Conversion**: A clearly isolated CTA has higher click-through rates.
- **Accessibility**: Users with motor impairments benefit from well-separated tap targets.
- **Cognitive load**: Visual grouping tells users "these belong together" — wrong grouping causes confusion.
- **Mobile UX**: On smaller screens, spacing issues are amplified; thumbs need clear targets.

## Quick Decision Checklist

- Is the primary CTA visually dominant (size, color, spacing)?
- Is there enough whitespace separating primary from secondary/destructive actions?
- Could a user mistake which button is the "main" action?
- On mobile viewports, are the actions still clearly distinct?

## General Spacing Guidelines for Action Areas

| Element Relationship               | Recommended Spacing                      |
| ---------------------------------- | ---------------------------------------- |
| Form fields within a group         | `gap-3` to `gap-4` (tight)               |
| Field group → Primary CTA          | `gap-6` to `gap-8` (generous)            |
| Primary CTA → Secondary action     | `gap-4` to `gap-6` (clear separation)    |
| Destructive action → Other actions | Maximum separation, often at page bottom |

## One-Line Rule

Give primary CTA buttons more vertical space than other form elements — spacing communicates hierarchy.

---

**Source**: PR Review Comment  
**Date**: February 2026  
**Topics**: UX, Visual Hierarchy, Spacing, CTA Design, Gestalt Principles
