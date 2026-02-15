# Conventions Plugin - Usage Guide

Complete guide with practical examples for using the Conventions Plugin.

## Quick Start Examples

### Create Your First Convention

**Simple file size rule:**
```bash
/convention TypeScript files should not exceed 800 lines
```

**Output:**
```
✓ Convention created: .conventions/typescript/rules/file-size.md
Version: 1.0
Lines: 45 lines, 3 sections, 2 examples, simple tier
Quality: 7.2/10 (Clarity: 7, Format: 8, Structure: 7, Completeness: 7, Efficiency: 7)

Run /convention-rebuild to update the conventions catalog.
```

### Standard Convention with Multiple Rules

**Function naming convention:**
```bash
/convention TypeScript functions: camelCase, verb-based, descriptive, max 3 words, boolean functions start with is/has/should
```

**Output:**
```
✓ Convention created: .conventions/typescript/naming/functions.md
Version: 1.0
Lines: 95 lines, 6 sections, 6 examples, standard tier
Quality: 8.4/10 (Clarity: 8, Format: 9, Structure: 9, Completeness: 8, Efficiency: 8)

Run /convention-rebuild to update the conventions catalog.
```

### Interactive Mode for Complex Conventions

**For detailed conventions with 7+ rules:**
```bash
/convention interactive
```

**You'll be prompted for:**
```
Topic: Error Handling
Scope: TypeScript
Category: patterns
Guidelines (multiline):
> try-catch blocks around async operations
> custom Error classes extending Error
> log errors with context
> user-friendly error messages
>
Tags: error, exceptions, async, logging
```

**Output:**
```
✓ Convention created: .conventions/typescript/patterns/error-handling.md
Version: 1.0
Lines: 220 lines, 10 sections, 8 examples, comprehensive tier
Quality: 8.7/10 (Clarity: 9, Format: 9, Structure: 9, Completeness: 8, Efficiency: 8)
```

---

## Command Reference

### `/knowledge-scan` - Discover Conventions

Automatically discover de-facto conventions from existing codebase.

#### Syntax

```bash
/knowledge-scan [directory] [--auto]
```

#### Arguments

- `directory` (optional) - Directory to scan (default: current working directory)
- `--auto` (optional) - Auto-generate high-confidence conventions (≥80%)

#### Workflow

**Step 1: Discovery Mode**
```bash
/knowledge-scan src/
```

Analyzes the codebase and presents an interactive plan:

```
🔍 Scanning codebase in src/...

Detected stack:
- TypeScript (156 files)
- React (package.json)

📊 Discovery Plan:

[1] typescript/naming/functions - 94% confidence
    Pattern: camelCase + verb-prefix + async naming
    Evidence: 156 matching files
    Examples: getUserData, fetchApiResponse, handleClick

[2] typescript/naming/variables - 87% confidence
    Pattern: camelCase, descriptive, >3 chars
    Evidence: 142 matching files

[3] typescript/structure/imports - 76% confidence
    Pattern: absolute imports, grouped by source
    Evidence: 98 matching files

⚠ [4] react/patterns/hooks - 58% confidence (low)
    Pattern: useState for local, Context for shared
    Evidence: 23 matching files

✓ High confidence (≥80%): 2 conventions
✓ Medium confidence (60-79%): 1 convention
⚠ Low confidence (<60%): 1 convention

What would you like to do?
1. Generate high-confidence conventions (1-2)
2. Select specific conventions to generate
3. Generate all conventions
4. Cancel
```

**Step 2: Generate Selected Conventions**
```bash
# Generate conventions 1 and 3
/knowledge-scan generate 1,3
```

**Or use auto mode:**
```bash
# Automatically generate all high-confidence (≥80%) conventions
/knowledge-scan src/ --auto
```

#### Conflict Resolution

When a discovered convention conflicts with existing entry:

```
⚠ Conflict detected: .kb/typescript/naming/functions.md (v1.2)

Existing:
  Rules: camelCase, verb-based, max 3 words
  Examples: 6 code blocks

Discovered (94% confidence, 156 files):
  Additional patterns: async naming rules, arrow function patterns
  New evidence: 12 code samples from codebase

Options:
[update]  Merge new patterns into existing entry (v1.2 → v1.3)
[skip]    Keep existing entry unchanged
[replace] Full overwrite with discovered patterns (v1.2 → v2.0, requires confirmation)

Your choice:
```

