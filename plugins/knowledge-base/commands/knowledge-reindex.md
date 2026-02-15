---
name: knowledge-reindex
description: Rebuild .kb/ index (README.md catalog)
argument-hint: "[--review]"
---

> **EXECUTION CONSTRAINT**: All code blocks below are pseudocode for reference only.
> - **NEVER** create or execute `.py` scripts
> - **NEVER** use Bash to run `python` or `python3` commands
> - Implement all logic using Claude's built-in tools: Read, Grep, Glob, Write, Edit, Bash, Task
> - Parse YAML/JSON content mentally from Read tool output — do NOT use Python yaml/json libraries
> - `invoke_skill()` / `invoke_agent()` in pseudocode = use **Task** tool with appropriate subagent_type

# Rebuild Knowledge Base Index

Scan all knowledge entries and update README.md catalog with organized listings.

## Usage

```bash
/knowledge-reindex           # Standard reindex only
/knowledge-reindex --review  # Reindex + Level 1 validation report
```

## Purpose

Maintains an up-to-date catalog of all knowledge entries in the `.kb/` directory, making it easy for users and AI assistants to discover and navigate knowledge entries.

## Workflow

### Step 1: Find All Convention Files

Use Glob tool to find all `.md` files:

```
pattern: .kb/**/*.md

Exclude:
- .kb/README.md (the index itself)
- .kb/TEMPLATE.md (template file)
- .kb/*/README.md (scope-level READMEs)
```

### Step 2: Extract Metadata

For each found file, extract information using Read tool:

For each file, use **Read** to load its content, then parse mentally:

- **YAML front matter**: Identify text between the first `---` and the second `---`, extract fields: type, version, scope, category, tags
- **Title**: Find the first H1 heading (`# Title`)
- **Description**: Get the first paragraph after the title (optional, for preview)

**Build data structure:**
```yaml
convention:
  path: "{relative path from .kb}"
  title: "{extracted title}"
  version: "{frontmatter version}"
  scope: "{frontmatter scope}"
  category: "{frontmatter category}"
  tags: "{frontmatter tags}"
```

### Step 3: Organize Conventions

Group knowledge entries by scope, then by category:

Group all extracted convention data by scope, then by category within each scope. For each convention, place it under `organized[scope][category]` as a list.

Sort:
- Scopes alphabetically (but `general` last)
- Categories alphabetically within scope
- Conventions alphabetically by title within category

### Step 3.5: Generate Category READMEs

For each category in the organized structure, generate a detailed README.md:

**For each scope and category:**

1. **Extract description excerpt** from each knowledge entry:
   - Skip YAML frontmatter (between `---` markers)
   - Find first paragraph after H1 title
   - Truncate to ~200 characters for preview

2. **Generate category README content:**
   ```markdown
   # {Category} Conventions

   **Parent:** [← {Scope}](../README.md) | [← All Conventions](../../README.md)
   **Total knowledge entries:** {count}

   ## Conventions in this Category

   ### [{Title}]({filename})
   **Version:** {version}
   **Tags:** {tags}

   {description_excerpt}

   ---

   [Repeat for each convention in category]

   ## Statistics
   - Total knowledge entries: {count}
   - Common tags: {top_tags_in_category}
   ```

3. **Write to file:**
   - Path: `.kb/{scope}/{category}/README.md`
   - Use Write tool to create file

**Example output:** `.kb/typescript/naming/README.md`

### Step 3.6: Generate Scope READMEs

For each scope in the organized structure, generate a summary README.md:

**For each scope:**

1. **Calculate totals** per category within scope

2. **Select featured knowledge entries** (top 3-5 per category):
   - Sort alphabetically by title
   - Take first 3-5 knowledge entries

