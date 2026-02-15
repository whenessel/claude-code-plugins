---
name: knowledge-draft
description: >
  Transform raw coding guidelines into standardized knowledge entry files.
  Handles parsing, optional codebase/web research, structure formalization,
  markdown generation, validation, and file writing. Produces .kb/
  files with YAML front matter, Format section, code examples, and cross-refs.
allowed-tools: Read, Grep, Glob, Write, Bash
---

> **⚠️ EXECUTION CONSTRAINT**: All code blocks below are pseudocode for reference only.
> - **NEVER** create or execute `.py` scripts
> - **NEVER** use Bash to run `python` or `python3` commands
> - Implement all logic using Claude's built-in tools: Read, Grep, Glob, Write, Edit, Bash, Task
> - Parse YAML/JSON content mentally from Read tool output — do NOT use Python yaml/json libraries
> - `invoke_skill()` / `invoke_agent()` in pseudocode = use **Task** tool with appropriate subagent_type

# Draft Knowledge Entry Skill

Transform raw coding guidelines into structured knowledge entry files.

## Input Expected

Raw guidelines from user:
- Natural language rules (e.g., "functions should be camelCase")
- Existing unstructured documentation
- Mixed Russian/English text
- Topic + scope hint (e.g., "TypeScript error handling best practices")

## Processing Pipeline

### Step 1: Parse Input

Extract from user's text:
- **Topic**: What knowledge entry covers
- **Rules mentioned**: List of guidelines
- **Scope hints**: Language/framework (typescript, python, react, etc.)
- **Examples provided**: Code snippets
- **Detail level**: Vague vs detailed (affects tier)

**Language detection:**
Check for Cyrillic characters to detect Russian input. If detected:
- `input_language = "russian"`
- Process correctly but output in English

### Step 2: Determine Research Need

**Auto-trigger codebase analysis if user input contains:**

**Русские триггеры:**
- "изучи стиль кода"
- "проанализируй код"
- "на основе кода"
- "из существующего кода"
- "сформулируй конвенцию из кода"
- "посмотри как написано"

**English triggers:**
- "analyze code"
- "based on existing code"
- "study code style"
- "from codebase"
- "how code is written"

**Auto-trigger web research if user input contains:**

**English triggers:**
- "best practices"
- "industry standard"
- "recommended approach"
- "what's the best way"
- "state of the art"

**Russian triggers:**
- "лучшие практики"
- "стандарты индустрии"
- "рекомендуемый подход"
- "как лучше"
- "принятые практики"
- "общепринятые стандарты"

**Auto-trigger web research if input quality is low:**
```text
DECISION LOGIC (reference — do NOT execute as code):

should_research_web(guidelines, rule_count):
  ru_triggers = ["лучшие практики", "стандарты индустрии", "рекомендуемый подход", "как лучше"]
  en_triggers = ["best practices", "industry standard", "recommended approach", "state of the art"]

  has_trigger = any trigger found in guidelines (case-insensitive)

  low_quality = rule_count < 3 AND "example" not in guidelines (case-insensitive)

  skip = any of ["skip research", "no research", "без исследования"] found in guidelines

  return (has_trigger OR low_quality) AND NOT skip
```

### Step 3: Conduct Research

**A. Codebase Analysis (ONLY if triggered):**

If codebase analysis triggered by keywords:

1. **Generate search patterns** based on topic:
   **Pattern generation lookup table:**

   | Topic Contains | Search Patterns |
   |---------------|----------------|
   | "error handling" | `try`, `catch`, `throw`, `Error(` |
   | "function naming" | `function `, `const.*=>`, `async function` |
   | "imports" | `import `, `from '`, `require(` |
   | *(other)* | Split topic into individual words |

2. **Search codebase** using Grep:
   For each search pattern from the table above:
   1. Use **Grep** tool with:
      - pattern: the search pattern
      - type: file types for the detected scope (e.g., `ts` for TypeScript, `py` for Python)
      - output_mode: `"files_with_matches"`
      - head_limit: 20
   2. Add all matching file paths to a combined results list

3. **Read examples** (up to 20 files):
   For each unique file path (up to 20):
   1. Use **Read** tool to get the file content
   2. Extract code sections relevant to the topic
   3. Store as: `{path: file_path, snippet: relevant_code}`

