# Documentation Hierarchy for Teams and AI Agents

## Problem

When working with teams (humans + AI), inconsistent documentation leads to:
- Different team members following different patterns
- AI agents not knowing which rules to follow
- Outdated or conflicting information across docs
- Unclear which document is the "source of truth"

## Solution: Documentation Priority Hierarchy

### Establish Clear Priority Levels

```
Priority 1: AGENTS.md / .cursor/rules/     (Mandatory rules)
Priority 2: planning/                      (Architecture & patterns)
Priority 3: features/                      (Product requirements)
Priority 4: implementation/                (Ephemeral task specs)
```

### Document Types and Their Roles

#### 1. **AGENTS.md** (Highest Priority)
- **Purpose**: Non-negotiable rules for AI and developers
- **Content**: MANDATORY requirements, banned patterns, architecture constraints
- **Lifecycle**: Permanent - changes require team review
- **Style**: Concise, actionable, uses keywords like "MANDATORY", "NEVER"

**Example structure:**
```markdown
## Rules you must follow

Environment & Secrets:
• MANDATORY: No client-side secrets (NEXT_PUBLIC_* forbidden)
• All secrets must be server-side only

Layout & Navigation:
• MANDATORY: Header must use `sticky top-0` positioning
• MANDATORY: All pages must use the standard PageLayout component

Code Quality:
• NEVER disable linting with eslint-disable comments
• If code doesn't pass strict checks, fix the code - don't change rules
```

#### 2. **planning/** (Second Priority)
- **Purpose**: Detailed architecture, patterns, and "how-to" guides
- **Content**: Design decisions, code examples, best practices
- **Lifecycle**: Permanent - source of truth for architecture
- **Style**: Detailed with examples and rationale

**Example files:**
- `planning/frontend-architecture.md` - Framework choices, rendering strategy
- `planning/component-patterns.md` - How to structure components
- `planning/api-integration.md` - How to call APIs

#### 3. **features/** (Third Priority)
- **Purpose**: Product roadmap and feature descriptions
- **Content**: User stories, acceptance criteria, business logic
- **Lifecycle**: Permanent - evolves with product
- **Style**: Product-focused, not implementation-focused

#### 4. **implementation/** (Lowest Priority)
- **Purpose**: Task specs for specific work items
- **Content**: Acceptance criteria, implementation steps for tickets
- **Lifecycle**: Ephemeral - archived after completion
- **Style**: Detailed task breakdown

### Key Principles

#### Single Source of Truth
**Problem:** Same information repeated in multiple places leads to inconsistency when updating.

**Solution:** 
- Define detailed information once in planning docs
- Reference it from higher-level docs
- Use cross-references to avoid duplication

**Example:**

```markdown
// ❌ BAD - Duplicating details in AGENTS.md
Layout Requirements:
• Use the <PageLayout> component
• Props: title (string, required), subtitle (string, optional), 
  icon (ReactNode, required), actions (ReactNode, optional)
• Example: <PageLayout title="Dashboard" icon={<Icon />}>...</PageLayout>
• Provides standardized header...

// ✅ GOOD - Reference detailed docs
Layout Requirements:
• MANDATORY: All pages must use <PageLayout> component
• See planning/frontend-architecture.md §4 for usage details
```

#### When Information Conflicts

If documents contradict each other, follow this priority:
1. AGENTS.md prevails over planning docs
2. planning/ prevails over features/
3. features/ prevails over implementation/

### How AI Agents Use This Hierarchy

**On every coding task, AI will:**
1. Read AGENTS.md rules (always enforced)
2. Check relevant planning/ docs for patterns
3. Reference features/ for product requirements
4. Follow implementation/ specs when provided

**This ensures:**
- ✅ Consistency across all AI-generated code
- ✅ Architectural decisions are respected
- ✅ New team members get same guidance as AI

## Implementation Steps

### 1. Create the Hierarchy

```bash
project/
├── AGENTS.md                    # Mandatory rules
├── planning/
│   ├── README.md               # Index of planning docs
│   ├── frontend-architecture.md
│   ├── api-conventions.md
│   └── testing-strategy.md
├── features/
│   └── user-authentication.md
└── implementation/
    └── tasks/
        └── add-login-page.md   # Archived after completion
```

### 2. Write AGENTS.md

