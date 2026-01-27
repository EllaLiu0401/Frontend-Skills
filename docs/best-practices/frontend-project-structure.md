# Frontend Project Structure: Organizing Utilities and Services

**Category**: Best Practices / Project Organization  
**Difficulty**: Intermediate  
**Tags**: `architecture`, `project-structure`, `organization`, `conventions`

---

## Overview

Proper file organization is critical for maintainability in large frontend projects. This guide covers best practices for structuring utilities, services, and other common directories in React/Next.js applications.

---

## The Problem: `lib/` vs `utils/` vs `services/`

Many projects use these directories interchangeably, leading to confusion:

```
❌ Inconsistent Structure
src/
  lib/
    formatNumber.ts         # Utility function
    apiClient.ts            # Service
    constants.ts            # Constants
  utils/
    dateUtils.ts            # Utility function
    errorMessages.ts        # Utility function
  helpers/
    validation.ts           # Utility function
```

**Issues:**
- No clear separation of concerns
- Hard to find files
- Inconsistent naming conventions
- Difficult for new team members

---

## Best Practice: Clear Separation

### Recommended Structure

```
✅ Clean Structure
src/
  app/                      # Next.js routes (lowercase)
  components/               # React components (PascalCase)
  hooks/                    # Custom hooks (camelCase, use* prefix)
  utils/                    # Pure utility functions (camelCase)
  services/                 # Business logic orchestration (camelCase)
  integrations/             # External API adapters (camelCase)
  contexts/                 # React contexts (PascalCase)
  types/                    # TypeScript type definitions
  config/                   # Configuration files
```

### Key Principles

1. **No `lib/` directory** - Too vague and ambiguous
2. **Use specific, descriptive directory names**
3. **Consistent naming conventions per directory**
4. **Co-locate related files**

---

## Directory Purposes

### `utils/` - Pure Utility Functions

**Purpose**: Stateless, reusable helper functions with no side effects

**Characteristics:**
- Pure functions (same input → same output)
- No external dependencies on services/APIs
- Generally small and focused
- Can be used anywhere in the app

**Examples:**
```typescript
// ✅ Good for utils/
utils/
  formatNumber.ts          // Format numbers with K/M suffixes
  dateUtils.ts             // Date parsing and formatting
  stringUtils.ts           // String manipulation
  validators.ts            // Input validation functions
  arrayUtils.ts            // Array transformation helpers

// formatNumber.ts
export function formatNumber(value: number): string {
  // Pure function, no side effects
  if (value >= 1_000_000) {
    return `${(value / 1_000_000).toFixed(1)}M`;
  }
  if (value >= 1_000) {
    return `${(value / 1_000).toFixed(1)}K`;
  }
  return value.toString();
}
```

**Anti-examples:**
```typescript
// ❌ Don't put these in utils/
utils/
  apiClient.ts             // ❌ This is a service
  UserService.ts           // ❌ This is a service
  twilioHelper.ts          // ❌ This is an integration
```

---

### `services/` - Business Logic Orchestration

**Purpose**: Domain-specific business logic that orchestrates data and operations

**Characteristics:**
- May have state or side effects
- Calls APIs, integrations, or other services
- Contains business rules
- Domain-focused (e.g., orders, users, notifications)

**Examples:**
```typescript
// ✅ Good for services/
services/
  authService.ts           // Authentication logic
  orderService.ts          // Order management
  notificationService.ts   // Notification handling
  analyticsService.ts      // Analytics tracking

// orderService.ts
import { apiClient } from '@/integrations/apiClient';
import { formatCurrency } from '@/utils/formatCurrency';

export async function createOrder(items: Item[]): Promise<Order> {
  // Orchestrates business logic
  const total = calculateTotal(items);
  const formattedTotal = formatCurrency(total);
  
  // Calls external API
  const order = await apiClient.post('/orders', {
    items,
    total,
  });
  
  return order;
}
```

---

### `integrations/` - External API Adapters

**Purpose**: Adapters for external services and third-party APIs

**Characteristics:**
- Wraps external SDKs/APIs
- Provides consistent interface for external services
- Handles authentication and configuration
- Abstracts implementation details

**Examples:**
```typescript
// ✅ Good for integrations/
integrations/
  stripe/
    stripeClient.ts        // Stripe payment integration
  sendgrid/
    sendgridClient.ts      // Email service integration
  twilio/
    twilioClient.ts        // SMS service integration
  analytics/
    googleAnalytics.ts     // GA4 integration

// sendgridClient.ts
import sgMail from '@sendgrid/mail';

export class SendGridClient {
  constructor(apiKey: string) {
    sgMail.setApiKey(apiKey);
  }
  
  async sendEmail(params: EmailParams): Promise<void> {
    // Wraps external SDK
    await sgMail.send({
      to: params.to,
      from: params.from,
      subject: params.subject,
      html: params.body,
    });
  }
}
```

---

### `hooks/` - Custom React Hooks

**Purpose**: Reusable React hooks for component logic

**Naming**: Always prefix with `use*`

**Examples:**
```typescript
hooks/
  useAuth.ts               // Authentication hook
  useLocalStorage.ts       // Local storage hook
  useDebounce.ts           // Debounce hook
  useFetch.ts              // Data fetching hook
```

---

### `components/` - React Components

**Purpose**: Reusable UI components

**Naming**: PascalCase for component files

**Structure:**
```typescript
components/
  ui/                      # Basic UI components
    Button.tsx
    Button.test.tsx
    Input.tsx
    Input.test.tsx
  forms/                   # Form-specific components
    LoginForm.tsx
    LoginForm.test.tsx
  layout/                  # Layout components
    Header.tsx
    Footer.tsx
```

---

