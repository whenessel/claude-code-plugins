# Changelog

All notable changes to the Knowledge Base Plugin will be documented in this file.

## 2026-02-15: Multi-Type Support - Version 1.3.0

### Added
- **Multi-type knowledge entries** - Expanded from single `convention` type to 8 semantic types
- **Type auto-detection** - Signal-based scoring algorithm detects entry type from content
- **`type:X` argument** - Explicit type override in `/knowledge-add` command
- **Type validation** - Review system validates type field against valid values

### Entry Types
- `convention` - Naming/style agreements, team norms (default)
- `rule` - Hard constraints, limits, prohibitions
- `pattern` - Code patterns, architectural approaches
- `guide` - How-to documentation, workflows
- `documentation` - Saved knowledge blocks, explanations
- `reference` - API specs, configuration descriptions
- `style` - Code formatting, visual preferences
- `environment` - Setup, configuration, scaffolding, infrastructure

### Features
- **Confidence-based detection**: Auto-detects type with confidence scoring (threshold: 70%)
- **Ambiguity handling**: Prompts user when type detection confidence < 70%
- **Backward compatible**: Existing `type: convention` entries remain valid
- **Scanner integration**: `/knowledge-scan` suggests type based on dimension
- **Review integration**: `/knowledge-review` validates type field and suggests fixes

### Type Detection Logic
- **Strong signals** (2x weight): Keywords like "must not", "how to", "pattern", "setup"
- **Weak signals** (1x weight): Context keywords like "avoid", "describe", "configure"
- **Dimension mapping**: Scanner maps dimensions to types (e.g., `functions` → `convention`, `error-handling` → `pattern`)
- **Fallback**: Defaults to `convention` when no clear signals detected

### Examples
```bash
# Auto-detect type from content
/knowledge-add TypeScript max 800 lines per file  # → type: rule
/knowledge-add Error handling best practices      # → type: pattern
/knowledge-add Node.js scaffolding: npm, docker   # → type: environment

# Explicit type override
/knowledge-add type:guide How to set up API endpoints
/knowledge-add type:documentation React component lifecycle
```

### Enhanced
- **`knowledge-draft` skill**: Added type detection in Step 5 (Extract Metadata)
- **`knowledge-writer` agent**: Handles `type_ambiguous` status with AskUserQuestion
- **`knowledge-review` skill**: Detects missing/invalid type, suggests fixes
- **`codebase-discover` skill**: Adds `suggested_type` field to discovery plan
- **Template documentation**: Updated to show all 8 valid types

## 2026-02-15: Knowledge Review System - Version 1.2.0

### Added
- **`/knowledge-review` command** - Validate knowledge base entries with optional auto-fixing
- **`knowledge-reviewer` agent** - Orchestrates three-level validation workflow
- **`knowledge-review` skill** - Comprehensive validation checklists and fix strategies
- **Auto-review hook** - Automatically validates new entries after Write operations

### Features
- **Three-level validation system**:
  - **Level 1 (Structural)**: Safe auto-fixes for YAML, code blocks, headings
  - **Level 2 (Content)**: AI-assisted section generation with user confirmation
  - **Level 3 (Quality)**: Recommendations via existing `knowledge-evaluate` skill
- **Fix modes**:
  - `none` (default): Report issues without modifying files
  - `--fix`: Apply Level 1 auto-fixes only
  - `--fix-all`: Apply Level 1 + Level 2 with confirmation
- **Batch processing**: Continue-on-failure strategy for reviewing multiple files
- **Quality scoring**: Integration with `knowledge-evaluate` in fast mode
- **Project memory**: Track review history and quality trends
- **Idempotent**: Safe to run multiple times

### Enhanced
- **`/knowledge-reindex`**: Added `--review` flag for optional validation check during reindexing
- **PostToolUse hook**: Changed from simple notification to auto-review with Level 1 fixes
  - Matcher: `Write` only (prevents recursion from Edit operations)
  - Type: `prompt` (triggers agent workflow instead of command)
  - Behavior: Silent Level 1 fixes with brief summary

### Validation Checks
- **YAML frontmatter**: Missing fields, syntax errors, version format
- **Markdown structure**: Heading hierarchy, code block tags, required sections
- **Path consistency**: File/directory naming alignment with metadata
- **Tier compliance**: Section requirements, example counts, line limits
- **Cross-references**: Broken links, external link accessibility
- **Quality assessment**: Clarity, completeness, efficiency

### Workflow Examples
```bash
# Review all entries (report only)
/knowledge-review

# Review specific directory with auto-fixes
/knowledge-review typescript/ --fix

# Review with AI-assisted fixes (requires confirmation)
/knowledge-review --fix-all

# Reindex with validation check
/knowledge-reindex --review
```

