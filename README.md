# Frontend Skills - Learning Repository

A living documentation of frontend development learnings, best practices, and patterns discovered through code reviews and real-world experience.

## Purpose

This repository serves as a personal knowledge base for frontend development insights. It captures lessons learned from PR reviews, refactoring experiences, and architectural decisions, with all examples generalized to be applicable across different projects.

## ⚡ Quick Reference

**需要快速查找规则？Need quick rule lookup?**

→ **[Quick Reference Guide](docs/quick-reference/)** - 按问题类型快速查找编码规则

常用规则速查 | Quick access to common rules:
- [TypeScript Rules](docs/quick-reference/typescript-rules.md) - 类型推断、类型错误、泛型
- [React Rules](docs/quick-reference/react-rules.md) - Hooks、性能、组件设计
- [Code Quality Rules](docs/quick-reference/code-quality-rules.md) - 命名、函数大小、复杂度
- [PR Review Checklist](docs/quick-reference/pr-review-checklist.md) - 完整的 PR 检查清单

## Quick Navigation

### Core Technologies
- [JavaScript](docs/javascript/) - Modern JavaScript patterns and best practices
- [TypeScript](docs/typescript/) - Type system patterns and type safety strategies
- [React](docs/react/) - React-specific patterns, hooks, and component design

### Architecture & Patterns
- [State Management](docs/state-management/) - State handling patterns and anti-patterns
- [API Integration](docs/api-integration/) - Data fetching, caching, and synchronization
- [Error Handling](docs/error-handling/) - Error boundaries, recovery strategies, and user feedback

### Performance & Quality
- [Performance](docs/performance/) - Optimization techniques and profiling insights
- [Testing](docs/testing/) - Testing strategies, patterns, and common pitfalls
- [Accessibility](docs/accessibility/) - A11y best practices and WCAG compliance
- [Best Practices](docs/best-practices/) - PR learnings, code quality rules, and reusable patterns

### Styling & Tooling
- [Styling](docs/styling/) - CSS, Sass, CSS-in-JS, and design system patterns
- [Build Tools](docs/build-tools/) - Webpack, Vite, and bundler configurations

### AI-Assisted Development
- [Prompt Templates](prompts/) - Reusable prompts for code review, debugging, generation, and learning

## Documentation Formats

This repository uses three different formats depending on the type of learning:

### 📊 Before/After Format
Used for refactoring examples and code improvements. Shows the original code with issues, the improved version, and key takeaways.

**Best for**: Performance optimizations, code quality improvements, bug fixes

### 📘 Pattern Guide Format
Used for documenting general patterns and best practices. Includes when to use, when to avoid, and common pitfalls.

**Best for**: Architectural patterns, design principles, reusable solutions

### 🔍 Problem/Solution Format
Used for documenting specific issues encountered during PR reviews. Includes root cause analysis and prevention strategies.

**Best for**: Debugging insights, edge cases, lessons from production issues

## Repository Structure

```
Frontend-Skills/
├── docs/              # Learning documentation organized by topic
├── prompts/           # AI prompt templates for development tasks
└── templates/         # Documentation templates for new entries
```

## How to Use This Repository

### Finding Information

**By Topic**: Navigate through the folder structure above to find learnings organized by category.

**By Search**: Use your editor's search or command line tools:
```bash
# Find all learnings about hooks
grep -r "hooks" docs/

# Find performance-related content
grep -r "performance" docs/
```

**By Pattern**: Look for descriptive filenames that match your current challenge:
- `avoid-prop-drilling.md`
- `custom-hooks-best-practices.md`
- `api-error-handling.md`

### Adding New Learnings

See [CONTRIBUTING.md](CONTRIBUTING.md) for detailed guidelines on adding new entries.

Quick workflow:
1. Identify the appropriate topic category
2. Choose the right documentation template
3. Create a new markdown file with a descriptive name
4. Follow the template structure
5. Update the topic's README with a link to your new entry

## Content Principles

**Logical**: Content flows from basic to advanced within each topic

**Consistent**: Similar learnings use similar formats and structure

**Clear**: Every example is self-contained and understandable without context

**General**: All examples are abstracted to be company-agnostic and broadly applicable

**Actionable**: Each entry provides concrete takeaways and prevention strategies

## Why This Exists

Every developer encounters similar challenges and learns similar lessons. This repository is a way to:
- Retain knowledge from code reviews and PR discussions
- Avoid repeating the same mistakes
- Share patterns that work (and anti-patterns that don't)
- Build a personal reference for future projects
- Track growth and evolution as a developer

## Getting Started

If you're new to this repository:
1. Browse the [JavaScript](docs/javascript/) and [React](docs/react/) sections for foundational patterns
2. Check out [Error Handling](docs/error-handling/) for common pitfalls
3. Review [Testing](docs/testing/) for quality assurance strategies
4. Read [CONTRIBUTING.md](CONTRIBUTING.md) before adding your first entry

---

**Last Updated**: January 2026
**Maintained By**: Personal learning collection
**Status**: Active and continuously growing