#### Discovery Dimensions

The scanner analyzes multiple dimensions based on detected stack:

**General (always)**
- File naming conventions
- Directory structure patterns

**TypeScript/JavaScript**
- Function naming (camelCase, verb-prefix, etc.)
- Class naming (PascalCase, etc.)
- Import/export patterns
- Error handling patterns
- JSDoc documentation style

**React**
- Component naming
- Hook usage patterns
- State management patterns

**Python**
- Function naming (snake_case, etc.)
- Class naming (PascalCase, etc.)
- Module structure
- Docstring patterns
- Error handling

#### Confidence Scores

- **≥80%**: High confidence → auto-generated in `--auto` mode
- **60-79%**: Medium confidence → included in plan, manual selection
- **<60%**: Low confidence → shown but flagged as uncertain

Confidence = dominant_pattern_percentage × sample_size_factor × clarity_bonus

#### Examples

**Example 1: Full workflow**
```bash
# Discover conventions
/knowledge-scan src/

# Review plan, select conventions 1, 2, and 5
/knowledge-scan generate 1,2,5

# Update catalog
/knowledge-reindex
```

**Example 2: Auto mode**
```bash
# Automatically generate high-confidence conventions
/knowledge-scan src/ --auto

# Check results
cat .kb/README.md
```

**Example 3: Specific directory**
```bash
# Scan only API routes
/knowledge-scan src/api/routes/

# Scan components
/knowledge-scan src/components/
```

---

### `/knowledge-add` - Create Convention

Create a standardized convention file from raw guidelines.

#### Syntax

```bash
/convention [--path PATH] <guidelines|interactive>
```

#### Arguments

- `--path PATH` (optional) - Custom output path
- `guidelines` - Raw coding guidelines (text)
- `interactive` - Launch interactive mode with prompts

#### Examples

**1. Direct input (simple):**
```bash
/convention TypeScript async functions must have try-catch blocks
```
Creates: `.conventions/typescript/rules/async-error-handling.md` (simple tier)

**2. Direct input (standard):**
```bash
/convention TypeScript variable naming: camelCase for locals, UPPER_CASE for constants, descriptive names, avoid abbreviations
```
Creates: `.conventions/typescript/naming/variables.md` (standard tier)

**3. Custom path:**
```bash
/convention --path .conventions/react/hooks/naming.md React hooks: start with use, camelCase, descriptive
```
Creates: `.conventions/react/hooks/naming.md` at specified path

**4. Interactive mode:**
```bash
/convention interactive
```
Guides you through creating a comprehensive convention

**5. Multi-language input (Russian):**
```bash
/convention Правила обработки ошибок: try-catch, логирование, custom классы
```
Detects Russian input, creates English output

---

### `/convention-rebuild` - Update Catalog

Rebuild the conventions catalog (`.conventions/README.md`).

#### Syntax

```bash
/convention-rebuild
```

#### What it does

1. Scans all `.conventions/**/*.md` files
2. Extracts metadata (YAML frontmatter) and titles
3. Groups by scope and category
4. Generates organized README.md
5. Calculates statistics

#### Output

```
Scanning conventions directory...
Found 12 convention files

Organizing by scope and category...
- typescript: 8 conventions
- python: 3 conventions
- general: 1 convention

✓ Index rebuilt: .conventions/README.md

Summary:
- Total conventions: 12
- Scopes: 3
- Categories: 4
```

**Generated catalog structure:**
```markdown
# Coding Conventions

## TypeScript

### naming
- [Function Naming](typescript/naming/functions.md) - v1.0
- [Variable Naming](typescript/naming/variables.md) - v1.0

### rules
- [File Size Limit](typescript/rules/file-size.md) - v1.0

### patterns
- [Error Handling](typescript/patterns/error-handling.md) - v1.0

## Python

### naming
- [Module Naming](python/naming/modules.md) - v1.0

---

**Statistics:**
- Total Conventions: 12
- Scopes: 3
- Categories: 4
```

