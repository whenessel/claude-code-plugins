# Knowledge Base Plugin

A Claude Code plugin for generating standardized convention files from raw coding guidelines with AI-driven research and validation.

## Overview

Transform informal coding guidelines into structured, AI-friendly documentation with YAML front matter, code examples, and cross-references. The plugin helps maintain consistent coding standards across your project.

**Key Features:**
- 🤖 AI-driven convention generation from natural language
- 📚 Automatic research of codebase patterns and best practices
- 🔍 Codebase scanning to discover de-facto conventions automatically
- 🎯 Three complexity tiers (simple, standard, comprehensive)
- 🌍 Multi-language support (input in Russian/English, output in English)
- 📊 Automatic indexing and cataloging
- ✅ Three-level validation with auto-correction (structural, content, quality)
- ⭐ Automatic quality scoring with detailed reports
- 🔧 Auto-review hook for immediate validation after file creation

---

## Installation

### Prerequisites

- Claude Code CLI installed
- Git repository (for project-level plugin installation)

### Install Plugin

**Option 1: Project-level (Recommended)**

Copy this plugin directory to your project's `.claude/plugins/` directory:

```bash
mkdir -p .claude/plugins
cp -r claude-plugins/knowledge-adds .claude/plugins/knowledge-adds
```

**Option 2: Global**

Copy to Claude Code's global plugins directory:

```bash
cp -r claude-plugins/knowledge-adds ~/.claude/plugins/knowledge-adds
```

**Verify Installation:**

```bash
claude plugins list
# Should show: conventions (v1.0.0)
```

---

## Quick Start

### Discover Conventions from Existing Code

```bash
# Scan your codebase
/knowledge-scan src/

# Generate selected conventions from discovery plan
/knowledge-scan generate 1,3,5

# Or auto-generate high-confidence conventions
/knowledge-scan src/ --auto
```

### Create Conventions Manually

```bash
# Simple example
/knowledge-add TypeScript files should not exceed 800 lines

# Standard example with multiple rules
/knowledge-add TypeScript functions: camelCase, verb-based, descriptive, max 3 words

# Interactive mode for complex conventions
/knowledge-add interactive
```

### Update Catalog

After creating conventions, rebuild the index:

```bash
/knowledge-reindex
```

This generates `.kb/README.md` with organized listings of all conventions.

### Review and Validate

Check all entries for structural issues:

```bash
# Report only
/knowledge-review

# Auto-fix structural issues
/knowledge-review --fix
```

### Evaluate Quality

Get detailed quality analysis:

```bash
/knowledge-report functions.md
```

---

## Commands

### `/knowledge-add` - Create Convention

Create a standardized convention file from raw guidelines.

**Syntax:**
```bash
/knowledge-add [--path PATH] <guidelines|interactive>
```

**Modes:**
- **Direct input:** Provide guidelines as text
- **Interactive:** Step-by-step prompts for complex conventions
- **Custom path:** Override auto-generated path with `--path`

**Examples:**
```bash
# Direct
/knowledge-add TypeScript async functions must have try-catch blocks

# Interactive
/knowledge-add interactive

# Custom path
/knowledge-add --path .kb/react/hooks/naming.md React hooks: start with use, camelCase
```

**Auto-detects:**
- Complexity tier (simple/standard/comprehensive)
- Language (Russian/English input)
- Scope and category from guidelines
- Research needs ("best practices" trigger)

---

### `/knowledge-scan` - Discover Conventions

Automatically discover de-facto conventions from existing codebase.

**Syntax:**
```bash
/knowledge-scan [directory] [--auto]
```

**Modes:**
- **Discovery mode (default):** Analyze codebase and present interactive plan
- **Auto mode (`--auto`):** Automatically generate high-confidence conventions
- **Generation mode:** Select specific conventions from discovery plan

**Examples:**
```bash
# Discover conventions in src/
/knowledge-scan src/

# Auto-generate high-confidence conventions
/knowledge-scan src/ --auto

# Generate specific conventions from plan
/knowledge-scan generate 1,3,5
```