4. **Analyze patterns**:
   Analyze the collected code examples to identify statistical patterns (naming conventions, structure, frequencies) relevant to the topic.

5. **Check linter configs**:
   Use **Glob** tool with pattern `**/{.eslintrc*,tsconfig.json,.prettierrc*}` to find linter configuration files. Then use **Read** to examine them and extract rules relevant to the current topic.

6. **Format as guidelines**:
   Format the discovered patterns as guidelines text:

   ```text
   Discovered patterns from codebase:
   - {dominant_pattern description}
   - Found in {example_count} files

   Examples:
   {top 3 code examples}

   Linter rules:
   {relevant linter rules}
   ```

**B. Web Research (if triggered):**

Use WebSearch for best practices:

Use **WebSearch** tool:
- Query: `"{topic} best practices 2026"`
- If input language is Russian, translate the topic to English first
- Limit: 3 results

**Merge findings:**
```
Priority:
1. User's explicit rules (highest)
2. Codebase patterns (if analyzed)
3. Web best practices (supplement)
```

### Step 4: Fixed Comprehensive Tier

**ALWAYS generate Comprehensive tier:**

```yaml
# Fixed tier (always comprehensive)
tier: comprehensive
target_lines: 150-300
required_sections:
  - Title
  - "Description (2-3 paragraphs)"
  - "Format (4+ bullet points)"
  - "Domain sections with subsections"
  - "Allowed (4+ examples)"
  - "Forbidden (4+ anti-patterns)"
  - "Rules (7+ guidelines)"
  - Summary table
```

**Content generation strategy:**

If user input is minimal:
1. Use web research to fill gaps
2. Generate realistic examples based on best practices
3. Create comprehensive rules covering edge cases
4. Add domain-specific subsections
5. Build summary table

If user input is detailed:
1. Expand provided rules with rationale
2. Add more examples showing edge cases
3. Organize into subsections
4. Create summary table

### Step 5: Extract Metadata with Smart Detection

**Version:**
- Start with `1.0` for new conventions
- Increment minor (1.1, 1.2) for additions/clarifications
- Increment major (2.0, 3.0) for breaking changes or rewrites

**Scope detection with confidence:**

```text
SCORING LOGIC (reference only — apply mentally, do NOT execute as code):

detect_scope(guidelines, explicit_scope):
  If explicit_scope provided: return (explicit_scope, confidence=1.0)

  Scope indicators:
    typescript: ["typescript", "ts", ".ts", "type", "interface"]
    python:     ["python", "py", "def ", "class ", "import "]
    react:      ["react", "component", "jsx", "tsx", "hook"]
    fastapi:    ["fastapi", "pydantic", "endpoint"]
    general:    ["any language", "language-agnostic"]

  For each scope: score = (matches found in guidelines) / (total indicators for that scope)
  Best scope = highest score
  Confidence = that score value

  If confidence < 0.7 AND no explicit_scope:
    → Signal "scope_ambiguous" with detected scope, confidence, and list of all scope options
    → Agent will use AskUserQuestion to clarify with user
  Else:
    → Proceed with the detected scope
```

**Category detection with confidence:**

```text
SCORING LOGIC (reference only — apply mentally, do NOT execute as code):

detect_category(guidelines, topic, explicit_category):
  If explicit_category provided: return (explicit_category, confidence=1.0)

  Category indicators:
    naming:    ["name", "naming", "camelCase", "snake_case"]
    rules:     ["limit", "constraint", "must not exceed", "maximum"]
    docs:      ["document", "comment", "jsdoc", "docstring"]
    structure: ["organize", "folder", "directory", "file structure"]
    patterns:  ["pattern", "practice", "handle", "error", "state"]

  combined = (guidelines + " " + topic), lowercased
  For each category: score = (matches in combined) / (total indicators)

  Ambiguity check:
    Sort scores descending. If gap between #1 and #2 < 0.2:
      → return (best category, confidence=0.5)

  If confidence < 0.7 AND no explicit_category:
    → Signal "category_ambiguous" with detected category, confidence, and options list
    → Agent will use AskUserQuestion to clarify
  Else:
    → Proceed with detected category
```