---

### `/convention-report` - Quality Evaluation

Evaluate convention quality with detailed analysis and recommendations.

#### Syntax

```bash
/convention-report [file-path or convention-name]
```

#### Arguments

- `file-path` (optional) - Full path, partial path, or name only
- If omitted, evaluates most recently modified convention

#### Path Resolution Examples

**1. Full path:**
```bash
/convention-report .conventions/typescript/naming/functions.md
```

**2. Partial path (from .conventions/):**
```bash
/convention-report typescript/naming/functions.md
```

**3. Name only:**
```bash
/convention-report functions.md
# or
/convention-report functions
```

**4. Most recent convention:**
```bash
/convention-report
```

#### Example Report

```markdown
# Convention Quality Report

**File**: .conventions/typescript/naming/functions.md
**Version**: 1.0
**Scope**: typescript | **Category**: naming
**Lines**: 503 | **Evaluated**: 2026-02-14

---

## Overall Score: 8.4/10 ⭐⭐⭐⭐

Grade: Excellent

This convention meets comprehensive quality standards with minor areas for improvement.

---

## Criteria Breakdown

### 1. Однозначность (Clarity): 8/10

**Strengths**:
- ✓ Excellent Allowed section (6 examples)
- ✓ Strong Forbidden section (5 examples)
- ✓ Comprehensive rules (10 numbered rules)

**Issues**:
- ⚠ Rules 3 and 7 use "should" instead of "must"
- ⚠ Could add more inline code examples

**Recommendations**:
1. Replace "should" with "must" in rules requiring strict compliance → +0.5 points
2. Add 2-3 more edge case examples → +1 point

---

### 2. Формат (Format): 9/10

**Strengths**:
- ✓ Complete YAML frontmatter
- ✓ Clear Format section (5 bullets)
- ✓ All code blocks properly tagged

**Issues**:
- ⚠ Summary table has 4 rows (target: 5+)

**Recommendations**:
1. Add row for "Async Functions" in summary table → +0.5 points

---

[... additional criteria ...]

---

## Summary

**Top Strengths**:
- Excellent structure and organization
- Comprehensive code examples
- Clear formatting with visual markers

**Priority Improvements**:
1. Replace "should" with "must" in 2 rules → +0.5 points
2. Add 2-3 edge case examples → +1 point
3. Simplify complex sentences → +0.5 points

**Potential Score After Improvements**: 9.4/10

---

**Next Steps**:
1. Apply recommendations above
2. Increment version to 1.1 after updates
3. Run /convention-rebuild to update catalog
```

---

## Usage Patterns

### Workflow 0: Automatic Discovery (New!)

For discovering conventions from existing code:

```bash
# 1. Scan codebase
/knowledge-scan src/

# 2. Review discovery plan
# (Interactive plan shown with confidence scores)

# 3. Generate selected conventions
/knowledge-scan generate 1,3,5

# 4. Update catalog
/knowledge-reindex
```

**Use when:**
- Starting new project documentation
- Discovering existing team patterns
- Documenting legacy codebase conventions
- Validating manual conventions against actual code

**Advantages:**
- Evidence-based: conventions derived from actual code
- Confidence scores guide decision-making
- Conflict detection prevents duplicates
- Batch generation saves time

---

### Workflow 1: Quick Convention

For simple, one-off rules:

```bash
# 1. Create convention
/convention TypeScript: avoid 'any' type

# 2. Update catalog
/convention-rebuild
```

**Use when:**
- Single rule or constraint
- Well-defined guideline
- No research needed

---

### Workflow 2: Standard Convention

For established patterns with multiple rules:

```bash
# 1. Create with multiple rules
/convention TypeScript error handling: try-catch for async, custom Error classes, log with context, user-friendly messages

# 2. Review quality
# (Automatic quality score shown)

# 3. If quality < 7.0, get detailed report
/convention-report error-handling

# 4. Update catalog
/convention-rebuild
```

**Use when:**
- 3-6 related rules
- Established patterns
- Need examples for clarity

---

### Workflow 3: Comprehensive Convention

For complex, multi-faceted conventions:

```bash
# 1. Use interactive mode
/convention interactive

# 2. Provide detailed input:
Topic: API Design Patterns
Scope: TypeScript
Category: patterns
Guidelines:
> RESTful endpoint structure: /api/v1/{resource}
> HTTP verbs: GET (read), POST (create), PUT (update), DELETE (remove)
> Status codes: 200 (OK), 201 (Created), 400 (Bad Request), 404 (Not Found), 500 (Server Error)
> Request validation with Joi schemas
> Response format: { data, error, meta }
> Authentication with JWT tokens
> Rate limiting: 100 requests/minute
> Error handling with custom error classes
> Logging all requests with correlation IDs
> Versioning in URL path
>
Tags: api, rest, design, patterns, validation, auth

# 3. Review generated convention
cat .conventions/typescript/patterns/api-design.md

# 4. Check quality
/convention-report api-design

# 5. Update catalog
/convention-rebuild
```

**Use when:**
- 7+ complex rules
- Multiple domains/aspects
- Need subsections
- Cross-references helpful

---

### Workflow 4: Research-Based Convention

When you need best practices from codebase:

```bash
# 1. Request with "best practices" trigger
/convention TypeScript error handling best practices

# 2. Plugin automatically:
#    - Searches codebase for try-catch patterns
#    - Checks tsconfig.json settings
#    - Looks for existing error classes
#    - Finds linter rules

# 3. Review generated convention (includes research)
cat .conventions/typescript/patterns/error-handling.md

# 4. Update catalog
/convention-rebuild
```

**Research triggers:**
- "best practices"
- "industry standard"
- "recommended approach"

---

## Convention Tiers Explained

The plugin automatically selects the appropriate tier based on input complexity.

### Simple Tier (20-50 lines)

**Best for:** Single rule or simple constraint

**Structure:**
- Title
- Description (brief context)
- Format section (1-2 bullets)
- Allowed examples (1-2)

**Example input:**
```bash
/convention TypeScript files: max 800 lines
```

**Generated sections:**
```markdown
# File Size Limit

TypeScript files should not exceed 800 lines...

## Format
- **Maximum lines**: 800 lines per file

## Allowed
```typescript
// Focused component (350 lines)
export class UserService { ... }
```
```

---

### Standard Tier (50-150 lines)

**Best for:** 3-6 related rules with examples

**Structure:**
- Title
- Description (detailed context)
- Format section (3-5 bullets)
- Allowed examples (3-5)
- Forbidden examples (3-5)
- Rules section (numbered list)

**Example input:**
```bash
/convention TypeScript functions: camelCase, verb-based, descriptive, max 3 words
```

**Generated sections:**
```markdown
# Function Naming

TypeScript functions should use camelCase...

## Format
- **Case**: camelCase
- **Structure**: verb + noun
- **Length**: Maximum 3 words

## Allowed
[3-5 good examples]

## Forbidden
[3-5 anti-patterns]

## Rules
1. Use verbs: Start with action words...
2. Be descriptive: ...
3. Max 3 words: ...
```

---

### Comprehensive Tier (150-300 lines)

**Best for:** 7+ complex rules across multiple domains

**Structure:**
- Title
- Description (extensive context)
- Format section (5+ bullets)
- Summary table
- Multiple domain subsections
- Allowed/Forbidden for each domain
- Rules section (10+ rules)
- Cross-references
- Edge cases

**Example input:**
```bash
/convention interactive
# Then provide 7+ detailed guidelines
```

**Generated sections:**
```markdown
# Error Handling Patterns

Comprehensive error handling for TypeScript applications...

## Format
- **Try-Catch**: Required for all async operations
- **Error Classes**: Custom classes extending Error
- **Logging**: Context-aware error logging
- **User Messages**: User-friendly error messages
- **Status Codes**: HTTP status codes for APIs

## Summary

| Pattern | Use Case | Example |
|---------|----------|---------|
| Try-Catch | Async operations | `try { await api() } catch (e) { ... }` |
| Custom Error | Domain errors | `class ValidationError extends Error` |
| ...

## Async Operations

### Allowed
[Examples]

### Forbidden
[Anti-patterns]

## Error Classes

### Allowed
[Examples]

### Forbidden
[Anti-patterns]

## Rules
1. **Always use try-catch**: Wrap all async operations...
2. **Create custom error classes**: Extend Error for domain errors...
[10+ detailed rules]

## Related Conventions
- [Logging Patterns](../patterns/logging.md)
- [API Error Handling](../api/errors.md)
```