**What it does:**
- Detects stack from package.json, tsconfig.json, etc.
- Analyzes file patterns and naming conventions
- Searches for code patterns with Grep/Glob
- Calculates confidence scores for discovered patterns
- Detects conflicts with existing knowledge entries
- Generates conventions from discovered patterns

**Discovery dimensions:**
- File and directory naming
- Function/class naming conventions
- Import/export patterns
- Error handling patterns
- Documentation styles (JSDoc, docstrings)

---

### `/knowledge-reindex` - Update Catalog

Rebuild the conventions catalog.

**Syntax:**
```bash
/knowledge-reindex [--review]
```

**What it does:**
- Scans all `.kb/**/*.md` files
- Extracts metadata and titles
- Groups by scope and category
- Generates `.kb/README.md`
- Calculates statistics
- **With --review:** Adds Level 1 validation check

---

### `/knowledge-review` - Validate Entries

Review and validate knowledge base entries with optional auto-fixing.

**Syntax:**
```bash
/knowledge-review [path] [--fix | --fix-all]
```

**Arguments:**
- **path** (optional): Target for review
  - Omit to review all entries
  - Directory: `typescript/` or `.kb/typescript/`
  - Filename: `functions.md` (searches `.kb/`)
  - Full path: `.kb/typescript/naming/functions.md`

**Fix modes:**
- **None (default):** Report issues without modifying files
- **--fix:** Apply Level 1 auto-fixes (YAML, code tags, headings)
- **--fix-all:** Apply Level 1 + Level 2 AI-assisted fixes (with confirmation)

**Examples:**
```bash
# Review all entries (report only)
/knowledge-review

# Review specific directory
/knowledge-review typescript/

# Review with auto-fixes
/knowledge-review --fix

# Review directory with all fixes
/knowledge-review typescript/ --fix-all
```

**Validation levels:**
- **Level 1 - Structural:** Auto-fix YAML, code blocks, headings (safe)
- **Level 2 - Content:** AI-assisted section generation (requires confirmation)
- **Level 3 - Quality:** Recommendations only (no modifications)

**Features:**
- Three-level validation system
- Safe auto-corrections for structural issues
- AI-assisted content generation for missing sections
- Quality scoring with recommendations
- Batch processing with continue-on-failure
- Idempotent (safe to run multiple times)

---

### `/knowledge-report` - Quality Evaluation

Evaluate convention quality with detailed analysis.

**Syntax:**
```bash
/knowledge-report [file-path or convention-name]
```

**Examples:**
```bash
# By full path
/knowledge-report .kb/typescript/naming/functions.md

# By partial path
/knowledge-report typescript/naming/functions.md

# By name only
/knowledge-report functions.md

# Most recent convention
/knowledge-report
```

**Features:**
- Evaluates across 5 quality criteria
- Provides detailed feedback and recommendations
- Estimates improvement potential
- Generates comprehensive reports

---

## Convention Tiers

The plugin automatically detects the appropriate complexity tier based on input:

| Tier | Lines | Best For | Sections |
|------|-------|----------|----------|
| **Simple** | 20-50 | Single rule or concept | Title, Description, Format, 1 example |
| **Standard** | 50-150 | 3-6 related rules | + Allowed/Forbidden, Rules, examples |
| **Comprehensive** | 150-300 | 7+ complex rules | + Subsections, Summary table, cross-refs |

**Detection factors:**
- Number of rules mentioned
- Presence of subsections
- Detail level
- Number of examples provided

---

## Quality Evaluation

All conventions are automatically evaluated using 5 criteria:

### 1. Однозначность (Clarity) - 0-10 points
Can developers understand exactly how to apply the convention?
- Examples in Allowed/Forbidden sections
- Numbered rules
- Language specificity

### 2. Формат (Format) - 0-10 points
Visual structure and readability
- Valid YAML frontmatter
- Code blocks with language tags
- Heading hierarchy

### 3. Структура (Structure) - 0-10 points
Logical organization and information flow
- Required sections present
- Logical section order
- Domain subsections

### 4. Полнота (Completeness) - 0-10 points
Comprehensive coverage of necessary aspects
- Example count
- Edge cases
- Context description