3. **Generate scope README content:**
   ```markdown
   # {Scope} Conventions

   **Parent:** [← All Conventions](../README.md)
   **Total knowledge entries:** {total_in_scope}
   **Categories:** {category_count}

   ## Categories

   ### [{Category}]({category}/README.md)
   **Conventions:** {count_in_category}

   Featured knowledge entries:
   - [{title}]({category}/{filename}) `v{version}`
   - [{title}]({category}/{filename}) `v{version}`
   - [{title}]({category}/{filename}) `v{version}`
   - ... and {remaining_count} more

   ---

   [Repeat for each category in scope]

   ## Statistics
   - Total knowledge entries: {total_in_scope}
   - Categories: {category_names_with_counts}
   - Top tags: {top_tags_in_scope}
   ```

4. **Write to file:**
   - Path: `.kb/{scope}/README.md`
   - Use Write tool to create file

**Example output:** `.kb/typescript/README.md`

### Step 4: Generate Root README.md (Modified)

Build markdown content with **summary format** (less detail, more aggregation):

**Header:**
```markdown
# Conventions Directory

Standardized coding knowledge entries and guidelines for AI assistants and developers.

**Last updated:** {current_date}
**Total knowledge entries:** {total_count}

---
```

**Directory Structure with Links:**
```markdown
## Directory Structure

```
.kb/
└── typescript/              TypeScript knowledge entries (17) → [View Details](typescript/README.md)
    ├── naming/             Naming knowledge entries (15) → [View Details](typescript/naming/README.md)
    └── rules/              Structural rules (2) → [View Details](typescript/rules/README.md)
```
```

**Important changes:**
- Add convention counts next to each directory/category
- Add `→ [View Details](path/to/README.md)` links to scope and category READMEs

**Conventions by Scope (Summary Format):**
```markdown
## Conventions by Scope

### typescript → [View All](typescript/README.md)

**Categories:** naming (15), rules (2)

**Featured knowledge entries:**
- [Function Naming](typescript/naming/functions.md) `v1.0`
- [Class Naming](typescript/naming/classes.md) `v1.0`
- [File Size](typescript/rules/file-size.md) `v1.0`

[→ View all 17 TypeScript knowledge entries](typescript/README.md)

---

{Repeat for each scope}
```

**Important changes:**
- Show only 3 featured knowledge entries per scope (top 3 alphabetically)
- Remove detailed descriptions and tags (keep summary only)
- Add link to scope README for full details
- Show category names with counts in summary

**Statistics Section:**
```markdown
## Statistics

- **Total knowledge entries**: {total_count}
- **Scopes**: {scope_count}
- **Categories**: {category_count}
- **Most common tags**: {top_5_tags}
```

### Step 5: Write All README Files

Use Write tool to create/overwrite all README files:

**Files to generate:**
1. **Category READMEs** (from Step 3.5):
   - `.kb/{scope}/{category}/README.md` for each category
   - Example: `.kb/typescript/naming/README.md`
   - Example: `.kb/typescript/rules/README.md`

2. **Scope READMEs** (from Step 3.6):
   - `.kb/{scope}/README.md` for each scope
   - Example: `.kb/typescript/README.md`

3. **Root README** (from Step 4):
   - `.kb/README.md`

**Write order:**
1. Category READMEs first (bottom-up approach)
2. Scope READMEs second
3. Root README last

**Report generated files:**
```
✓ Generated category READMEs:
  - .kb/typescript/naming/README.md (15 knowledge entries)
  - .kb/typescript/rules/README.md (2 knowledge entries)

✓ Generated scope READMEs:
  - .kb/typescript/README.md (17 knowledge entries)

✓ Generated root README:
  - .kb/README.md (17 total knowledge entries)
```

## Output Format

### Example: Root README (Summary Format)

```markdown
# Conventions Directory

Standardized coding knowledge entries and guidelines for AI assistants and developers.

**Last updated:** 2026-02-14
**Total knowledge entries:** 17

---

## Directory Structure

```
.kb/
└── typescript/              TypeScript knowledge entries (17) → [View Details](typescript/README.md)
    ├── naming/             Naming knowledge entries (15) → [View Details](typescript/naming/README.md)
    └── rules/              Structural rules (2) → [View Details](typescript/rules/README.md)