---

## Best Practices

### Writing Effective Guidelines

**Good input structure:**
```
<Topic>: <Rule1>, <Rule2>, <Rule3>
```

**Examples:**

✅ **Good:**
```
TypeScript functions: camelCase, verb-based, max 3 words, descriptive
```

✅ **Better (with details):**
```
TypeScript Function Naming:

Rules:
1. Use camelCase (getUserData, not GetUserData)
2. Start with verbs (get, set, create, update, delete)
3. Maximum 3 words preferred
4. Boolean functions: is/has/should prefix
5. Async operations: include context suffix

Examples:
Good: calculateTotalPrice(items)
Bad: process(data)
```

❌ **Too vague:**
```
use good names
```

❌ **Too specific (should be example, not guideline):**
```
the function getUserById should return a user object
```

---

### When to Create Conventions

**✅ Create conventions for:**
- Recurring patterns in your codebase
- Team agreements on code style
- Project-specific standards
- Framework-specific patterns
- Best practices you want to enforce
- Decisions made during code reviews
- Common mistakes to prevent

**❌ Don't create conventions for:**
- One-off decisions
- Obvious best practices (already in linters)
- Personal preferences (unless team-agreed)
- Framework defaults (already documented)
- Temporary workarounds

---

### Organizing Conventions

#### By Scope

Create a scope for each major language/framework:

```
.conventions/
├── typescript/     # TypeScript-specific
├── python/         # Python-specific
├── react/          # React framework
├── fastapi/        # FastAPI framework
├── nodejs/         # Node.js specific
└── general/        # Language-agnostic
```

**Use `general` for:**
- Git commit messages
- PR descriptions
- Documentation standards
- File naming (cross-language)
- Code review guidelines

#### By Category

Organize within scopes by category:

- **`naming/`** - How things are named
  - Functions, variables, classes, files, etc.

- **`rules/`** - What's allowed/forbidden
  - File size limits, complexity limits, dependency rules

- **`structure/`** - How code is organized
  - Directory structure, file organization, module boundaries

- **`patterns/`** - How to implement features
  - Error handling, state management, API design

- **`docs/`** - How to document
  - Comments, JSDoc, README structure

---

### Versioning Conventions

Update version when editing existing conventions:

- **1.0** - Initial convention
- **1.1, 1.2, 1.3** - Minor updates:
  - Add examples
  - Clarify rules
  - Fix typos
  - Add edge cases

- **2.0** - Major changes:
  - Breaking changes
  - Complete rewrites
  - Fundamental rule changes

**Example:**
```bash
# Edit existing convention
vim .conventions/typescript/naming/functions.md

# Update version in YAML frontmatter:
---
type: convention
version: 1.1  # was 1.0
...
---

# Rebuild index
/convention-rebuild
```

---

## Advanced Usage

### Custom Output Paths

Override auto-generated path structure:

```bash
/convention --path .conventions/custom/special-rules.md <guidelines>
```

**Valid paths:**
- Must end with `.md`
- Within `.conventions/` (recommended)
- No `..` (security)

**Example:**
```bash
/convention --path .conventions/project-specific/api-versioning.md API versioning: use /api/v1, /api/v2 in URL path
```

---

### Russian Input Support

Write guidelines in Russian, get English output:

```bash
/convention Функции TypeScript: camelCase, глаголы, максимум 3 слова
```

**Output:** English convention file with proper structure

**Auto-detection:**
- Detects Cyrillic characters
- Processes correctly
- Always outputs in English
- Stores language preference in agent memory

---

### Triggering Codebase Research

Automatically research codebase patterns:

**Triggers:**
- "best practices"
- "industry standard"
- "recommended approach"

**Example:**
```bash
/convention TypeScript error handling best practices
```