**Type detection with confidence:**

```text
SCORING LOGIC (reference only — apply mentally, do NOT execute as code):

detect_type(guidelines, topic, explicit_type):
  If explicit_type provided: return (explicit_type, confidence=1.0)

  combined = (guidelines + " " + topic), lowercased

  Type indicators (strong and weak for each type):

    rule:
      strong: ["must not", "must never", "forbidden", "prohibited",
               "limit", "maximum", "minimum", "constraint", "enforce",
               "required", "mandatory", "not allowed"]
      weak:   ["should not", "avoid", "restrict", "cap at", "ceiling"]

    pattern:
      strong: ["pattern", "architecture", "design pattern", "approach",
               "error handling", "state management", "data flow"]
      weak:   ["practice", "strategy", "technique", "method"]

    guide:
      strong: ["how to", "step by step", "tutorial", "walkthrough",
               "getting started", "workflow", "instructions"]
      weak:   ["migrate", "upgrade", "follow these steps"]

    documentation:
      strong: ["documentation", "document this", "knowledge article",
               "describe how", "explanation of", "overview of",
               "save this documentation", "docs block"]
      weak:   ["docs", "describe", "explain", "context", "background"]

    reference:
      strong: ["reference", "specification", "api docs", "cheat sheet",
               "options", "parameters", "configuration reference"]
      weak:   ["schema", "interface", "types", "lookup"]

    style:
      strong: ["formatting", "indentation", "spacing", "prettier",
               "code style", "lint rule", "visual style"]
      weak:   ["format", "indent", "whitespace", "semicolons"]

    environment:
      strong: ["setup", "scaffold", "boilerplate", "environment",
               "infrastructure", "ci/cd", "pipeline", "deploy",
               "project initialization", "dev environment"]
      weak:   ["configure", "install", "docker", "terraform",
               "github actions", ".env", "toolchain"]

    convention:
      strong: ["convention", "naming", "camelCase", "PascalCase",
               "snake_case", "agreed", "team standard"]
      weak:   ["prefer", "consistently", "standardize", "uniform"]

  Formula: score = (strong_count * 2 + weak_count) / (total_strong + total_weak)

  best_type = type with highest score
  confidence = that score

  If confidence < 0.1: return ("convention", 0.5)  — default fallback
  Otherwise: return (best_type, min(1.0, confidence + 0.3))

  If final confidence < 0.7 AND no explicit_type:
    → Signal "type_ambiguous" with detected type, confidence, and options:
      ["convention", "rule", "pattern", "guide", "documentation", "reference", "style", "environment"]
    → Agent will use AskUserQuestion to clarify
  Else:
    → Proceed with detected type
```

**Tags:**
Extract 3-5 relevant keywords from:
- Input text (camelCase, async, hooks)
- Domain terms (functions, classes, components)
- Technologies (react, typescript, fastapi)

### Step 6: Structure Knowledge Entry Content

Build sections based on detected tier:

**ALL tiers get:**
```markdown
# {title}

{description with context}

## Format

- **{concept}**: {explanation}
- **{concept}**: {explanation}
```

**Standard+ tiers get:**
```markdown
## Allowed

```{language}
// Good example with realistic code
{code snippet}
```

## Forbidden

```{language}
// Bad example showing anti-pattern
{code snippet}
```

## Rules

1. **{principle}**: {explanation}
2. **{principle}**: {explanation}
```

**Comprehensive tier gets:**
```markdown
## {Domain Section 1}

### Subsection A
Detailed guidelines...

### Subsection B
Detailed guidelines...

## {Domain Section 2}
...

## Summary

| Aspect | Requirement | Example |
|--------|-------------|---------|
| ... | ... | ... |
```

### Step 7: Generate YAML Front Matter

```yaml
---
type: {type}
version: {version}
scope: {scope}
category: {category}
tags: [{tag1}, {tag2}, {tag3}]
---
```

Validate format before writing:
- type must be one of: convention, rule, pattern, guide, documentation, reference, style, environment
- version must match pattern: `\d+\.\d+`
- scope must be non-empty string
- category must be one of predefined categories
- tags must be array of strings

### Step 8: Determine Output Path & Check for Updates