```

## Conventions by Scope

### typescript → [View All](typescript/README.md)

**Categories:** naming (15), rules (2)

**Featured knowledge entries:**
- [Function Naming Conventions](typescript/naming/functions.md) `v1.0`
- [Class Naming](typescript/naming/classes.md) `v1.0`
- [File Size Convention](typescript/rules/file-size.md) `v1.0`

[→ View all 17 TypeScript knowledge entries](typescript/README.md)

---

## Statistics

- **Total knowledge entries**: 17
- **Scopes**: 1 (typescript)
- **Categories**: 2 (naming, rules)
- **Most common tags**: typescript (9), PascalCase (7), camelCase (6), kebab-case (5), naming (4)

## Quick Reference by Tag

**Case Conventions:**
- [#PascalCase](typescript/naming/classes.md) - 7 knowledge entries
- [#camelCase](typescript/naming/functions.md) - 6 knowledge entries
- [#kebab-case](typescript/naming/files.md) - 5 knowledge entries

**TypeScript Specific:**
- [#typescript](typescript/naming/types.md) - 9 knowledge entries
- [#react](typescript/naming/react-components.md) - 4 knowledge entries

---

*Generated by knowledge entries plugin. Run `/knowledge entries:convention-rebuild` to update.*
```

### Example: Scope README (.kb/typescript/README.md)

```markdown
# TypeScript Conventions

**Parent:** [← All Conventions](../README.md)
**Total knowledge entries:** 17
**Categories:** 2

## Categories

### [Naming](naming/README.md)
**Conventions:** 15

Featured knowledge entries:
- [Function Naming Conventions](naming/functions.md) `v1.0`
- [Class Naming](naming/classes.md) `v1.0`
- [Variable Naming](naming/variables.md) `v1.0`
- [Type and Interface Naming](naming/types.md) `v1.0`
- [React Component Naming](naming/react-components.md) `v1.0`
- ... and 10 more

---

### [Rules](rules/README.md)
**Conventions:** 2

Featured knowledge entries:
- [File Size Convention](rules/file-size.md) `v1.0`
- [Import Path Convention](rules/import-paths.md) `v1.0`

---

## Statistics

- **Total knowledge entries**: 17
- **Categories**: naming (15), rules (2)
- **Top tags**: typescript (9), PascalCase (7), camelCase (6), kebab-case (5), naming (4)
```

### Example: Category README (.kb/typescript/naming/README.md)

```markdown
# Naming Conventions

**Parent:** [← TypeScript Conventions](../README.md) | [← All Conventions](../../README.md)
**Total knowledge entries:** 15

## Conventions in this Category

### [Function Naming Conventions](functions.md)
**Version:** 1.0
**Tags:** functions, methods, camelCase, verbs, predicates, async

Standardized naming knowledge entries for functions and methods in TypeScript applications.

---

### [Class Naming](classes.md)
**Version:** 1.0
**Tags:** classes, PascalCase, domain-suffixes, nouns, architecture

TypeScript class and constructor naming with PascalCase and domain suffixes.

---

### [Variable Naming Conventions](variables.md)
**Version:** 1.0
**Tags:** variables, camelCase, typescript, semantic-naming, booleans, collections

Naming for variables with semantic clarity.

---

[... 12 more knowledge entries ...]

---

## Statistics

- **Total knowledge entries**: 15
- **Common tags**: PascalCase (7), camelCase (6), typescript (9), react (4)
```

## Error Handling

**No knowledge entries found:**
```markdown
# Conventions Directory

No knowledge entries found. Create your first convention using `/convention`.

Example:
```
/convention TypeScript functions should use camelCase
```

For more information, run `/convention` without arguments.
```

**Parse errors:**
```
If YAML front matter is invalid in a file:
- Log warning: "Skipping {file}: Invalid YAML front matter"
- Continue processing other files
- Include error summary at end of README
```

**Missing .knowledge entries directory:**
```
Create directory first:
mkdir -p .knowledge entries

Then create empty README.md
```

## Performance

- **Batch reads**: Read all files in parallel if possible
- **Caching**: Consider caching parsed metadata (future enhancement)
- **Limit**: No limit on number of knowledge entries (scales to hundreds)

