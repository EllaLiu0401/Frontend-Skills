# Comprehensive i18n Audit and Implementation

## Overview

A systematic approach to ensuring complete internationalization coverage across a frontend application, with focus on identifying and fixing hardcoded strings.

**Key Learning**: i18n is not just about translating text - it's about creating a maintainable, globally-accessible application from the start.

---

## The Problem

When building features, it's easy to introduce hardcoded strings that break internationalization:

```tsx
// ❌ Bad: Hardcoded strings scattered throughout
function ActivityFeed() {
  return (
    <div>
      <p>Just now</p>
      <p>2 hours ago</p>
      <button aria-label="Back to dashboard">←</button>
    </div>
  );
}
```

**Common sources of hardcoded text:**

- Time/date formatting strings
- ARIA labels for accessibility
- Placeholder text ("Coming soon", "Loading...")
- Error messages
- Button labels and tooltips

---

## Systematic Audit Process

### 1. Inventory All Files

Use glob patterns to find all relevant files:

```bash
# Find all component files in a feature
find ./src/components/dashboard -name "*.tsx"

# Or use glob patterns
**/dashboard/**/*.tsx
```

### 2. Search for Hardcoded Patterns

Use grep/ripgrep to find common hardcoded patterns:

```bash
# Search for common English phrases
rg "['\"](Just now|ago|Back to|Coming soon|Loading)" --type tsx

# Search for hardcoded time references
rg "['\"]\d+\s+(min|hour|day)s?\s+ago" --type tsx

# Find potential aria-labels
rg 'aria-label="[^{]' --type tsx
```

### 3. Check Translation Hook Usage

Verify files are using i18n:

```bash
# Find files using translations
rg "useTranslations|getTranslations" --type tsx

# Find files WITHOUT translations (potential issues)
rg --files-without-match "useTranslations|getTranslations" --type tsx
```

---

## Implementation Patterns

### Pattern 1: Time Formatting with i18n

**Before:**

```tsx
function formatRelativeTime(timestamp: string): string {
  const diffMins = Math.floor(diffMs / 60000);

  if (diffMins < 1) return "Just now";
  if (diffMins < 60) return `${diffMins} min ago`;
  return date.toLocaleDateString();
}
```

**After:**

```tsx
function ActivityFeed() {
  const t = useTranslations("activity.time");

  function formatRelativeTime(timestamp: string): string {
    const diffMins = Math.floor(diffMs / 60000);

    if (diffMins < 1) return t("justNow");
    if (diffMins < 60) return t("minutesAgo", { count: diffMins });
    return date.toLocaleDateString();
  }

  // ... rest of component
}
```

**Translation files:**

```json
// en.json
{
  "activity": {
    "time": {
      "justNow": "Just now",
      "minutesAgo": "{count} min ago",
      "hoursAgo": "{count} hr ago",
      "daysAgo": "{count} day ago",
      "daysAgoPlural": "{count} days ago"
    }
  }
}

// es.json
{
  "activity": {
    "time": {
      "justNow": "Justo ahora",
      "minutesAgo": "hace {count} min",
      "hoursAgo": "hace {count} hr",
      "daysAgo": "hace {count} día",
      "daysAgoPlural": "hace {count} días"
    }
  }
}
```

**Key points:**

- Move formatting function INSIDE component to access `t()`
- Use interpolation for dynamic values: `{count}`
- Handle pluralization with separate keys
- Locale-aware date formatting for fallback

---

### Pattern 2: Accessibility Labels

**Before:**

```tsx
<button aria-label="Back to dashboard">
  <Icon name="ArrowLeft" />
</button>
```

**After:**

```tsx
"use client";

function PlaceholderLayout() {
  const t = useTranslations("layout");

  return (
    <button aria-label={t("backToDashboard")}>
      <Icon name="ArrowLeft" />
    </button>
  );
}
```

**Important:** If a Server Component needs i18n, it may need to become a Client Component ('use client').

---

### Pattern 3: Placeholder/Description Text

**Before:**

```tsx
export default function ComingSoonPage() {
  return (
    <div>
      <p>Full feature list coming soon. This page will display...</p>
    </div>
  );
}
```

**After:**

```tsx
export default function ComingSoonPage() {
  const t = useTranslations("pages.comingSoon");

  return (
    <div>
      <p>{t("description")}</p>
    </div>
  );
}
```

---

## Translation File Organization

### Hierarchical Structure

Organize keys to match component hierarchy:

```json
{
  "dashboard": {
    "overview": {
      "title": "Dashboard Overview",
      "activity": {
        "title": "Recent Activity",
        "showMore": "Show More",
        "empty": "No recent activity",
        "time": {
          "justNow": "Just now",
          "minutesAgo": "{count} min ago"
        }
      },
      "metrics": {
        "callsMade": "Calls Made",
        "totalCost": "Total Cost"
      }
    }
  }
}
```

### Benefits of Hierarchical Keys:

1. **Scoped translations**: `useTranslations('dashboard.overview.activity')`
2. **Clear ownership**: Easy to find which component owns which text
3. **Avoids naming collisions**: `dashboard.title` vs `settings.title`
4. **Easier refactoring**: Move sections together

---

## Client vs Server Components

### Client Components (Browser)

```tsx
"use client";

import { useTranslations } from "next-intl";

export function ClientComponent() {
  const t = useTranslations("namespace");

  return <div>{t("key")}</div>;
}
```

### Server Components (Server-side)

```tsx
import { getTranslations } from "next-intl/server";

export default async function ServerComponent() {
  const t = await getTranslations("namespace");

  return <div>{t("key")}</div>;
}
```

**When to use which:**

- **Server Component**: Default, better performance, no hydration
- **Client Component**: When you need state, effects, or browser APIs

---

## Quality Checklist

Before submitting code, check:

### ✅ All Text Internationalized

- [ ] No hardcoded strings in JSX
- [ ] All aria-labels use i18n
- [ ] Error messages internationalized
- [ ] Time/date formatting uses i18n
- [ ] Placeholder text internationalized

### ✅ All Locales Updated

- [ ] Added keys to ALL locale files (en, es, etc.)
- [ ] Tested in multiple languages
- [ ] Plural forms handled correctly

### ✅ Proper Hook Usage

- [ ] `useTranslations()` in Client Components
- [ ] `getTranslations()` in Server Components
- [ ] Translation namespace scoped appropriately

### ✅ Dynamic Values Handled

- [ ] Used interpolation: `t('key', { value })`
- [ ] Separate keys for plurals when needed
- [ ] Locale-aware number/date formatting

---

## Adding to Project Rules

**Best Practice**: Document i18n requirements in your project's coding standards:

```markdown
## Internationalization (i18n)

• MANDATORY: ALL user-facing text MUST use i18n - NO hardcoded strings
• Use `useTranslations()` in Client Components, `getTranslations()` in Server Components
• Translation files: `src/locales/{locale}/common.json`
• Structure keys with dot notation: `dashboard.overview.activity.title`
• Dynamic values: `t('key', { variableName: value })`
• Add translations to ALL locales when adding features
• Before submitting: grep for hardcoded patterns like 'Back to', 'Just now'
```

---

## Common Pitfalls

### ❌ Pitfall 1: Forgetting Accessibility Text

```tsx
// Wrong - hardcoded aria-label
<button aria-label="Close dialog">×</button>

// Correct
<button aria-label={t('actions.close')}>×</button>
```

### ❌ Pitfall 2: Hardcoded Time Strings

```tsx
// Wrong
return `${hours} hours ago`;

// Correct
return t("time.hoursAgo", { count: hours });
```

### ❌ Pitfall 3: Only Updating One Locale

```tsx
// Wrong - only added to en.json
// Other locales (es.json, fr.json) missing the key

// Correct - add to ALL locale files
// en.json: "newFeature": "New Feature"
// es.json: "newFeature": "Nueva Función"
// fr.json: "newFeature": "Nouvelle Fonctionnalité"
```

### ❌ Pitfall 4: Server Component Without 'use client'

```tsx
// Wrong - using useTranslations in Server Component
export default function ServerPage() {
  const t = useTranslations("page"); // Error!
}

// Option 1: Use getTranslations
export default async function ServerPage() {
  const t = await getTranslations("page");
}

// Option 2: Convert to Client Component
("use client");
export default function ClientPage() {
  const t = useTranslations("page");
}
```

---

## Tools and Commands

### Find Hardcoded Strings

```bash
# Regex patterns to search for
grep -r "['\"](Just now|ago|Coming soon)" src/

# Using ripgrep (faster)
rg "return ['\"][A-Z]" --type tsx
rg "aria-label=\"[^{]" --type tsx
```

### Verify Translation Coverage

```bash
# Find components without i18n
rg --files-without-match "useTranslations|getTranslations" src/components/

# Check for missing translations in locale files
diff <(jq -r 'keys[]' locales/en.json) <(jq -r 'keys[]' locales/es.json)
```

---

## Summary

**Key Takeaways:**

1. **Audit systematically**: Use glob + grep to find all hardcoded strings
2. **Move formatting inside components**: Access translation functions
3. **Organize hierarchically**: Match translation keys to component structure
4. **Update all locales**: Never leave translations incomplete
5. **Document in rules**: Make i18n requirements explicit and enforceable
6. **Check accessibility**: aria-labels need i18n too
7. **Use appropriate hooks**: `useTranslations()` vs `getTranslations()`

**The 5-Minute i18n Check:**

```bash
# Before submitting any PR:
rg "['\"](Back to|Just now|ago|Coming soon|Loading)" src/
rg 'aria-label="[^{]' src/
rg --files-without-match "useTranslations|getTranslations" src/components/
```

Remember: **i18n is not just translation - it's about building maintainable, globally-accessible software from day one.**