Generate path from metadata:

Construct the output path from metadata:
- Pattern: `.kb/{scope}/{category}/{kebab-case-title}.md`
- Convert the title to kebab-case for the filename (e.g., "Function Naming" → `function-naming`)

**Check for existing knowledge entry:**

**Check if file already exists:** Use **Glob** or **Read** (handling errors for missing files) to check if `output_path` already exists.

**If file exists:**
1. Use **Read** to get existing content
2. Extract current version from YAML frontmatter
3. Analyze user intent using the intent analysis logic below
4. Determine update strategy and version increment:

```text
DECISION LOGIC (reference only — apply mentally, do NOT execute as code):

  intent → strategy mapping:
    "append_rules"    → strategy="append",       increment="minor"
    "update_values"   → strategy="update_fields", increment="minor"
    "rewrite_section" → strategy="rewrite",       increment="minor" (or "major" if Rules/Format section)
    "full_overwrite"  → strategy="overwrite",     increment="major"
    "ambiguous"       → signal "update_confirmation_needed" to agent for user clarification

  If increment is "major": requires_confirmation = true (agent must confirm with user)
```

**If file does NOT exist:** Create new entry with version "1.0".

**Intent analysis patterns:**

```text
INTENT ANALYSIS (reference only — apply mentally, do NOT execute as code):

Check guidelines (case-insensitive) for these phrases in ORDER (most specific first):

  1. full_overwrite — if contains: "completely rewrite", "полностью переписать"
  2. rewrite_section — if contains: "improve examples", "enhance examples", "улучши примеры"
  3. update_values — if contains: "change", "update", "increase", "decrease", "изменить", "увеличить"
  4. append_rules — if contains: "add rule", "добавь правило", "include also", "ещё один"
  5. If none matched → "ambiguous"
```

### Step 9: Execute File Operation

**For NEW conventions:**

1. Use **Bash** to create the directory structure: `mkdir -p` followed by the directory portion of the output path
2. Use **Write** tool to save the complete file (YAML frontmatter + blank line + content sections) to the output path

**For UPDATES:**

**Execute update based on strategy:**

