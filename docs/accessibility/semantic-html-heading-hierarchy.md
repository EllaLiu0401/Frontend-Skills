# Semantic HTML: Heading Hierarchy Best Practices

## Overview

Proper heading hierarchy (`<h1>` through `<h6>`) is crucial for accessibility, SEO, and document structure. This guide explains why it matters and how to get it right.

## The Problem: Multiple H1 Elements

**Bad Example:**

```tsx
// Page component
<PageHeader title="Dashboard" /> {/* Renders <h1> */}

// Section within page
<section>
  <h1>Recent Activity</h1> {/* Second <h1>! */}
  <ActivityFeed />
</section>
```

**Why this is problematic:**

- ❌ Violates HTML semantic standards
- ❌ Confuses screen readers and assistive technology
- ❌ Breaks document outline structure
- ❌ Hurts SEO (search engines expect one main topic per page)
- ❌ Makes navigation harder for users with disabilities

## The Rule: One H1 Per Page

**A web page should have exactly one `<h1>` element** that represents the main topic or purpose of that page.

### Correct Heading Structure

```
<h1> Main Page Title </h1>           ← One per page
  <h2> Section Title </h2>           ← Major sections
    <h3> Subsection Title </h3>      ← Subsections
      <h4> Sub-subsection </h4>      ← Further divisions
  <h2> Another Section </h2>         ← Back to section level
    <h3> Another Subsection </h3>
```

## How Screen Readers Use Headings

Screen reader users rely on headings for:

1. **Quick Navigation** - Jump between sections using heading shortcuts
2. **Page Structure** - Understand content organization
3. **Content Scanning** - Get overview without reading everything
4. **Landmark Navigation** - Navigate complex pages efficiently

### Screen Reader Announcement Example

With proper hierarchy:

```
"Heading level 1: Dashboard"
"Heading level 2: Recent Activity"
"Heading level 2: Metrics"
"Heading level 3: Total Calls"
```

With broken hierarchy (multiple h1s):

```
"Heading level 1: Dashboard"
"Heading level 1: Recent Activity"  // Confusing!
"Heading level 1: Metrics"           // Where am I?
```

## Common Mistakes and Fixes

### Mistake 1: Multiple H1s

**❌ Wrong:**

```tsx
export default function DashboardPage() {
  return (
    <>
      <PageHeader title="Dashboard" /> {/* h1 */}
      <section>
        <h1>Recent Activity</h1> {/* Second h1! */}
      </section>
      <section>
        <h1>Metrics</h1> {/* Third h1! */}
      </section>
    </>
  );
}
```

**✅ Correct:**

```tsx
export default function DashboardPage() {
  return (
    <>
      <PageHeader title="Dashboard" /> {/* h1 */}
      <section>
        <h2>Recent Activity</h2> {/* h2 for sections */}
      </section>
      <section>
        <h2>Metrics</h2> {/* h2 for sections */}
      </section>
    </>
  );
}
```

### Mistake 2: Skipping Heading Levels

**❌ Wrong:**

```tsx
<h1>Main Title</h1>
<h4>Subsection</h4> {/* Skipped h2 and h3! */}
```

**✅ Correct:**

```tsx
<h1>Main Title</h1>
<h2>Section</h2>
<h3>Subsection</h3>
```

### Mistake 3: Using Headings for Styling

**❌ Wrong:**

```tsx
{
  /* Using h4 just because it looks the right size */
}
<h4 className="text-large">Important Text</h4>;
```

**✅ Correct:**

```tsx
{
  /* Use appropriate heading level, style with CSS */
}
<h2 className="text-sm">Section Title</h2>;
```

## React/Next.js Best Practices

### Pattern 1: Heading Component with Level Prop

```tsx
interface HeadingProps {
  level: 1 | 2 | 3 | 4 | 5 | 6;
  children: React.ReactNode;
  className?: string;
}

export function Heading({ level, children, className }: HeadingProps) {
  const Tag = `h${level}` as const;
  return <Tag className={className}>{children}</Tag>;
}

// Usage
<Heading level={1}>Main Title</Heading>
<Heading level={2} className="text-h4">Section Title</Heading>
```

### Pattern 2: Page Layout Pattern

