# How to Respond to PR Review Comments

A practical guide for responding to code review feedback — covering analysis, categorization, and writing effective replies.

---

## Step 1: Categorize Every Comment Before Responding

Not every comment is a real bug. Before writing a single reply, categorize each one:

| Category | Definition | Action |
|---|---|---|
| **Bug** | Code behaves incorrectly in a real scenario | Fix it |
| **ARIA / Spec violation** | Breaks an official spec (ARIA, WCAG, HTML) | Fix it |
| **Edge case** | Works for 99% of cases, breaks at a boundary | Fix if the boundary is realistic |
| **Enhancement** | Valid improvement, but not in scope | Acknowledge, track as follow-up |
| **Over-engineering** | Valid but adds complexity for no concrete need | Decline with reasoning |
| **Wrong assumption** | Reviewer misunderstood the code | Explain clearly, no code change |
| **Style / Nitpick** | Preference, not correctness | Team decision |

**Key principle:** Only fix what is actually broken or violates a spec. Avoid gold-plating or fixing things that "might be a problem someday."

---

## Step 2: Verify Before Claiming Something Works

For "not a bug" responses, verify your claim before replying:

- Check the actual library version (e.g., is `bg-(--var)` actually valid in Tailwind v4?)
- Check related files (e.g., are different opacity tokens used for hover vs active?)
- Check the spec (e.g., is `role="combobox"` actually needed for `aria-activedescendant`?)
- Check the flow (e.g., does `onBlur` really fire correctly after a click-outside?)

A confident reply backed by evidence is far more persuasive than "I think it's fine."

---

## Step 3: Write the Reply

### For fixed bugs

```
**Fixed.**

**Before:** [brief description of the broken state]
**After:** [brief description of the fix]

[One sentence explaining why this matters, if not obvious]
```

### For intentional decisions

```
**Intentional — no change needed.**

[Explain the design rationale in 2-3 sentences]
[Reference a standard, spec, or consistent pattern in the codebase if applicable]
```

### For out-of-scope enhancements

```
**Acknowledged — follow-up enhancement.**

[Confirm you understand the improvement]
[Explain why it's not in scope for this PR — scope, consistency, complexity]
[State you'll track it as a follow-up item]
```

### For wrong assumptions

```
**Not a bug — [one-liner summary]**

[Explain how it actually works, concisely]
[Reference a token value, spec link, or code path to back it up]
```

---

## Step 4: Handle Conflicting Reviewer Opinions

Sometimes two reviewers flag the same thing with opposite conclusions (e.g., "remove `role="combobox"`" vs "you need `role="combobox"` for `aria-activedescendant`").

1. **Go to the spec first.** ARIA APG examples are authoritative for accessibility patterns.
2. **Trace the actual code flow** to see which reviewer is correct.
3. If you made a change and then need to revert it, update your original reply:
   - Add a strikethrough on the old reply: `~~Fixed.~~`
   - Explain the revert concisely
4. Post the corrected code.

---

## Common PR Review Anti-Patterns to Avoid

### As an author:

- Immediately agreeing to changes without verifying they're correct
- Fixing things that work fine (makes reviewers think you weren't confident in your own code)
- Leaving comments unanswered
- Closing issues with vague "acknowledged" without explanation
- Making scope-creep changes under review pressure

### As a reviewer:

- Flagging valid v4 syntax as an error without checking the version
- Treating progressive enhancements as blocking bugs
- Mixing personal preference with spec violations (different severity)

---

## Quick Reference: When to Fix vs. Decline

| Situation | Fix? |
|---|---|
| ARIA spec violation (confirmed) | Yes |
| WCAG AA compliance failure | Yes |
| Real runtime bug (proven by tracing code) | Yes |
| Edge case that affects documented use cases | Yes |
| Touch target below 44×44px | Yes |
| Enhancement "for future flexibility" | No — YAGNI |
| Adding a dependency for one component | No — consistency first |
| Refactoring that doesn't change behavior | No — separate PR |
| Style preference with no spec backing | No — team convention, not this PR |
