# i18n Quick Reference Checklist

## Before Submitting Any PR

### 🔍 Quick Audit Commands

```bash
# Find hardcoded strings
rg "['\"](Just now|ago|Back to|Coming soon|Loading)" src/ --type tsx

# Find hardcoded aria-labels
rg 'aria-label="[^{]' src/ --type tsx

# Find components without i18n
rg --files-without-match "useTranslations|getTranslations" src/components/
```

---

## Implementation Quick Reference

### Client Component

```tsx
"use client";
import { useTranslations } from "next-intl";

export function MyComponent() {
  const t = useTranslations("namespace");
  return <div>{t("key")}</div>;
}
```

### Server Component

```tsx
import { getTranslations } from "next-intl/server";

export default async function MyPage() {
  const t = await getTranslations("namespace");
  return <div>{t("key")}</div>;
}
```

### With Dynamic Values

```tsx
// Component
t('message', { name: userName, count: 5 })

// Translation file
{
  "message": "Hello {name}, you have {count} items"
}
```

### Pluralization

```tsx
// Component
const key = count === 1 ? 'itemSingular' : 'itemPlural';
t(key, { count })

// Translation file
{
  "itemSingular": "{count} item",
  "itemPlural": "{count} items"
}
```

---

## Common Fixes

### ❌ → ✅ Time Formatting

```tsx
// ❌ Before
return "Just now";
return `${mins} min ago`;

// ✅ After
return t("time.justNow");
return t("time.minutesAgo", { count: mins });
```

### ❌ → ✅ Aria Labels

```tsx
// ❌ Before
<button aria-label="Close">×</button>

// ✅ After
<button aria-label={t('actions.close')}>×</button>
```

### ❌ → ✅ Description Text

```tsx
// ❌ Before
<p>Coming soon. This page will display...</p>

// ✅ After
<p>{t('comingSoon.description')}</p>
```

---

## Translation File Structure

```json
{
  "feature": {
    "section": {
      "title": "Section Title",
      "description": "Section description",
      "actions": {
        "submit": "Submit",
        "cancel": "Cancel"
      },
      "time": {
        "justNow": "Just now",
        "minutesAgo": "{count} min ago"
      }
    }
  }
}
```

**Key structure**: `feature.section.subsection.key`
**Usage**: `useTranslations('feature.section')` then `t('title')`

---

## Checklist

### Before Submitting

- [ ] No hardcoded strings in JSX
- [ ] All aria-labels use `t()`
- [ ] Time/date formatting internationalized
- [ ] Error messages internationalized
- [ ] Added translations to ALL locale files
- [ ] Tested in multiple languages
- [ ] Used correct hook (useTranslations vs getTranslations)
- [ ] Dynamic values use interpolation
- [ ] Plurals handled correctly

### File Changes

- [ ] Updated `en.json`
- [ ] Updated `es.json`
- [ ] Updated any other locale files
- [ ] Keys match component hierarchy

---

## Red Flags to Search For

```bash
# These patterns indicate missing i18n:
return 'Just now'
return "Coming soon"
aria-label="Back to"
<p>Loading...</p>
{count} items
```

---

## When in Doubt

**Rule of thumb**: If a user can see it or a screen reader can read it, it needs i18n.

**This includes:**

- Button text
- Labels and placeholders
- Error messages
- Loading states
- Empty states
- Tooltips
- aria-labels and aria-descriptions
- Time/date displays
- Number formatting