## Validation

After generation, verify:
- [ ] All README files created/updated successfully:
  - [ ] Root README (`.kb/README.md`) with summary format
  - [ ] Scope READMEs (`.kb/{scope}/README.md`)
  - [ ] Category READMEs (`.kb/{scope}/{category}/README.md`)
- [ ] All knowledge entrys were processed
- [ ] Scopes are correctly grouped
- [ ] Categories are alphabetically sorted within scopes
- [ ] Navigation links work correctly:
  - [ ] Root → Scope links
  - [ ] Scope → Category links
  - [ ] Category → Parent (Scope and Root) links
- [ ] Paths are correct relative links
- [ ] Statistics are accurate at all levels
- [ ] Content hierarchy correct:
  - [ ] Root: minimal detail, summary with 3 featured knowledge entries per scope
  - [ ] Scope: medium detail, category previews with 3-5 knowledge entries each
  - [ ] Category: full detail, all knowledge entries listed
- [ ] No parse errors (or errors logged)

## Integration

**PostToolUse hook** reminds users to run this command after creating/editing knowledge entries:
```
✓ Convention file updated. Run /knowledge-reindex to update catalog.
```

**Manual invocation:**
```bash
/knowledge-reindex
```

**Expected output:**
```
Scanning knowledge entries directory...
Found 17 knowledge entrys

Organizing by scope and category...
- typescript: 17 knowledge entries (naming: 15, rules: 2)

Generating category READMEs...
✓ .kb/typescript/naming/README.md (15 knowledge entries)
✓ .kb/typescript/rules/README.md (2 knowledge entries)

Generating scope READMEs...
✓ .kb/typescript/README.md (17 knowledge entries)

Generating root README...
✓ .kb/README.md (summary with links)

Summary:
- Total knowledge entries: 17
- Scopes: 1
- Categories: 2
- README files generated: 4 (1 root + 1 scope + 2 category)
```

## Review Mode (--review flag)

When invoked with `--review` flag, the command performs additional validation after reindexing:

### Workflow Extension

```
/knowledge-reindex --review
  ↓
1. Scan + generate README files (standard behavior)
  ↓
2. Invoke knowledge-reviewer agent in report-only mode
   Parameters: { path: "", fix_mode: "none" }
  ↓
3. Display summary after index:
```

### Example Output

```
Scanning knowledge entries directory...
Found 17 knowledge entries

Organizing by scope and category...
- typescript: 17 knowledge entries (naming: 15, rules: 2)

Generating category READMEs...
✓ .kb/typescript/naming/README.md (15 knowledge entries)
✓ .kb/typescript/rules/README.md (2 knowledge entries)

Generating scope READMEs...
✓ .kb/typescript/README.md (17 knowledge entries)

Generating root README...
✓ .kb/README.md (summary with links)

📋 Quick Review (Level 1 validation):
  ❌ 2 files have structural issues:
    • python/naming/modules.md — missing field: tags
    • react/patterns/hooks.md — invalid YAML syntax

  Run /knowledge-review --fix to auto-fix

Summary:
- Total knowledge entries: 17
- Scopes: 1
- Categories: 2
- README files generated: 4 (1 root + 1 scope + 2 category)
```

### Purpose

Provides lightweight quality check during routine indexing without requiring separate command. Useful for:
- **CI/CD pipelines**: Catch structural issues during automated builds
- **Weekly maintenance**: Quick health check of entire knowledge base
- **After batch imports**: Validate multiple new entries at once

### Performance Impact

- Standard reindex: ~1-2 seconds
- With --review: +10-20 seconds (for 50+ entries)
- Review uses Level 1 validation only (fast structural checks)
- No auto-fixes applied (report-only mode)

## Notes

- **Idempotent**: Can be run multiple times safely
- **No confirmation**: Overwrites README.md without asking
- **Fast operation**: Should complete in <2 seconds for 100s of knowledge entries
- **Auto-discovery**: Automatically finds all `.md` files in directory structure
- **Review mode**: Optional --review flag adds validation without modifying files