- **append**: Use **Read** to get existing file. Find the Rules section (## Rules). Add the new rules at the end of that section. Update version in YAML frontmatter. Use **Write** to save.
- **update_fields**: Use **Read** to get existing file. Replace specific field values in the Format section. Update version in YAML frontmatter. Use **Write** to save.
- **rewrite**: Use **Read** to get existing file. Replace the target section entirely with new content. Update version in YAML frontmatter. Use **Write** to save.
- **overwrite**: Use **Write** to save the complete new content, replacing the file entirely.

**Version increment logic:**

```text
VERSION INCREMENT (reference only — apply mentally, do NOT execute as code):

  Parse "major.minor" from current version string
  If increment_type is "major": return "{major+1}.0"
  If increment_type is "minor": return "{major}.{minor+1}"
```

**Helper functions:**

**How to parse markdown into sections:**
1. Extract YAML frontmatter: find content between the first `---` and second `---`
2. Parse the YAML fields mentally (type, version, scope, category, tags)
3. Split remaining content by `## ` headings
4. Each heading becomes a section name; lines until the next heading become the section content
5. Store as a mental map of `{section_name: content_lines}`

**How to rebuild markdown from sections:**
1. Format YAML frontmatter between `---` markers
2. For each section in order: write `## {section_name}` followed by its content lines
3. Separate sections with blank lines

### Step 10: Validate

Run post-write validation checks:

- [ ] YAML front matter parses correctly
- [ ] Required fields present: type, version, scope, category, tags
- [ ] Type field is one of valid values: convention, rule, pattern, guide, documentation, reference, style, environment
- [ ] Version format valid (e.g., 1.0, 1.1, 2.0)
- [ ] Required sections present (Title, Description, Format)
- [ ] Code blocks have language tags
- [ ] Cross-references point to existing files (if any)
- [ ] Line count within tier limits (±20% tolerance)
- [ ] No duplicate conventions (same scope+category+title)

**Return validation result:**
```json
{
  "status": "success",
  "path": ".kb/typescript/naming/functions.md",
  "tier": "standard",
  "line_count": 95,
  "sections": 5,
  "examples": 3,
  "validation": {
    "yaml_valid": true,
    "sections_complete": true,
    "within_tier_limits": true,
    "warnings": []
  }
}
```

**Warnings to report** (non-blocking):
- Line count near tier limits
- No code examples provided
- Custom scope used (not predefined)

### Step 11: Quality Evaluation

Evaluate the created knowledge entry using the knowledge-evaluate skill with text-based invocation:

Invoke the **knowledge-evaluate** skill via **Task** tool:
- Description: "Evaluate knowledge entry quality"
- Prompt: `"Evaluate knowledge entry quality for file: {output_path}\nMode: fast"`

Parse the returned text to extract scores. Expected format:
`Quality: X.X/10 (Clarity: N, Format: N, Structure: N, Completeness: N, Efficiency: N)`

```text
RESULT HANDLING (reference only — apply mentally, do NOT execute as code):

  If evaluation text matches the expected format:
    Extract overall_score and individual scores (clarity, format, structure, completeness, efficiency)

    If overall_score < 7.0:
      Add warning: "⚠ Quality below target. Run /knowledge-report for detailed analysis."

  If evaluation fails or format doesn't match:
    Continue without quality metrics — do NOT fail the knowledge entry creation
    Set quality to "Evaluation skipped"

  Quality evaluation is NON-BLOCKING — knowledge entry creation always succeeds
```

**Important Notes**:
- Quality evaluation is **non-blocking** - wrapped in try-catch
- Uses **text-based prompts** (no JSON serialization overhead)
- Parses text output with regex (no JSON deserialization overhead)
- If evaluation fails, knowledge entry creation still succeeds
- Fast mode is used for quick structural analysis (<2 seconds)
- Quality score is informational only, doesn't affect creation
- User can run `/knowledge-report` for deep evaluation later

## Edge Cases

### 1. Insufficient Input

If user input is too vague (<2 rules and no examples):

```json
{
  "status": "insufficient_input",
  "questions": [
    "What specific rules should this knowledge entry enforce?",
    "Can you provide an example of correct usage?",
    "What problem does this knowledge entry solve?"
  ],
  "suggestion": "Would you like me to research best practices for this topic?"
}
```

### 2. Language Mix

- **Russian input** → Parse correctly, detect Cyrillic
- **Output** → ALWAYS in English
- **Store preference** in agent memory for future sessions

### 3. File Exists (Conflict)

```json
{
  "status": "conflict",
  "path": ".kb/typescript/naming/functions.md",
  "options": ["overwrite", "rename", "merge"]
}
```

Wait for user decision before proceeding.

### 4. Custom Scope

Accept and create directory structure for non-predefined scopes:
```
User mentions "golang" or "vue" (not in predefined list)
→ Accept as valid custom scope
→ Create .kb/golang/ directory structure
→ Note in validation result: "Custom scope 'golang' created"
```

### 5. Research Conflicts

When best practices differ from codebase patterns:
```
Best practice: camelCase for TypeScript
Codebase: 145/167 files use snake_case

→ Document project pattern as primary rule
→ Add note about best practice as context
→ Example: "This project uses snake_case for consistency with
           Python backend, though TypeScript best practices
           typically recommend camelCase"
```

## Supporting Files

Reference these only when needed (progressive disclosure):

- `template.md` - Complete knowledge entry template specification
- `examples.md` - Reference examples for all three tiers

Use Read tool to access these files when:
- User asks about template structure
- Need examples for specific tier
- Validation requires template comparison

## Implementation Notes

**Tool usage:**
- Use **Grep** for searching code patterns
- Use **Glob** for finding config files
- Use **Read** for reading existing conventions/configs
- Use **Write** for creating knowledge entrys
- Use **Bash** only for directory creation (mkdir -p)
- **NEVER** create Python scripts (.py files) — all logic must use built-in tools above

**Error handling:**
- Catch file write errors and report clearly
- Handle missing directories gracefully
- Validate all paths before writing
- Never overwrite without confirmation

**Performance:**
- Limit research results (20 grep results, 3 web sources max)
- Read only first 100 lines of config files
- Skip research for well-defined input
- Cache research results in agent memory if available