## Migration Example

### Before: Inconsistent Structure

```typescript
// ❌ Bad: File in wrong directory
src/lib/formatNumber.ts

// ❌ Bad: Import from lib
import { formatNumber } from '@/lib/formatNumber';
```

### After: Proper Structure

```typescript
// ✅ Good: File in utils
src/utils/formatNumber.ts

// ✅ Good: Import from utils
import { formatNumber } from '@/utils/formatNumber';
```

### Step-by-Step Migration

1. **Identify the file type**
   - Is it a pure utility? → `utils/`
   - Does it orchestrate business logic? → `services/`
   - Does it wrap an external API? → `integrations/`

2. **Move the file**
   ```bash
   mv src/lib/formatNumber.ts src/utils/formatNumber.ts
   mv src/lib/formatNumber.test.ts src/utils/formatNumber.test.ts
   ```

3. **Update all imports**
   ```bash
   # Find all files importing from old path
   grep -r "@/lib/formatNumber" src/
   
   # Update each file
   # Change: import { formatNumber } from '@/lib/formatNumber';
   # To:     import { formatNumber } from '@/utils/formatNumber';
   ```

4. **Update test mocks**
   ```typescript
   // Before
   vi.mock('@/lib/formatNumber', () => ({ ... }));
   
   // After
   vi.mock('@/utils/formatNumber', () => ({ ... }));
   ```

5. **Run tests**
   ```bash
   npm test src/utils/formatNumber.test.ts
   npm test  # Run all tests to catch missed imports
   ```

6. **Delete old files**
   ```bash
   rm src/lib/formatNumber.ts
   rm src/lib/formatNumber.test.ts
   ```

---

## Enforcing Conventions

### ESLint Rules

Use `eslint-plugin-boundaries` to enforce import rules:

```javascript
// .eslintrc.js
module.exports = {
  plugins: ['boundaries'],
  rules: {
    'boundaries/element-types': ['error', {
      default: 'disallow',
      rules: [
        {
          from: 'components',
          allow: ['hooks', 'utils', 'types'],
          disallow: ['services', 'integrations']
        },
        {
          from: 'services',
          allow: ['integrations', 'utils', 'types']
        },
        {
          from: 'integrations',
          allow: ['utils', 'types']
        }
      ]
    }]
  }
};
```

### File Naming Convention

Use `eslint-plugin-unicorn` for file naming:

```javascript
// .eslintrc.js
rules: {
  'unicorn/filename-case': ['error', {
    cases: {
      camelCase: true,
      pascalCase: true
    },
    ignore: [
      // Next.js special files
      'page.tsx',
      'layout.tsx',
      'not-found.tsx'
    ]
  }]
}
```

---

## Decision Tree

Use this flowchart to decide where a file belongs:

```
Is it a React component?
├─ Yes → components/
└─ No ↓

Is it a React hook?
├─ Yes → hooks/
└─ No ↓

Does it call external APIs or SDKs?
├─ Yes → integrations/
└─ No ↓

Does it contain business logic or orchestrate operations?
├─ Yes → services/
└─ No ↓

Is it a pure utility function?
├─ Yes → utils/
└─ No ↓

Is it a type definition?
├─ Yes → types/ or co-locate
└─ No ↓

Is it configuration?
└─ Yes → config/
```

---

## Common Mistakes

### ❌ Mistake 1: Using `lib/` as a Catch-All

```
❌ Bad
lib/
  everything.ts            # Too vague!
  apiClient.ts             # Should be in integrations/
  formatNumber.ts          # Should be in utils/
  UserService.ts           # Should be in services/
```

### ❌ Mistake 2: Mixing Concerns

```
❌ Bad
utils/
  dateUtils.ts             # ✅ OK - Pure utility
  apiClient.ts             # ❌ NO - This is an integration
  fetchUsers.ts            # ❌ NO - This is a service
```

### ❌ Mistake 3: Not Co-locating Tests

```
❌ Bad
src/
  utils/
    formatNumber.ts
  __tests__/
    utils/
      formatNumber.test.ts  # Too far from source!

✅ Good
src/
  utils/
    formatNumber.ts
    formatNumber.test.ts    # Co-located!
```

---

## Benefits of Proper Organization

### 1. **Faster Onboarding**
New developers can quickly understand where to find and add code.

### 2. **Better Maintainability**
Clear separation makes it easier to refactor and update code.

### 3. **Improved Testability**
Pure utilities in `utils/` are easier to test than mixed-concern files.

### 4. **Enforced Architecture**
ESLint rules can enforce layer boundaries automatically.

### 5. **Reduced Cognitive Load**
Developers don't waste time deciding where files should go.

---

## Summary

| Directory | Purpose | Characteristics | Examples |
|-----------|---------|-----------------|----------|
| `utils/` | Pure utility functions | Stateless, no side effects | `formatNumber`, `dateUtils` |
| `services/` | Business logic | May have state, orchestrates | `orderService`, `authService` |
| `integrations/` | External API adapters | Wraps SDKs, abstracts APIs | `stripeClient`, `sendgridClient` |
| `hooks/` | React hooks | React-specific, reusable logic | `useAuth`, `useFetch` |
| `components/` | UI components | React components | `Button`, `LoginForm` |

---

## Related Reading

- **[Clean Architecture in Frontend](./clean-architecture.md)**
- **[Dependency Management Best Practices](./dependency-management.md)**
- **[Testing Strategies by File Type](../testing/testing-strategies.md)**

---

**Key Takeaway**: **No `lib/` directory. Use specific, purpose-driven directories: `utils/`, `services/`, `integrations/`.**

---

**Last Updated**: January 2026  
**Contributor**: Based on real project refactoring experience
