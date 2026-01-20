# Debugging Prompts

Prompt templates for diagnosing issues, understanding errors, and finding root causes in frontend code.

## Contents

Prompts in this category help you:
- Understand error messages
- Trace bugs to their root cause
- Debug React-specific issues
- Diagnose performance problems
- Fix TypeScript errors
- Resolve build and runtime issues

## Available Templates

*As you add prompt templates, list them here*

Example structure:
- [Error Analysis](error-analysis.md) - Understanding error messages
- [React Bug Diagnosis](react-bug-diagnosis.md) - Debugging React issues
- [Performance Investigation](performance-investigation.md) - Finding performance bottlenecks

## Common Debugging Scenarios

### Runtime Errors
- Undefined is not a function
- Cannot read property of undefined
- Maximum call stack exceeded
- Memory leaks

### React-Specific Issues
- Component not updating
- Infinite re-render loops
- Props not passed correctly
- State management bugs

### Build/Compilation Errors
- Module not found
- TypeScript type errors
- Webpack/Vite build failures
- Dependency conflicts

### Performance Issues
- Slow rendering
- UI freezing
- High memory usage
- Large bundle size

## Debugging Best Practices

### When Using Debug Prompts

1. **Include error messages**: Copy the complete error with stack trace
2. **Provide context**: Share relevant code, not just the line with error
3. **Describe expected behavior**: What should happen vs what is happening
4. **Mention environment**: Browser, React version, Node version, etc.
5. **Share what you've tried**: Helps avoid suggesting things you've already tested

### Information to Include

**Minimal reproducible example**:
```typescript
// Enough code to reproduce the issue
// But no more than necessary
```

**Error message**:
```
Complete error message with stack trace
```

**Environment**:
- React version
- Browser/Node version
- Relevant library versions

---

*Start adding your debugging prompts here!*