```tsx
// app/dashboard/page.tsx
export default function DashboardPage() {
  return (
    <div>
      {/* Page h1 - only one per page */}
      <PageHeader title="Dashboard" />

      {/* Tab navigation doesn't create new h1s */}
      <TabNavigation>
        <TabPanel id="overview">
          <DashboardOverview /> {/* Uses h2 for sections */}
        </TabPanel>
        <TabPanel id="settings">
          <Settings /> {/* Uses h2 for sections */}
        </TabPanel>
      </TabNavigation>
    </div>
  );
}

// components/DashboardOverview.tsx
export function DashboardOverview() {
  return (
    <>
      {/* Section headings use h2 */}
      <section>
        <Heading level={2}>Recent Activity</Heading>
        <ActivityFeed />
      </section>

      <section>
        <Heading level={2}>Metrics</Heading>
        <MetricsGrid />
      </section>
    </>
  );
}
```

### Pattern 3: Reusable Section Component

```tsx
interface SectionProps {
  title: string;
  headingLevel?: 2 | 3 | 4; // No h1, that's for page title
  children: React.ReactNode;
}

export function Section({ title, headingLevel = 2, children }: SectionProps) {
  return (
    <section>
      <Heading level={headingLevel}>{title}</Heading>
      {children}
    </section>
  );
}

// Usage
<Section title="Recent Activity" headingLevel={2}>
  <ActivityFeed />
</Section>;
```

## Testing Heading Hierarchy

### Manual Testing

1. **Browser DevTools:**

   ```
   Open DevTools → Elements → Search for "h1"
   Verify: Only one h1 exists
   ```

2. **HeadingsMap Browser Extension:**
   - Shows visual hierarchy of all headings
   - Identifies skipped levels and multiple h1s

3. **Screen Reader Testing:**
   - macOS: VoiceOver (Cmd+F5)
   - Windows: NVDA (free) or JAWS
   - Navigate using heading shortcuts (H key)

### Automated Testing

```tsx
// Example test with Testing Library
describe("Dashboard Page Accessibility", () => {
  it("should have exactly one h1 element", () => {
    render(<DashboardPage />);

    const h1Elements = screen.getAllByRole("heading", { level: 1 });
    expect(h1Elements).toHaveLength(1);
    expect(h1Elements[0]).toHaveTextContent("Dashboard");
  });

  it("should have proper heading hierarchy", () => {
    render(<DashboardPage />);

    // Check section headings are h2
    const h2Elements = screen.getAllByRole("heading", { level: 2 });
    expect(h2Elements.length).toBeGreaterThan(0);

    // Verify no multiple h1s
    const headings = screen.getAllByRole("heading");
    const h1Count = headings.filter(
      (h) => h.tagName.toLowerCase() === "h1",
    ).length;
    expect(h1Count).toBe(1);
  });
});
```

## Code Review Checklist

When reviewing heading usage:

- [ ] Is there exactly one h1 per page?
- [ ] Do section headings use h2?
- [ ] Are heading levels sequential (no skipping)?
- [ ] Are headings used for structure, not styling?
- [ ] Do reusable components accept heading level props?
- [ ] Are tests verifying heading hierarchy?

## Common Questions

**Q: Can I style h1 to look like h3?**  
A: Yes! Use semantic HTML for structure, CSS for appearance.

```tsx
<h1 className="text-sm">Page Title</h1> {/* Looks small but semantically correct */}
```

**Q: What about modal dialogs?**  
A: Modal content should continue the page's heading hierarchy. If it's a major modal, use h2 for the modal title.

**Q: What about tabs/accordions?**  
A: Tab/accordion content should use h2 for section titles, not h1. They're part of the page, not separate pages.

**Q: Should I use ARIA instead?**  
A: Use semantic HTML first. ARIA is for when semantic HTML isn't enough. Don't use `<div role="heading" aria-level="1">` when you can use `<h1>`.

## WCAG Guidelines

This relates to:

- **WCAG 2.1 Success Criterion 1.3.1** (Info and Relationships - Level A)
- **WCAG 2.1 Success Criterion 2.4.6** (Headings and Labels - Level AA)

## Key Takeaways

1. **One h1 per page** - represents main topic
2. **Use h2 for major sections** - not h1
3. **Don't skip levels** - h1 → h2 → h3, not h1 → h4
4. **Semantic structure ≠ visual style** - Use CSS for appearance
5. **Think document outline** - Headings create page structure
6. **Test with screen readers** - Experience what users experience

## Resources

- [MDN: HTML Heading Elements](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/Heading_Elements)
- [WebAIM: Semantic Structure](https://webaim.org/techniques/semanticstructure/)
- [WCAG 2.1 Guidelines](https://www.w3.org/WAI/WCAG21/quickref/)
- [HeadingsMap Browser Extension](https://chromewebstore.google.com/detail/headingsmap)