### 5. Польза/Размер (Utility/Efficiency) - 0-10 points
High information density, optimal token usage
- Line count optimization
- Example-to-prose ratio
- Readability

**Grading scale:**
- 9.0-10.0: Outstanding ⭐⭐⭐⭐⭐
- 8.0-8.9: Excellent ⭐⭐⭐⭐
- 7.0-7.9: Good ⭐⭐⭐
- 6.0-6.9: Adequate ⭐⭐
- <6.0: Needs improvement ⚠️

---

## Scope and Categories

### Predefined Scopes

- `typescript` - TypeScript-specific
- `python` - Python-specific
- `react` - React framework
- `fastapi` - FastAPI framework
- `general` - Language-agnostic

### Custom Scopes

The plugin accepts any scope (e.g., `golang`, `vue`, `django`) and creates appropriate directory structure.

### Categories

- `naming` - Naming conventions
- `rules` - Structural constraints
- `docs` - Documentation standards
- `structure` - File/directory organization
- `patterns` - Code patterns and best practices

---

## Research Features

### Automatic Codebase Discovery (`/knowledge-scan`)

The `/knowledge-scan` command discovers conventions from existing code patterns:

**When to use:**
- Starting new project documentation
- Discovering existing team patterns
- Documenting legacy codebase
- Validating manual conventions

**How it works:**
1. Detects stack (TypeScript, React, Python, etc.)
2. Analyzes patterns across multiple dimensions
3. Calculates confidence scores (evidence-based)
4. Presents interactive plan with conflict detection
5. Generates conventions from actual code samples

**Advantages:**
- Evidence-based with confidence scores
- Batch discovery and generation
- Conflict detection with existing entries
- Enriched with real code examples

### Manual Entry Research (`/knowledge-add`)

The plugin automatically researches when creating manual entries:

**Research triggers:**
- User mentions "best practices" or "industry standard"
- Input is vague (<3 specific rules)
- Topic is well-established

**Research sources:**

1. **Codebase** (conditional):
   - Code patterns via grep
   - Linter configs (.eslintrc, tsconfig.json, etc.)
   - Existing conventions
   - Project dependencies

2. **Web** (if requested):
   - Style guides (Airbnb, Google, etc.)
   - Framework documentation
   - Best practice repositories

**Conflict resolution:**
- User's rules take priority
- Codebase patterns documented
- Best practices noted as context

---

## Architecture

### Plugin Structure

```
knowledge-base/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest
├── agents/
│   ├── knowledge-writer.md      # Entry generation agent
│   ├── knowledge-evaluator.md   # Quality evaluation agent
│   └── knowledge-scanner.md     # Codebase discovery agent
├── skills/
│   ├── knowledge-draft/         # Entry transformation skill
│   │   ├── SKILL.md
│   │   ├── template.md
│   │   └── examples.md
│   ├── knowledge-evaluate/      # Quality evaluation skill
│   │   └── SKILL.md
│   └── knowledge-codebase-discover/ # Codebase analysis skill
│       └── SKILL.md
├── commands/
│   ├── knowledge-add.md         # /knowledge-add command
│   ├── knowledge-scan.md        # /knowledge-scan command
│   ├── knowledge-reindex.md     # /knowledge-reindex command
│   └── knowledge-report.md      # /knowledge-report command
└── hooks/
    └── hooks.json               # PostToolUse notifications
```

### Pipeline Flow: Manual Entry Creation

```
User Input
    ↓
/knowledge-add command (interactive if needed)
    ↓
knowledge-writer agent (orchestrator)
    ↓
knowledge-draft skill:
    1. Parse input
    2. Detect language (Russian/English)
    3. Determine if research needed
    4. Conduct research (codebase + optional web)
    5. Detect complexity tier
    6. Extract metadata
    7. Structure content
    8. Generate YAML front matter
    9. Determine output path
    10. Handle file conflicts
    11. Quality evaluation (fast mode)
    12. Write and validate
    ↓
.kb/{scope}/{category}/{name}.md
    ↓
PostToolUse hook → Notification
    ↓
User runs /knowledge-reindex
    ↓
.kb/README.md generated
```

