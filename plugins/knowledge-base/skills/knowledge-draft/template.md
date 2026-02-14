# Convention Template Specification

All conventions use the **Comprehensive tier** format (150-300 lines).

## File Structure

Every convention file must follow this structure:

```markdown
---
type: convention
version: {major.minor}
scope: {scope_identifier}
category: {category_type}
tags: [{tag1}, {tag2}, {tag3}]
---

# {Convention Title}

{Brief description explaining what this convention covers and why it exists}

## Format

- **{Concept 1}**: {Explanation}
- **{Concept 2}**: {Explanation}

[Additional sections based on tier...]
```

## YAML Front Matter Fields

### Required Fields

- **type**: Always `convention` (literal value)
- **version**: Semantic version `{major}.{minor}` (e.g., `1.0`, `1.2`, `2.0`)
- **scope**: Language or framework identifier
- **category**: Classification of convention type
- **tags**: Array of 3-5 relevant keywords

### Scope Values

**Predefined scopes:**
- `typescript` - TypeScript-specific conventions
- `python` - Python-specific conventions
- `react` - React framework conventions
- `fastapi` - FastAPI framework conventions
- `general` - Language-agnostic conventions

**Custom scopes:**
Accept any user-specified scope (e.g., `golang`, `vue`, `django`)

### Category Values

- `naming` - Naming conventions for code elements
- `rules` - Structural rules and constraints
- `docs` - Documentation standards
- `structure` - File/directory organization
- `patterns` - Code patterns and best practices

### Version Guidelines

- **1.0** - Initial convention
- **1.x** - Minor updates (additions, clarifications, examples)
- **2.0+** - Major updates (breaking changes, rewrites)

## Content Sections

### Comprehensive Tier (150-300 lines)

**Required sections:**
1. Title (H1)
2. Description (2-3 paragraphs with context)
3. Format section with 4+ bullet points
4. Multiple domain-specific sections with subsections
5. Allowed section with 4+ examples
6. Forbidden section with 4+ anti-patterns
7. Rules section with 7+ numbered guidelines
8. Summary table

**Optional sections:**
- Additional context sections
- Edge cases
- Common mistakes

**Example structure:**
```markdown
# {Title}

{Comprehensive description with context and rationale}

## Format

- **{Concept 1}**: {Explanation}
- **{Concept 2}**: {Explanation}
- **{Concept 3}**: {Explanation}
- **{Concept 4}**: {Explanation}

## {Domain Section 1}

### {Subsection A}

{Guidelines and explanations}

### {Subsection B}

{Guidelines and explanations}

## {Domain Section 2}

{More detailed content}

## Allowed

```{language}
// Example 1: {Scenario}
{code}
```

[Multiple examples...]

## Forbidden

```{language}
// Anti-pattern 1: {Issue}
{code}
```

[Multiple anti-patterns...]

## Rules

1. **{Principle 1}**: {Detailed explanation with rationale}
2. **{Principle 2}**: {Detailed explanation with rationale}
[7+ rules total...]

## Summary

| Aspect | Requirement | Example |
|--------|-------------|---------|
| {Aspect 1} | {Requirement} | `{code}` |
| {Aspect 2} | {Requirement} | `{code}` |
```

## Code Block Guidelines

### Language Tags

Always specify language for syntax highlighting:
- TypeScript: ```typescript or ```ts
- Python: ```python or ```py
- JavaScript: ```javascript or ```js
- React: ```tsx or ```jsx
- Shell: ```bash or ```sh
- General: ```text

### Code Examples

**Good examples (Allowed section):**
- Show realistic, production-quality code
- Include comments explaining why it's good
- Demonstrate the principle clearly
- Use meaningful variable/function names

**Bad examples (Forbidden section):**
- Show common mistakes or anti-patterns
- Include comments explaining the problem
- Make the issue obvious
- Use similar context to good examples for comparison

## Description Guidelines

A good description should:
- Explain WHAT the convention covers
- Explain WHY it matters (context, rationale)
- Mention when it applies vs. exceptions
- Reference the project context if relevant

**Example:**
```markdown
# TypeScript Function Naming

This convention defines naming standards for TypeScript functions and methods.
Consistent naming improves code readability and helps AI assistants understand
code intent. This project emphasizes descriptive, verb-based names that clearly
communicate function purpose.
```

## Validation Checklist

Before finalizing a convention file:

- [ ] YAML front matter is valid and complete
- [ ] Version number follows semantic versioning
- [ ] Scope and category are appropriate
- [ ] 3-5 relevant tags are included
- [ ] Title is clear and concise (H1)
- [ ] Description provides context and rationale
- [ ] Format section has appropriate bullet points for tier
- [ ] Code examples include language tags
- [ ] Good and bad examples are realistic
- [ ] All relative paths are correct
- [ ] Line count is within tier range (±20% tolerance)
- [ ] No duplicate conventions exist for same topic
