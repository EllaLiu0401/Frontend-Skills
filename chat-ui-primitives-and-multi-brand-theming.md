# Chat UI Primitives and Multi-Brand Dark Mode Theming

Lessons learned from a design system PR review covering chat component architecture, React performance patterns, semantic HTML choices, and multi-brand theme token refactoring.

---

## 1. Compose Chat UIs from Primitives, Not Monoliths

### Problem

A single `<Chat>` wrapper component tries to handle every chat layout variant — user bubbles, assistant messages, tool calls, thinking indicators, loading states. It becomes rigid and hard to extend.

### Solution: Build Small, Composable Primitives

Instead of one monolithic component, create focused primitives:

```
ChatBubble       — colored message bubble (user messages)
ChatMessageList  — scrollable container with auto-scroll-to-bottom
ChatInput        — auto-resizing textarea with send action
ChatThinking     — expandable AI reasoning disclosure
ChatToolCall     — collapsible tool invocation panel
ChatSkeleton     — loading placeholder for incoming messages
```

Each primitive handles one concern. Consumers compose them freely:

```tsx
<ChatMessageList>
  <ChatBubble align="end" color="primary">{userMessage}</ChatBubble>
  <div className="py-2">
    <ChatThinking content={reasoning} isActive={!hasResponse} />
    <p>{assistantMessage}</p>
    <ChatToolCall name="search" input={json} variant="inline" />
  </div>
</ChatMessageList>
<ChatInput onSend={handleSend} />
```

### Why This Matters

- **Flexibility**: User messages use bubbles; assistant messages use plain text — different patterns, same primitives
- **Extensibility**: Adding a new message type (e.g., image, code block) doesn't require modifying existing components
- **Testing**: Each primitive can be tested and documented in isolation

### Rule of Thumb

> If a chat component has more than 3 conditional render paths based on message type, it should be split into focused primitives that consumers compose.

---

## 2. Multi-Brand Dark Mode: Fix Tokens at the Source

### Problem

A design system supports multiple brands (Brand A, Brand B), each with light and dark themes. In dark mode, colored chat bubbles need dark text on bright backgrounds. The initial approach:

```css
/* ❌ Per-brand overrides with hardcoded brand variables */
[data-theme='brand-a-dark'] .chat-bubble-primary {
  color: var(--brand-a-dark-text) !important;
}
[data-theme='brand-a-dark'] .chat-bubble-success {
  color: var(--brand-a-dark-text) !important;
}
/* ... repeat for every color variant */

/* Brand B needs its own block */
[data-theme='brand-b-dark'] .chat-bubble-primary {
  color: var(--brand-b-dark-text) !important;
}
/* ... repeat again */
```

This approach has three problems:

1. **Duplicates token information** — the theme already defines `--color-primary-content` for this purpose
2. **Scales poorly** — every new brand requires a new block of overrides
3. **Masks bugs** — if a theme token is wrong, the override silently fixes it instead of exposing the root cause

### Root Cause

The UI framework (e.g., DaisyUI) defines `--color-*-content` tokens for text on colored backgrounds, but its CSS specificity doesn't always win in dark themes. Developers added per-brand overrides as a workaround, hardcoding brand-specific variables instead of referencing the framework's own tokens.

Worse: some theme files had incorrect token values (e.g., white text on a light cyan background), and the overrides silently fixed the wrong token with the right hardcoded value — hiding the bug.

### Solution: Two-Step Refactor

**Step 1: Fix the theme tokens at the source**

```css
/* ❌ Before — theme defines wrong contrast */
--color-info: #33cdef;           /* light cyan background */
--color-info-content: #ffffff;   /* white text — fails WCAG AA */

/* ✅ After — theme defines correct contrast */
--color-info: #33cdef;
--color-info-content: #000000;   /* dark text — 11.5:1 ratio */
```

**Step 2: Replace per-brand overrides with generic token references**

```css
/* ✅ One rule set covers ALL dark themes */
[data-theme$='-dark'] .chat-bubble-primary,
[data-theme='dark'] .chat-bubble-primary {
  color: var(--color-primary-content) !important;
}

[data-theme$='-dark'] .chat-bubble-success,
[data-theme='dark'] .chat-bubble-success {
  color: var(--color-success-content) !important;
}

/* ... same pattern for error, warning, info */
```

Key techniques:
- `[data-theme$='-dark']` — CSS attribute ends-with selector matches any theme ending in `-dark`
- `var(--color-*-content)` — references the theme's own token, not a hardcoded brand variable
- `!important` — only needed to fix the framework's specificity issue, not to override theme values

### Before vs After

| Aspect | Before | After |
|--------|--------|-------|
| Lines of CSS | ~50 (per-brand blocks) | ~15 (one generic block) |
| Adding a new brand | Write new override block | Zero CSS changes needed |
| Token as source of truth | No — overrides bypass tokens | Yes — overrides reference tokens |
| Hidden bugs | Yes — wrong tokens silently fixed | No — wrong tokens cause visible failures |

### Rule of Thumb

> When a UI framework's specificity requires `!important` overrides, reference the framework's own tokens (`var(--color-*-content)`) instead of hardcoding brand-specific variables. Fix incorrect tokens at the source (theme file), not at the override site.

---

## 3. React Performance Micro-Patterns for Chat Components

### 3a. Stable Callbacks with Value Refs

**Problem**: A send handler reads the current textarea value, causing it to be recreated on every keystroke:

```tsx
// ❌ handleSend recreates on every value change
const [value, setValue] = useState('');

const handleSend = useCallback(() => {
  const trimmed = value.trim();  // reads from state
  if (!trimmed) return;
  onSend(trimmed);
  setValue('');
}, [value, onSend]);  // value in deps = recreates on every keystroke
```