### Pipeline Flow: Automatic Discovery

```
User: /knowledge-scan src/
    ↓
knowledge-scan command (parse args, detect mode)
    ↓
knowledge-scanner agent (orchestrator)
    ↓
Phase 1: Discovery
    knowledge-codebase-discover skill:
    1. Detect stack (package.json, tsconfig, etc.)
    2. Build dimension registry for detected languages
    3. For each dimension:
       - Grep/Glob for patterns
       - Sample code from matching files
       - Calculate confidence score
    4. Return discovery plan
    ↓
knowledge-scanner agent:
    1. Parse plan
    2. Detect conflicts with existing .kb/ entries
    3. Store plan in session memory
    4. Present interactive plan to user
    ↓
User: /knowledge-scan generate 1,3,5
    ↓
Phase 2: Generation
    knowledge-scanner agent:
    1. Retrieve plan from session memory
    2. For each selected convention:
       - Check for conflicts
       - Build enriched guidelines from evidence
       - Delegate to knowledge-writer agent
    ↓
knowledge-writer agent → knowledge-draft skill
    ↓
.kb/{scope}/{category}/{name}.md (created or updated)
    ↓
PostToolUse hook → Notification
```

---

## File Structure

After using the plugin, your project will have:

```
.kb/
├── README.md                    # Auto-generated catalog
├── TEMPLATE.md                  # Convention template (optional)
├── typescript/
│   ├── naming/
│   │   ├── functions.md
│   │   └── variables.md
│   ├── rules/
│   │   └── file-size.md
│   └── patterns/
│       └── error-handling.md
├── python/
│   └── naming/
│       └── modules.md
└── general/
    └── rules/
        └── git-commits.md
```

---

## Configuration

### Agent Memory

The `convention-writer` agent maintains project memory:

```yaml
Convention Patterns:
  preferred_tier: standard
  example_depth: 2-3 code blocks

User Preferences:
  input_language: russian
  output_language: english
  detail_level: comprehensive

Project Context:
  frameworks: [react, fastapi]
  languages: [typescript, python]
  naming_style: camelCase
```

Memory persists across sessions to improve future convention generation.

---

## Validation

All generated conventions are validated for:

- ✅ Valid YAML front matter
- ✅ Required fields present
- ✅ Correct version format
- ✅ Required sections
- ✅ Code blocks have language tags
- ✅ Line count within tier limits
- ✅ No duplicate conventions
- ✅ Quality score meets threshold

**Warnings reported:**
- Line count near limits
- No code examples
- Custom scope used
- Cross-references to non-existent files
- Quality score < 7.0

---

## Documentation

- **[USAGE.md](USAGE.md)** - Comprehensive usage guide with examples
- **[TESTING.md](TESTING.md)** - Complete test suite and validation
- **README.md** (this file) - Overview and architecture

---

## Contributing

To improve this plugin:

1. **Report issues:** Use GitHub issues in the repository
2. **Suggest features:** Open a discussion
3. **Submit improvements:** Pull requests welcome

---

## Development

### Extending the Plugin

**Add new scope:**
- Scopes are auto-detected from input
- No code changes needed
- New directories created automatically

**Add new category:**
- Edit `SKILL.md` to add category to list
- Update documentation

**Customize templates:**
- Edit `skills/draft-convention/template.md`
- Modify tier specifications

---

## Support

**Questions or Issues?**

- Check [USAGE.md](USAGE.md) for detailed examples
- Review [TESTING.md](TESTING.md) for test cases
- Check the examples in `skills/draft-convention/examples.md`
- Review the template in `skills/draft-convention/template.md`
- Run `/knowledge-add` without arguments for usage help

**Links:**
- Repository: [ai-knowledge-base](https://github.com/whenessel/ai-knowledge-base)
- Report issues: [GitHub Issues](https://github.com/whenessel/ai-knowledge-base/issues)

---

## License

MIT License - see repository for details

---

**Author:** Artem Demidenko
**Version:** 1.1.0
**Plugin:** knowledge-base
