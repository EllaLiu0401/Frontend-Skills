# Prompt Templates

A collection of reusable prompt templates for AI-assisted frontend development, code review, debugging, and learning.

## Overview

This section contains prompt templates that have proven effective for various development tasks. Use these as starting points and customize them for your specific needs.

## Categories

### [Code Review](code-review/)
Prompts for reviewing code quality, identifying issues, and suggesting improvements.

### [Debugging](debugging/)
Prompts for diagnosing issues, understanding error messages, and finding root causes.

### [Code Generation](code-generation/)
Prompts for generating components, utilities, tests, and boilerplate code.

### [Refactoring](refactoring/)
Prompts for improving code structure, performance, and maintainability.

### [Learning](learning/)
Prompts for understanding concepts, patterns, and best practices.

### [Documentation](documentation/)
Prompts for generating comments, README files, and API documentation.

## How to Use Prompt Templates

### Structure of Each Template

Each prompt template includes:

1. **Purpose**: What the prompt is designed to achieve
2. **Template**: The actual prompt with placeholders for variables
3. **Variables**: Explanation of what to fill in
4. **Example**: Concrete example of the prompt in use
5. **Tips**: Best practices for using this prompt
6. **Expected Output**: What kind of response to expect

### Using Variables

Templates use placeholders in the format `{VARIABLE_NAME}`. Replace these with your specific content:

```
Example template:
"Review this {COMPONENT_TYPE} and suggest improvements for {FOCUS_AREA}"

Filled in:
"Review this React component and suggest improvements for performance"
```

### Customization

Feel free to:
- Combine multiple templates
- Add context specific to your project
- Adjust the level of detail requested
- Specify output format (bullet points, code blocks, etc.)

## Adding New Templates

When you discover an effective prompt:

1. Identify which category it belongs to
2. Create a new markdown file with a descriptive name
3. Follow the template structure (see below)
4. Add practical examples
5. Update the category's README

### Template File Structure

```markdown
# [Prompt Name]

## Purpose
Brief description of what this prompt achieves

## Template
\`\`\`
Your prompt template with {VARIABLES}
\`\`\`

## Variables
- {VARIABLE_NAME}: Description of what to provide

## Example
Concrete example of the prompt filled in

## Expected Output
What kind of response you'll get

## Tips
- Best practices
- Common pitfalls
- When to use/not use
```

## Best Practices

### Writing Effective Prompts

1. **Be specific**: Clear, detailed prompts get better results
2. **Provide context**: Include relevant information about your codebase or requirements
3. **Set constraints**: Specify what to avoid or limitations to respect
4. **Request format**: Ask for specific output formats (code blocks, explanations, etc.)
5. **Iterate**: Refine prompts based on results

### Organizing Your Prompts

- Use descriptive filenames: `review-react-performance.md` not `prompt1.md`
- Keep related prompts together in categories
- Cross-reference related prompts
- Update prompts when you find better versions

### Security Considerations

⚠️ **Never include**:
- API keys or credentials
- Proprietary business logic
- Sensitive user data
- Internal system details

Always generalize examples to be company-agnostic.

## Quick Start Examples

### Quick Code Review
```
Review this code for potential issues, focusing on:
- Performance concerns
- Security vulnerabilities
- Best practice violations
- Potential bugs

[paste code here]
```

### Quick Refactor
```
Refactor this code to improve:
1. Readability
2. Maintainability
3. Performance

Keep the functionality identical. Explain each change.

[paste code here]
```

### Quick Debug
```
I'm getting this error: [error message]

Here's the relevant code:
[paste code]

Help me:
1. Understand what's causing this error
2. Find the root cause
3. Suggest a fix
```

## Resources

### Learning More About Prompts

- [OpenAI Prompt Engineering Guide](https://platform.openai.com/docs/guides/prompt-engineering)
- [Anthropic Prompt Library](https://docs.anthropic.com/claude/prompt-library)

---

**Remember**: Good prompts are iterative. Start with these templates, refine them based on results, and save your improvements back to this repository.