### Performance
- Single file review: ~2-3 seconds (including quality evaluation)
- Batch review (20 files): ~40-60 seconds
- Level 1 validation only: <500ms per file
- Auto-review hook: ~500ms overhead per Write operation

### Version
- Plugin version: `1.1.0` → `1.2.0`
- Keywords: added `"review"`, `"lint"`, `"validate"`

---

## 2026-02-15: Codebase Discovery - `/knowledge-scan`

### Added
- **`/knowledge-scan` command** - Automatically discover de-facto conventions from existing codebase
- **`knowledge-scanner` agent** - Orchestrates two-phase discovery and generation workflow
- **`knowledge-codebase-discover` skill** - Analyzes codebase patterns with Grep/Glob

### Features
- **Two-phase workflow**: Discovery → Interactive plan → Selective generation
- **Auto mode (`--auto`)**: Automatically generate high-confidence (≥80%) conventions
- **Confidence scoring**: Evidence-based scores guide decision-making
- **Conflict detection**: Compares discovered patterns with existing entries
- **Multi-language support**: TypeScript, JavaScript, React, Python detection
- **Dimension registry**: Extensible pattern analysis (naming, structure, patterns, docs)
- **Session memory**: Discovery plan persists for selective generation
- **Batch processing**: Generate multiple conventions in one command

### Discovery Dimensions
- General: file/directory naming
- TypeScript: functions, classes, imports, exports, error handling, JSDoc
- React: components, hooks, state management
- Python: functions, classes, modules, docstrings, error handling

### Workflow
```bash
# Phase 1: Discovery
/knowledge-scan src/

# Phase 2: Generation
/knowledge-scan generate 1,3,5

# Or: Auto mode
/knowledge-scan src/ --auto
```

### Technical Details
- **Performance**: ~180 tool calls for full scan of 500-file project (<60s)
- **Optimizations**: Grouped Glob, shared Read calls, `grep -l` for file lists
- **Reuse**: Delegates to existing `knowledge-writer` → `knowledge-draft` for generation

### Version
- Plugin version: `1.0.0` → `1.1.0`
- Keywords: added `"codebase-scan"`

---

## 2026-02-14: Major Refactoring - Conventions → Knowledge Base

### Breaking Changes

- **Plugin renamed**: `conventions` → `knowledge-base`
- **Storage directory**: `.conventions/` → `.kb/`
- **Commands renamed**:
  - `/convention` → `/knowledge-add`
  - `/convention-rebuild` → `/knowledge-reindex`
  - `/convention-report` → `/knowledge-report`
- **Agents renamed**: `convention-writer` → `knowledge-writer`, `convention-evaluator` → `knowledge-evaluator`
- **Skills renamed**: `convention-draft` → `knowledge-draft`, `convention-evaluate` → `knowledge-evaluate`

### Concept Evolution

The plugin now focuses on building a comprehensive knowledge base of development guidelines, including conventions, rules, styles, and practices.

### Migration

**No automatic migration provided.** Users with existing `.conventions/` directories should:
1. Manually rename `.conventions/` → `.kb/` in their projects
2. Update any custom scripts or documentation referencing the old paths
3. Re-run `/knowledge-reindex` to rebuild the catalog

### Backwards Compatibility

- YAML frontmatter `type: convention` preserved for compatibility
- No changes to knowledge entry structure or metadata fields

## 2026-02-13: Quality Evaluation - Multi-Line Format

### Added
- Multi-line text format for evaluation results (replaces JSON)
- Detailed breakdown for each criterion with strengths/issues/recommendations
- Overall quality score with grade (Outstanding ⭐⭐⭐⭐⭐ to Needs Improvement ⭐)

### Changed
- `/knowledge-report` now returns formatted text instead of JSON
- Evaluation speed improved (<30s for deep mode)

### Fixed
- Evaluation serialization overhead eliminated
- Better error messages for malformed entries

## 2026-02-12: Russian Language Support

### Added
- Full Russian input support for all commands
- Language auto-detection
- Bilingual help and documentation

### Changed
- Guidelines can now be provided in Russian or English
- Output files remain in English for consistency

## Initial Release - 2026-02-10

### Features
- `/knowledge-add` - Generate knowledge entries from raw guidelines
- `/knowledge-reindex` - Rebuild knowledge base catalog
- `/knowledge-report` - Quality evaluation and recommendations
- Three-tier complexity system (simple, standard, comprehensive)
- Automatic codebase pattern detection
- Web research capabilities
- Quality scoring (1-10 scale across 5 criteria)
- Batch processing support for directories
- Update detection and version management