**What happens:**
1. Searches codebase for try-catch patterns
2. Checks tsconfig.json for strict mode
3. Looks for existing Error classes
4. Finds linter rules (ESLint, etc.)
5. Checks existing conventions
6. Merges findings with AI knowledge

**Research output:**
```
Researching codebase patterns...
Found 15 try-catch blocks in TypeScript files
Found tsconfig.json with strict mode enabled
Found 3 custom Error classes

✓ Convention created: .conventions/typescript/patterns/error-handling.md

Tier: comprehensive
Research: codebase patterns included
```

---

### Handling File Conflicts

When a convention file already exists:

```bash
/convention TypeScript variables: camelCase, descriptive
```

**If `.conventions/typescript/naming/variables.md` exists:**

```
⚠ File already exists: .conventions/typescript/naming/variables.md

Options:
1. Overwrite existing file
2. Rename new file (variables-v2.md)
3. Cancel

What would you like to do? [1/2/3]
```

**Choose:**
- **1 (Overwrite)** - Replace with new content (update version)
- **2 (Rename)** - Create `variables-v2.md`
- **3 (Cancel)** - Abort creation

---

## Common Scenarios

### Scenario 0: Documenting Existing Codebase (New!)

Automatically discover and document conventions from existing code:

```bash
# 1. Scan main source directory
/knowledge-scan src/

# Review discovery plan:
# [1] typescript/naming/functions - 94% confidence (156 files)
# [2] typescript/naming/variables - 87% confidence (142 files)
# [3] typescript/structure/imports - 76% confidence (98 files)
# [4] react/patterns/hooks - 58% confidence (23 files)

# 2. Generate high and medium confidence conventions
/knowledge-scan generate 1,2,3

# 3. Review generated entries
cat .kb/typescript/naming/functions.md
cat .kb/typescript/naming/variables.md
cat .kb/typescript/structure/imports.md

# 4. For low-confidence patterns, create manually with more context
/knowledge-add React state management: useState for local state, Context for shared state, avoid prop drilling >2 levels

# 5. Update catalog
/knowledge-reindex

# 6. Share with team
git add .kb/
git commit -m "docs: add discovered coding conventions"
```

**Benefits:**
- Fast initial documentation from existing patterns
- Evidence-based with real code examples
- Identifies inconsistencies (low confidence scores)
- Validates conventions against actual codebase

---

### Scenario 1: Team Onboarding

Create conventions for your team's coding standards:

```bash
# 1. TypeScript naming
/convention TypeScript naming: camelCase for variables/functions, PascalCase for classes/interfaces, UPPER_CASE for constants

# 2. File structure
/convention TypeScript file structure: one class per file, index.ts for re-exports, test files adjacent with .test.ts suffix

# 3. Error handling
/convention TypeScript error handling: try-catch for async, custom Error classes, log errors, user-friendly messages

# 4. Code review
/convention Code review guidelines: check tests, verify error handling, ensure types, review naming

# 5. Rebuild catalog
/convention-rebuild

# 6. Share .conventions/README.md with team
```

---

### Scenario 2: Migrating Existing Standards

Convert your existing coding standards document:

```bash
# For each section in your standards doc:

# Example: Function naming section
/convention interactive
# Topic: Function Naming
# Scope: TypeScript
# Category: naming
# Guidelines: [paste from standards doc]
# Tags: functions, naming, camelCase

# Repeat for each section
# Then rebuild
/convention-rebuild
```

---

### Scenario 3: Framework-Specific Patterns

Create React-specific conventions:

```bash
# Hooks naming
/convention React hooks: start with use, camelCase, descriptive, custom hooks in hooks/ directory

# Component structure
/convention React components: PascalCase, one component per file, props interface above component, default export

# State management
/convention React state: useState for local, Context for shared, no prop drilling >2 levels

# Rebuild
/convention-rebuild
```

---

### Scenario 4: Quality Improvement

Improve low-quality conventions:

```bash
# 1. Create initial convention (might be low quality)
/convention Use good variable names

# Output shows low score:
# Quality: 5.5/10
# ⚠ Quality below target. Run /convention-report for detailed analysis.

# 2. Get detailed feedback
/convention-report good-variable-names

# 3. Review recommendations and apply

# 4. Recreate with improvements
/convention TypeScript variable naming: camelCase for locals, UPPER_CASE for constants, descriptive names >3 chars, avoid abbreviations except common (id, url, api)

# Output shows improved score:
# Quality: 8.2/10
```

---

## Troubleshooting

### Convention Not Created

**Issue:** Command runs but no file created

**Solutions:**
- Provide more detailed guidelines (at least 2-3 rules)
- Try interactive mode for step-by-step guidance
- Check for error messages in output

**Example fix:**
```bash
# Too vague
/convention error handling
# ❌ Insufficient input

# Better
/convention TypeScript: use try-catch, handle all errors
# ✓ Creates convention
```

---

### Invalid YAML Error

**Issue:** Validation fails with YAML error

**Solution:**
- Plugin auto-generates YAML, this shouldn't happen
- If manually editing: check YAML syntax
- Ensure tags are array: `tags: [tag1, tag2]`

**Check YAML:**
```bash
cat .conventions/typescript/naming/functions.md | head -10
```

**Valid YAML:**
```yaml
---
type: convention
version: 1.0
scope: typescript
category: naming
tags: [functions, naming, camelCase]
---
```

---

### Index Not Updating

**Issue:** `/convention-rebuild` doesn't show new convention

**Solutions:**
- Verify convention file exists in `.conventions/`
- Check YAML front matter is valid
- Ensure file is not excluded (README.md, TEMPLATE.md)

**Debug:**
```bash
# List all conventions
find .conventions -name "*.md" -not -name "README.md" -not -name "TEMPLATE.md"

# Check specific file YAML
head -10 .conventions/path/to/file.md
```

---

### Quality Score Too Low

**Issue:** Convention quality < 7.0

**Solutions:**
1. Get detailed report:
   ```bash
   /convention-report your-convention
   ```

2. Apply recommendations:
   - Add more examples
   - Use "must" instead of "should"
   - Add Forbidden section
   - Structure with clear headings

3. Recreate or edit convention

4. Re-evaluate:
   ```bash
   /convention-report your-convention
   ```

---

## Tips & Tricks

### Tip 1: Use Templates

Save time by basing new conventions on existing ones:

```bash
# Create first convention
/convention TypeScript functions: camelCase, verb-based

# Review structure
cat .conventions/typescript/naming/functions.md

# Use as template for similar conventions
/convention TypeScript variables: camelCase, descriptive, avoid abbreviations
# Plugin will use similar structure
```

---

### Tip 2: Iterate on Quality

Start simple, improve over time:

```bash
# v1.0 - Simple start
/convention TypeScript: avoid any type

# v1.1 - Add examples
# (Edit file, add examples)

# v1.2 - Add exceptions
# (Edit file, add when any is acceptable)

# Rebuild after each update
/convention-rebuild
```

---

### Tip 3: Link Related Conventions

Create cross-references:

```markdown
## Related Conventions
- [Variable Naming](../naming/variables.md)
- [Error Handling](../patterns/error-handling.md)
```

Plugin detects these and validates paths.

---

### Tip 4: Use Tags Strategically

Tags help find related conventions:

```yaml
tags: [functions, naming, camelCase, verbs, async]
```

**Good tags:**
- Language features (async, promises, classes)
- Patterns (factory, singleton, observer)
- Concepts (naming, testing, errors)

**Avoid:**
- Too generic (code, programming)
- Too specific (getUserById)

---

## Next Steps

1. **Create your first convention:**
   ```bash
   /convention <your guideline>
   ```

2. **Review the generated file:**
   ```bash
   cat .conventions/<scope>/<category>/<name>.md
   ```

3. **Update the catalog:**
   ```bash
   /convention-rebuild
   ```

4. **Check catalog:**
   ```bash
   cat .conventions/README.md
   ```

5. **Share with team:**
   - Commit `.conventions/` directory
   - Include in PR reviews
   - Reference in documentation

---

**Need help?**
- See [README.md](README.md) for architecture and features
- See [TESTING.md](TESTING.md) for comprehensive test cases
- Run `/convention` without arguments for usage help