**Fix**: Use a ref to read the current value without adding it to the dependency array:

```tsx
// ✅ handleSend is stable — only recreates when onSend or disabled changes
const [value, setValue] = useState('');
const valueRef = useRef(value);
valueRef.current = value;

const handleSend = useCallback(() => {
  const trimmed = valueRef.current.trim();  // reads from ref
  if (!trimmed || disabled) return;
  onSend(trimmed);
  setValue('');
}, [disabled, onSend]);  // stable deps
```

### 3b. `requestAnimationFrame` for DOM Measurements

**Problem**: Auto-resizing a textarea reads and writes `style.height` on every input event, causing layout thrashing:

```tsx
// ❌ Synchronous read + write = layout thrashing
const handleInput = useCallback(() => {
  const el = textareaRef.current;
  if (!el) return;
  el.style.height = 'auto';
  el.style.height = `${Math.min(el.scrollHeight, 160)}px`;
}, []);
```

**Fix**: Batch the DOM manipulation into a single animation frame:

```tsx
// ✅ Batched in rAF — no layout thrashing
const handleInput = useCallback(() => {
  const el = textareaRef.current;
  if (!el) return;
  requestAnimationFrame(() => {
    el.style.height = 'auto';
    el.style.height = `${Math.min(el.scrollHeight, 160)}px`;
  });
}, []);
```

### 3c. `Children.count()` over `Children.toArray()`

```tsx
// ❌ O(n) — creates a full array just to check emptiness
const hasChildren = Children.toArray(children).length > 0;

// ✅ O(1) — counts without allocating an array
const hasChildren = Children.count(children) > 0;
```

### 3d. Auto-Scroll `useEffect` Needs Dependencies

```tsx
// ❌ Missing dependency — only runs once, not on new messages
useEffect(() => {
  bottomRef.current?.scrollIntoView({ behavior: 'smooth' });
}, []);

// ✅ Runs when children change, uses 'auto' to avoid jank
useEffect(() => {
  if (stickToBottom.current) {
    bottomRef.current?.scrollIntoView({ behavior: 'auto' });
  }
}, [children]);
```

Note: `behavior: 'auto'` (instant jump) is better than `'smooth'` for programmatic scroll-to-bottom because smooth scrolling can fall behind during rapid message bursts.

---

## 4. Semantic HTML: `<details>/<summary>` for Disclosure Widgets

### Problem

A "thinking" indicator uses a `<button>` with manual `useState` and `aria-expanded`:

```tsx
// ❌ Manual state management for a native browser feature
const [expanded, setExpanded] = useState(false);

return (
  <div>
    <button
      aria-expanded={expanded}
      onClick={() => setExpanded(!expanded)}
    >
      Thinking...
    </button>
    {expanded && <div>{content}</div>}
  </div>
);
```

### Fix: Use `<details>/<summary>` — Native, Accessible, Less Code

```tsx
// ✅ Browser handles expand/collapse, keyboard nav, and screen reader announcements
return (
  <details>
    <summary className="cursor-pointer list-none [&::-webkit-details-marker]:hidden">
      <Icon name="Brain" /> Thinking...
    </summary>
    <div>{content}</div>
  </details>
);
```

Benefits:
- **No state needed** — browser manages open/closed natively
- **Accessible by default** — screen readers announce the disclosure role
- **Keyboard support** — Enter/Space toggle without custom handlers
- **Less code** — removes `useState`, `aria-expanded`, and click handler

### When to Use `<details>` vs Custom Toggle

| Use `<details>/<summary>` | Use custom `<button>` + state |
|---|---|
| Simple show/hide content | Animated transitions (CSS transitions on `<details>` are limited) |
| No animation required | Need to control open state from parent |
| Content is secondary/supplementary | Toggle affects other parts of the UI |

---

## 5. Skeleton Loading Over Spinners for Chat

### Problem

A chat interface shows a spinner while waiting for AI responses. This creates perceived latency — the user sees "nothing" becoming "something" in a jarring transition.

### Fix: Use Skeleton Placeholders

Skeleton placeholders with staggered widths mimic the shape of natural text:

```tsx
function ChatSkeleton({ lines = 3 }) {
  return (
    <div className="space-y-2 py-2">
      {Array.from({ length: lines }, (_, i) => (
        <Skeleton
          key={i}
          variant="text"
          animation="shimmer"
          className={i === lines - 1 ? 'w-2/3' : 'w-full'}
        />
      ))}
    </div>
  );
}
```

Why skeletons win over spinners:
- **Perceived speed** — users perceive skeleton screens as 10-20% faster (Doherty Threshold)
- **Layout stability** — content area maintains its shape, no layout shift when content arrives
- **Progressive feel** — the UI looks like it's "almost ready" instead of "still loading"

---

## Summary Cheat Sheet

| Situation | Wrong Approach | Right Approach |
|-----------|---------------|----------------|
| Multi-brand dark mode overrides | Per-brand CSS blocks with hardcoded variables | Generic `[data-theme$='-dark']` selector + `var(--color-*-content)` |
| Theme token has wrong contrast | Add CSS override to silently fix it | Fix the token in the theme file |
| Callback reads frequently-changing state | State variable in `useCallback` deps | `useRef` to read current value |
| DOM read/write on every input event | Synchronous in event handler | Wrap in `requestAnimationFrame` |
| Show/hide supplementary content | `<button>` + `useState` + `aria-expanded` | `<details>/<summary>` |
| Chat loading state | Spinner | Skeleton with staggered line widths |
| Chat component handles many message types | One monolithic component with conditionals | Composable primitives (Bubble, Thinking, ToolCall, etc.) |
