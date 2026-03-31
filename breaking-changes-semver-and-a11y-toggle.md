# Breaking Changes, Semantic Versioning, and Accessible Toggle Components

Lessons learned from a PR review involving a component refactor that removed public props, dropped an ARIA role, and exposed gaps in CI pipeline configuration for breaking change workflows.

---

## 1. Breaking API Changes Require Major Version Bumps

### Problem

When refactoring a component from a custom implementation to a library-native wrapper, it's tempting to treat the change as a simple `fix` or `refactor`. But if the refactor **removes props from the public interface**, consumers passing those props will get TypeScript compilation errors after upgrading.

### Example

```typescript
// Before: custom implementation with color override props
interface ToggleProps {
  size?: 'sm' | 'md' | 'lg';
  color?: 'primary' | 'success' | 'error';
  activeColor?: string;    // custom on-state color
  inactiveColor?: string;  // custom off-state color
}

// After: thin wrapper around a UI library's native toggle
interface ToggleProps {
  size?: 'sm' | 'md' | 'lg';
  color?: 'primary' | 'success' | 'error';
  // activeColor and inactiveColor removed
}
```

A consumer using `<Toggle activeColor="red" />` will now fail to compile.

### Rule

Per [Semantic Versioning](https://semver.org/):

| Change type | Version bump | Example |
|---|---|---|
| Bug fix, no API change | **patch** (1.2.3 → 1.2.**4**) | `fix: resolve hover flash` |
| New feature, backward compatible | **minor** (1.2.3 → 1.**3**.0) | `feat: add size="xl"` |
| Removed/renamed prop, changed behavior | **major** (1.2.3 → **2**.0.0) | `fix!: simplify Toggle` |

Even if no known consumers use the removed props, the semver contract should reflect the potential for breakage. This is about **trust in the version number**.

### Conventional Commits Syntax

Two ways to signal a breaking change:

```bash
# Option A: ! after the type
fix!(ui): simplify Toggle component

# Option B: BREAKING CHANGE footer in commit body
fix(ui): simplify Toggle component

BREAKING CHANGE: activeColor and inactiveColor props removed from Toggle
```

Both trigger a major version bump in `semantic-release`.

---

## 2. ARIA `role="switch"` for Toggle Components

### Problem

A visually presented toggle/switch that uses `<input type="checkbox">` under the hood will be announced as "checkbox" by screen readers. This creates a disconnect — the user sees a switch but hears "checkbox."

### The Fix

Add `role="switch"` to the input element:

```tsx
// ❌ Screen readers announce "checkbox"
<input type="checkbox" className="toggle" />

// ✅ Screen readers announce "switch"
<input type="checkbox" role="switch" className="toggle" />
```

### Key Points

- `role="switch"` is a **pure ARIA attribute** — it has zero impact on visual styling
- No CSS selector in any major UI library (DaisyUI, Tailwind, etc.) targets `[role="switch"]`
- The WAI-ARIA spec defines [switch role](https://www.w3.org/TR/wai-aria-1.1/#switch) for elements that represent an on/off toggle
- Axe and similar automated a11y tools may **not** flag a missing `role="switch"` — it's a semantic best practice, not a hard failure

### When to Use

| Visual presentation | Correct role |
|---|---|
| Toggle switch (on/off slider) | `role="switch"` |
| Checkbox (tick/untick square) | Default `type="checkbox"` role (no override needed) |

### Refactoring Checklist

When migrating a custom component to a library wrapper, audit ARIA attributes from the original implementation — they're easy to lose in the rewrite.

---

## 3. PR Title Validation and Breaking Change Support

### Problem

Many projects use [amannn/action-semantic-pull-request](https://github.com/amannn/action-semantic-pull-request) to enforce Conventional Commits format on PR titles. The default configuration **does not** parse the `!` breaking change notation:

```
fix!(ui): simplify Toggle    →  ❌ "No release type found"
fix(ui): simplify Toggle      →  ✅ passes
```

### Root Cause

The action's default `headerPattern` regex is:

```
^(\w*)(?:\(([\w$.\-*/ ]*)\))?: (.*)$
```

This matches `fix(scope): subject` but **not** `fix!(scope): subject` — the `!` is unrecognized.

### The Fix

Override `headerPattern` to make `!` optional:

```yaml
- uses: amannn/action-semantic-pull-request@v5
  with:
    types: |
      feat
      fix
      docs
      refactor
      chore
    headerPattern: '^(\w*)(?:\(([\w$.\-*/ ]*)\))?!?: (.*)$'
    headerPatternCorrespondence: type, scope, subject
```

The key change is `?!?:` — making `!` an optional character before the colon.

### Important

Note that `headerPattern` and `headerPatternCorrespondence` must be set **together** — the correspondence tells the parser which capture group maps to type, scope, and subject.

---

## 4. YAML Quoting for Regex Patterns in GitHub Actions

### Problem

YAML treats `:` followed by a space as a key-value delimiter. An unquoted regex containing `: ` will break YAML parsing silently or cause unexpected behavior:

```yaml
# ❌ YAML interprets `: (.*)$` as a mapping
headerPattern: ^(\w*)(?:\(([\w$.\-*/ ]*)\))?!?: (.*)$

# ✅ Quoted string — YAML treats entire value as a string
headerPattern: '^(\w*)(?:\(([\w$.\-*/ ]*)\))?!?: (.*)$'
```

### Rule of Thumb

Always quote YAML values that contain any of these characters:

| Character | Why it's dangerous |
|---|---|
| `:` | Key-value delimiter |
| `#` | Comment start |
| `{` `}` | Flow mapping |
| `[` `]` | Flow sequence |
| `*` `&` | Anchors and aliases |
| `!` | Tag indicator |
| `>` `\|` | Block scalar indicators |
| `@` `` ` `` | Reserved characters |

For regex patterns in workflow files, **always use single quotes** — they prevent YAML interpretation while keeping the regex intact (no escape sequences like double quotes would need).

---

## 5. Separation of Concerns: PR Validation vs Release Pipeline

### Key Insight

PR title validation and the release pipeline are **independent systems** that both parse commit messages:

```
PR Title Check (CI)              Release Pipeline (CD)
─────────────────                ─────────────────────
action-semantic-pull-request     @semantic-release/commit-analyzer
Runs on: pull_request events     Runs on: push to main
Purpose: enforce format          Purpose: determine version bump
Config: headerPattern in YAML    Config: preset in .releaserc.json
```

A fix to one does **not** automatically fix the other. When adding breaking change support:

1. **Verify the PR title checker** accepts `!` syntax (headerPattern)
2. **Verify the release tool** handles `!` for major bumps (check `commit-analyzer` version — v11+ supports it natively)

### Version Compatibility

| `@semantic-release/commit-analyzer` version | `!` support |
|---|---|
| < v11 | ❌ Only `BREAKING CHANGE:` footer |
| v11+ | ✅ Both `!` and `BREAKING CHANGE:` footer |

---

## Summary Checklist

When refactoring components in a design system:

- [ ] Audit removed/renamed props — if public API changes, it's a **major** version bump
- [ ] Audit ARIA attributes from the original implementation (role, aria-checked, aria-label, etc.)
- [ ] Verify `role="switch"` is present on any element visually presented as a toggle
- [ ] Use `fix!:` or `BREAKING CHANGE:` footer for breaking changes
- [ ] Confirm CI pipeline supports `!` notation in PR titles (check `headerPattern`)
- [ ] Confirm release pipeline supports `!` notation (check `commit-analyzer` version)
- [ ] Always quote regex patterns in YAML workflow files
