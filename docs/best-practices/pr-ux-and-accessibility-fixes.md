# PR Learning: UX & Accessibility Fixes

## TL;DR

Two common issues that appear innocuous but significantly impact user experience and accessibility:

1. **Empty onClick handlers** confuse users and damage trust
2. **Multiple h1 elements** break screen reader navigation and document structure

## Issues & Fixes

### 1) Empty onClick Handler with Mock Data

**Symptom:**

- Button displays notification badge count (e.g., "3 notifications")
- Clicking the button does nothing
- User confusion: "Why isn't it working?"

**Root Cause:**

- Feature not yet implemented
- Mock data displayed to preview UI
- Empty onClick handler as placeholder

**Fix:**
Remove the incomplete feature entirely:

```tsx
// Before (Bad)
const mockNotificationCount = 3;
<NotificationBell
  count={mockNotificationCount}
  onClick={() => {
    // TODO: Implement in Sprint 2
  }}
/>;

// After (Good)
// TODO: Add NotificationBell in Sprint 2 when API is ready
// Component temporarily removed to avoid user confusion
```

**Prevention:**

- Don't show UI elements that don't work
- Remove mock data that could mislead users
- If element must be visible, disable it with clear tooltip
- Follow KISS principle: simplest solution = don't show it

**Alternative Approaches:**

```tsx
// Option 1: Disable with tooltip
<Tooltip content="Coming soon">
  <NotificationBell disabled count={0} />
</Tooltip>

// Option 2: Show informative message
<NotificationBell
  count={0}
  onClick={() => toast.info('Feature launching soon!')}
/>

// Option 3 (Best): Don't show it at all
{isFeatureReady && <NotificationBell {...props} />}
```

**Files:**

- Component with empty handler
- Page/layout rendering the component

### 2) Multiple H1 Elements Breaking Accessibility

**Symptom:**

- Page has `<PageHeader>` rendering an h1
- Section within page also uses `<h1>`
- Multiple h1 elements on same page

**Root Cause:**

- Misunderstanding of HTML heading hierarchy
- Component using wrong heading level for visual consistency
- Not considering screen reader navigation

**Fix:**
Use h2 for section headings:

```tsx
// Before (Bad)
export default function DashboardPage() {
  return (
    <>
      <PageHeader title="Dashboard" /> {/* h1 */}
      <section>
        <Heading level={1}>Recent Activity</Heading> {/* h1 - Wrong! */}
      </section>
    </>
  );
}

// After (Good)
export default function DashboardPage() {
  return (
    <>
      <PageHeader title="Dashboard" /> {/* h1 */}
      <section>
        <Heading level={2}>Recent Activity</Heading> {/* h2 - Correct! */}
      </section>
    </>
  );
}
```

**Correct Heading Structure:**

```
<h1> Page Title </h1>           ← One per page
  <h2> Section Title </h2>      ← Major sections
    <h3> Subsection </h3>       ← Subsections
  <h2> Another Section </h2>    ← Back to section level
```

**Prevention:**

- Understand HTML heading hierarchy
- One h1 per page (the main page title)
- Use h2 for major sections
- Don't skip heading levels (h1 → h4)
- Use CSS for visual styling, not wrong heading levels
- Test with screen reader or HeadingsMap extension

**Why It Matters:**

- Screen readers use headings for page navigation
- Users jump between sections using heading shortcuts
- Improper hierarchy confuses document structure
- Affects SEO (search engines expect semantic structure)
- WCAG 2.1 compliance requirement

**Testing:**

```tsx
describe("Dashboard Accessibility", () => {
  it("should have exactly one h1", () => {
    render(<DashboardPage />);
    const h1s = screen.getAllByRole("heading", { level: 1 });
    expect(h1s).toHaveLength(1);
  });

  it("should use h2 for section headings", () => {
    render(<DashboardPage />);
    const heading = screen.getByText("Recent Activity");
    expect(heading.tagName).toBe("H2");
  });
});
```

**Files:**

- Component using wrong heading level
- Tests asserting heading levels

## Reusable Rules

### UX Rules

1. **No empty onClick handlers** - If it doesn't work, don't show it
2. **No mock data in production** - Misleads users about functionality
3. **Progressive disclosure** - Only show features when they're ready
4. **Provide feedback** - Never leave users wondering if something worked
5. **KISS principle** - Simplest solution for incomplete features = remove them

### Accessibility Rules

1. **One h1 per page** - Represents the main topic/purpose
2. **h2 for sections** - Major page sections use h2, not h1
3. **Sequential heading levels** - Don't skip levels (h1 → h2 → h3)
4. **Semantic structure ≠ visual style** - Use CSS for appearance
5. **Test with screen readers** - Verify navigation works correctly

### Code Review Checks

- [ ] Any onClick handlers that do nothing?
- [ ] Any mock data being displayed as real?
- [ ] Multiple h1 elements on the page?
- [ ] Heading levels skip (h1 → h4)?
- [ ] Tests verify heading structure?

## Impact

**UX Impact:**

- 🔴 **Critical**: Empty handlers destroy user trust
- Users expect visual elements to work
- Silent failures are worse than errors
- Mock data misleads users about product functionality

**Accessibility Impact:**

- 🔴 **Critical**: Improper headings break navigation
- Screen reader users rely on heading structure
- Multiple h1s confuse document outline
- Affects 15%+ of users (disability statistics)
- Legal compliance requirement (ADA, WCAG)

## Key Takeaways

1. **Empty onClick handlers are user-facing bugs**, not just incomplete code
2. **HTML semantics matter** - they're not optional or stylistic
3. **Think about all users** - including those using assistive technology
4. **Test early** - Check UX and accessibility during development
5. **When in doubt, remove** - Better to not show incomplete features

## Related Documentation

- [Handling Incomplete UI Features](./handling-incomplete-ui-features.md)
- [Semantic HTML Heading Hierarchy](../accessibility/semantic-html-heading-hierarchy.md)
- [PR Review Checklist](../quick-reference/pr-review-checklist.md)

## References

- WCAG 2.1 Success Criterion 1.3.1 (Info and Relationships)
- WCAG 2.1 Success Criterion 2.4.6 (Headings and Labels)
- [MDN: HTML Heading Elements](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/Heading_Elements)
- [WebAIM: Semantic Structure](https://webaim.org/techniques/semanticstructure/)