```markdown
# Project Rules for AI and Developers

## Documentation Hierarchy
Priority: AGENTS.md > planning/ > features/ > implementation/

## Mandatory Rules

[Your non-negotiable rules here]

## Architecture References
• Frontend patterns: see planning/frontend-architecture.md
• API conventions: see planning/api-conventions.md
```

### 3. Add Cross-References

In AGENTS.md (brief):
```markdown
• MANDATORY: All dashboard pages must use <DashboardPage> component
• See planning/frontend-architecture.md §4 for details
```

In planning/frontend-architecture.md (detailed):
```markdown
## Standard Components

### DashboardPage
Required for all dashboard pages to ensure consistent styling.

**Props:**
- `title` (string, required) - Page heading
- `subtitle` (string, optional) - Page description
- `icon` (ReactNode, required) - Page icon
- `actions` (ReactNode, optional) - Action buttons

**Example:**
[detailed code example here]
```

### 4. Keep AGENTS.md Concise

**Target:** Under 200 lines
**Strategy:** 
- High-level rules only
- Link to planning docs for details
- Use bullet points, not paragraphs
- Include only what AI *must always remember*

## Real-World Benefits

### For Teams
- ✅ New developers know where to look
- ✅ No confusion about "which doc is right"
- ✅ Easy to update (change once, reference everywhere)

### For AI Agents
- ✅ Consistent code generation
- ✅ Follows architectural decisions automatically
- ✅ Reduces back-and-forth corrections

### For Code Reviews
- ✅ Easy to point to specific rules
- ✅ Less "why did you do it this way?" questions
- ✅ Objective standards for approval

## Common Mistakes to Avoid

### ❌ Mistake 1: No Clear Priority
```markdown
// Multiple docs with conflicting info, no hierarchy
docs/coding-standards.md: "Use PageWrapper component"
docs/guidelines.md: "Use LayoutContainer component"
README.md: "Wrap pages in StandardLayout"
```

### ✅ Fix: Establish Priority
```markdown
AGENTS.md: "MANDATORY: Use <PageLayout>"
All other docs: Reference AGENTS.md or planning/
```

### ❌ Mistake 2: Duplicating Details
```markdown
// Same example in 3 places - hard to maintain
AGENTS.md: [full component example]
planning/frontend.md: [same full example]
implementation/task.md: [same full example again]
```

### ✅ Fix: Define Once, Reference Everywhere
```markdown
planning/frontend.md: [full example - source of truth]
AGENTS.md: "See planning/frontend.md §4"
implementation/task.md: "Follow pattern in planning/frontend.md §4"
```

### ❌ Mistake 3: Too Much in AGENTS.md
```markdown
// 1000+ lines with every detail
AGENTS.md:
  - Detailed API documentation
  - Full code examples for every pattern
  - Product requirements
  - Task breakdowns
```

### ✅ Fix: Keep It Focused
```markdown
// ~150 lines of mandatory rules only
AGENTS.md:
  - Must-follow rules
  - Banned patterns
  - Links to detailed docs
```

## Maintenance Strategy

### Monthly Review
- Check for conflicting information
- Update cross-references if docs move
- Archive completed implementation/ tasks

### When Adding New Rules
1. Is it mandatory for everyone? → AGENTS.md
2. Is it architectural guidance? → planning/
3. Is it a product feature? → features/
4. Is it a specific task? → implementation/

### When Information Changes
1. Update the source of truth (usually planning/)
2. Verify cross-references still work
3. Check if AGENTS.md needs updating
4. Announce changes to team

## Summary

**Key Takeaways:**
- Establish clear documentation priority hierarchy
- Define information once (Single Source of Truth)
- Use cross-references to avoid duplication
- Keep AGENTS.md concise and focused on mandatory rules
- AI and humans follow the same hierarchy
- Regular maintenance keeps docs accurate

**Next Steps:**
1. Create your documentation hierarchy
2. Write concise AGENTS.md with mandatory rules
3. Add detailed planning docs
4. Use cross-references everywhere
5. Train team on the hierarchy

---

**Related Topics:**
- [Code Quality Rules](../quick-reference/code-quality-rules.md)
- [Component Standardization](./component-standardization-patterns.md)
- [Git Hook Environment Setup](./git-hooks-nvm-path-fix.md)
