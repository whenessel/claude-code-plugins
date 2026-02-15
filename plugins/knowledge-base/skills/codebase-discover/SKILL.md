---
name: codebase-discover
description: >
  Analyze project codebase to discover de-facto coding conventions.
  Detects project stack, scans files across multiple dimensions,
  calculates consistency scores, and produces a ranked discovery plan.
  Does NOT generate knowledge entries — only produces the analysis.
allowed-tools: Read, Grep, Glob, Bash
---

> **⚠️ EXECUTION CONSTRAINT**: All code blocks below are pseudocode for reference only.
> - **NEVER** create or execute `.py` scripts
> - **NEVER** use Bash to run `python` or `python3` commands
> - Implement all logic using Claude's built-in tools: Read, Grep, Glob, Write, Edit, Bash, Task
> - Parse YAML/JSON content mentally from Read tool output — do NOT use Python yaml/json libraries
> - `invoke_skill()` / `invoke_agent()` in pseudocode = use **Task** tool with appropriate subagent_type

# Codebase Discovery Skill

Analyze project codebase to find patterns, conventions, and standards that are consistently followed.

## Input Expected

Text prompt from knowledge-scanner agent:

```
Scan codebase to discover conventions.

Scan path: . (or specific directory)
Scope filter: auto-detect (or specific scope)
Minimum confidence: 60
```

## Processing Pipeline

### Step 1: Detect Project Stack

Identify languages, frameworks, and tools by examining project configuration files.

**1A. Find configuration files:**
```yaml
config_patterns:
  # JavaScript/TypeScript ecosystem
  - "package.json"
  - "tsconfig.json"
  - "tsconfig*.json"
  - ".eslintrc*"
  - ".prettierrc*"
  - "prettier.config.*"
  - "biome.json"
  - "biome.jsonc"
  - ".stylelintrc*"
  - "vite.config.*"
  - "next.config.*"
  - "webpack.config.*"
  - "tailwind.config.*"

  # Python ecosystem
  - "pyproject.toml"
  - "setup.py"
  - "setup.cfg"
  - "requirements*.txt"
  - "Pipfile"
  - ".flake8"
  - "mypy.ini"
  - ".mypy.ini"
  - "ruff.toml"
  - ".ruff.toml"
  - ".isort.cfg"

  # Go
  - "go.mod"
  - "go.sum"

  # Rust
  - "Cargo.toml"

  # Java/Kotlin
  - "build.gradle*"
  - "pom.xml"

  # General
  - ".editorconfig"
  - "Dockerfile*"
  - "docker-compose*.yml"
  - ".gitignore"
```

For each pattern in the list above, use **Glob** with `pattern=<pattern>` and `path=<scan_path>`. Collect all patterns that return results into a `found_configs` mapping of pattern to matched file paths.

**1B. Count files by extension:**
| Extension | Language | Notes |
|-----------|----------|-------|
| ts | typescript | |
| tsx | typescript | also signals React |
| js | javascript | |
| jsx | javascript | also signals React |
| py | python | |
| go | golang | |
| rs | rust | |
| java | java | |
| kt | kotlin | |
| rb | ruby | |
| php | php | |
| vue | vue | |
| svelte | svelte | |

For each extension in the table above, use **Glob** with `pattern=**/*.<ext>` and `path=<scan_path>`. From the results, exclude files whose paths contain any of: `node_modules`, `dist`, `build`, `.next`, `__pycache__`, `.venv`, `venv`, `.git`, `vendor`, `target`, `.tox`. Record the count of remaining files per extension into `file_counts`.

**1C. Detect frameworks from dependencies:**
**If `package.json` was found:** Use **Read** to get its content, then parse the JSON mentally. Combine `dependencies` and `devDependencies` into a single set of dependency names. Check for the following markers:

```yaml
framework_markers:
  react: "react"
  next: "nextjs"
  "@angular/core": "angular"
  vue: "vue"
  svelte: "svelte"
  express: "express"
  fastify: "fastify"
  "@nestjs/core": "nestjs"
  tailwindcss: "tailwind"
  "@chakra-ui/react": "chakra-ui"
  zustand: "zustand"
  "@reduxjs/toolkit": "redux"
  prisma: "prisma"
  drizzle-orm: "drizzle"
  vitest: "vitest"
  jest: "jest"
  mocha: "mocha"
  storybook: "storybook"
```

For each marker key present in the combined dependencies, add the corresponding value to the frameworks list.

**If `pyproject.toml` was found:** Use **Read** to get its content. Check (case-insensitive) for the following markers:

```yaml
python_markers:
  fastapi: "fastapi"
  django: "django"
  flask: "flask"
  sqlalchemy: "sqlalchemy"
  pydantic: "pydantic"
  pytest: "pytest"
  alembic: "alembic"
```

For each marker found in the file content, add the corresponding value to the frameworks list.

**1D. Read linter configurations:**
Extract linter rules from discovered config files into a `rules` mapping:

1. **ESLint**: Find config files matching `.eslintrc` among discovered configs. If found, use **Read** to get the first file's content, parse JSON/YAML mentally, and extract the `rules` section. Store under `rules["eslint"]`.

2. **TypeScript config**: If `tsconfig.json` was found, use **Read** to get its content, parse JSON mentally, and extract the `compilerOptions` section. Store under `rules["typescript"]`.

3. **Prettier**: Find config files matching `.prettierrc` or `prettier.config` among discovered configs. If found, use **Read** to get the first file's content, parse JSON mentally, and store the entire config under `rules["prettier"]`.

4. **Python linters**: For each of `ruff.toml`, `.ruff.toml`, `.flake8`, `mypy.ini` — if found in configs, use **Read** to get the file content and store the raw content under the tool name (with dots removed) in `rules`.

If any read or parse fails, skip that tool silently and continue.

**1E. Build stack summary:**
```yaml
stack:
  languages: <determined from file_counts — extensions with highest counts>
  frameworks: <detected from found_configs via framework markers above>
  tools: <list of all config pattern keys that had matches>
  file_counts: <map of extension to count from Step 1B>
  linter_rules: <extracted rules from Step 1D>
```

### Step 2: Build Dimension Registry

Define analysis dimensions based on detected stack. Each dimension specifies what to search for and how to measure consistency.

Build a list of dimensions to analyze based on the detected project stack.

#### Universal dimensions (always included for all projects)

```yaml
- id: file-naming
  scope: general
  category: naming
  dimension: files
  title: "File Naming Conventions"
  applicable_extensions: <all keys from file_counts>
  analysis_type: filename_pattern
  patterns:
    kebab-case: '^[a-z][a-z0-9]*(-[a-z0-9]+)*\.\w+$'
    camelCase: '^[a-z][a-zA-Z0-9]*\.\w+$'
    PascalCase: '^[A-Z][a-zA-Z0-9]*\.\w+$'
    snake_case: '^[a-z][a-z0-9]*(_[a-z0-9]+)*\.\w+$'

- id: directory-structure
  scope: general
  category: structure
  dimension: directories
  title: "Directory Structure Conventions"
  analysis_type: directory_pattern
  patterns:
    feature-based:
      - "features/"
      - "modules/"
      - "domains/"
    layer-based:
      - "controllers/"
      - "services/"
      - "models/"
      - "repositories/"
    type-based:
      - "components/"
      - "hooks/"
      - "utils/"
      - "types/"
      - "helpers/"
```

#### **Include the following if detected stack includes TypeScript or JavaScript:**

Applicable extensions for all dimensions in this group: `ts`, `tsx`, `js`, `jsx`

```yaml
- id: ts-function-naming
  scope: typescript
  category: naming
  dimension: functions
  title: "Function Naming Conventions"
  analysis_type: grep_pattern
  grep_patterns:
    - label: "camelCase functions"
      pattern: '(function|const|let)\s+[a-z][a-zA-Z0-9]*\s*[=(]'
    - label: "verb prefix (get/set/is/has/create/update/delete/fetch/handle)"
      pattern: '(function|const)\s+(get|set|is|has|create|update|delete|fetch|handle|on|use)[A-Z]'
    - label: "async functions"
      pattern: 'async\s+(function\s+\w+|\w+\s*=\s*async)'
    - label: "arrow functions"
      pattern: 'const\s+\w+\s*=\s*(\([^)]*\)|[a-zA-Z]+)\s*=>'
  sample_limit: 30

- id: ts-class-naming
  scope: typescript
  category: naming
  dimension: classes
  title: "Class and Interface Naming"
  analysis_type: grep_pattern
  grep_patterns:
    - label: "PascalCase classes"
      pattern: '(class|interface|type|enum)\s+[A-Z][a-zA-Z0-9]*'
    - label: "I-prefix interfaces"
      pattern: 'interface\s+I[A-Z][a-zA-Z0-9]*'
    - label: "T-prefix types"
      pattern: 'type\s+T[A-Z][a-zA-Z0-9]*'
    - label: "Suffix patterns (Service/Controller/Repository)"
      pattern: 'class\s+\w+(Service|Controller|Repository|Handler|Factory|Provider|Manager)'
  sample_limit: 30

- id: ts-imports
  scope: typescript
  category: structure
  dimension: imports
  title: "Import Organization"
  analysis_type: file_analysis
  check_patterns:
    - label: "grouped imports (blank line between groups)"
      check: import_grouping
    - label: "path aliases (@/ or ~/)"
      pattern: 'from\s+[''"][@~]/'
    - label: "barrel imports (index.ts re-exports)"
      check: barrel_exports
    - label: "type imports (import type)"
      pattern: 'import\s+type\s+'
  sample_limit: 30

- id: ts-error-handling
  scope: typescript
  category: patterns
  dimension: error-handling
  title: "Error Handling Patterns"
  analysis_type: grep_pattern
  grep_patterns:
    - label: "try-catch blocks"
      pattern: '\btry\s*\{'
    - label: "custom error classes"
      pattern: 'class\s+\w+Error\s+extends\s+(Error|AppError)'
    - label: "error logging"
      pattern: '(logger|console)\.(error|warn)\('
    - label: "Result/Either types"
      pattern: '(Result|Either|Success|Failure)<'
  sample_limit: 30

- id: ts-exports
  scope: typescript
  category: structure
  dimension: exports
  title: "Export Conventions"
  analysis_type: grep_pattern
  grep_patterns:
    - label: "named exports"
      pattern: 'export\s+(const|function|class|interface|type|enum)\s+'
    - label: "default exports"
      pattern: 'export\s+default\s+'
    - label: "barrel re-exports"
      pattern: 'export\s+(\*|\{[^}]+\})\s+from\s+[''"']'
    - label: "inline exports (export at declaration)"
      pattern: '^export\s+(const|function|class)'
  sample_limit: 30

- id: ts-docs
  scope: typescript
  category: docs
  dimension: jsdoc
  title: "Documentation Standards"
  analysis_type: grep_pattern
  grep_patterns:
    - label: "JSDoc comments"
      pattern: '/\*\*'
    - label: "@param tags"
      pattern: '@param\s+'
    - label: "@returns tags"
      pattern: '@returns?\s+'
    - label: "@throws/@example tags"
      pattern: '@(throws|example)\s+'
  sample_limit: 30
  consistency_base: exported_functions
```

#### **Include the following if detected stack includes React or tsx files exist in file_counts:**

```yaml
- id: react-components
  scope: react
  category: naming
  dimension: components
  title: "Component Naming and Structure"
  applicable_extensions: [tsx, jsx]
  analysis_type: grep_pattern
  grep_patterns:
    - label: "PascalCase components"
      pattern: '(export\s+)?(default\s+)?function\s+[A-Z][a-zA-Z0-9]*\s*\('
    - label: "arrow function components"
      pattern: '(export\s+)?const\s+[A-Z][a-zA-Z0-9]*\s*[=:]\s*.*=>\s*'
    - label: "Props type/interface"
      pattern: '(type|interface)\s+\w+Props\s*[={]'
    - label: "component file = component name"
      check: filename_matches_component
  sample_limit: 30

- id: react-hooks
  scope: react
  category: patterns
  dimension: hooks
  title: "Hooks Usage Patterns"
  applicable_extensions: [tsx, jsx, ts, js]
  analysis_type: grep_pattern
  grep_patterns:
    - label: "custom hooks (use* prefix)"
      pattern: '(export\s+)?(function|const)\s+use[A-Z][a-zA-Z0-9]*'
    - label: "hooks in dedicated files"
      check: hooks_file_pattern
    - label: "useCallback/useMemo usage"
      pattern: '(useCallback|useMemo)\s*\('
    - label: "useEffect cleanup"
      pattern: 'useEffect\s*\(\s*\(\)\s*=>\s*\{[\s\S]*return\s'
  sample_limit: 30

- id: react-state
  scope: react
  category: patterns
  dimension: state-management
  title: "State Management Patterns"
  applicable_extensions: [tsx, jsx, ts, js]
  analysis_type: grep_pattern
  grep_patterns:
    - label: "useState"
      pattern: 'useState\s*[<(]'
    - label: "useReducer"
      pattern: 'useReducer\s*[<(]'
    - label: "Zustand stores"
      pattern: '(create|useStore)\s*[<(]'
    - label: "Redux toolkit"
      pattern: '(createSlice|useSelector|useDispatch)\s*[<(]'
    - label: "React Query/TanStack"
      pattern: '(useQuery|useMutation)\s*[<(]'
  sample_limit: 30
```

#### **Include the following if detected stack includes Python:**

```yaml
- id: py-function-naming
  scope: python
  category: naming
  dimension: functions
  title: "Function Naming Conventions"
  applicable_extensions: [py]
  analysis_type: grep_pattern
  grep_patterns:
    - label: "snake_case functions"
      pattern: 'def\s+[a-z][a-z0-9_]*\s*\('
    - label: "type hints in signatures"
      pattern: 'def\s+\w+\(.*:\s*\w+.*\)\s*(->|:)'
    - label: "return type annotations"
      pattern: 'def\s+\w+\(.*\)\s*->\s*\w+'
    - label: "async functions"
      pattern: 'async\s+def\s+'
  sample_limit: 30

- id: py-class-naming
  scope: python
  category: naming
  dimension: classes
  title: "Class Naming Conventions"
  applicable_extensions: [py]
  analysis_type: grep_pattern
  grep_patterns:
    - label: "PascalCase classes"
      pattern: 'class\s+[A-Z][a-zA-Z0-9]*'
    - label: "Base/Abstract prefix"
      pattern: 'class\s+(Base|Abstract)[A-Z]'
    - label: "Mixin suffix"
      pattern: 'class\s+\w+Mixin'
    - label: "dataclass/pydantic models"
      pattern: '(@dataclass|class\s+\w+\(BaseModel\))'
  sample_limit: 30

- id: py-module-structure
  scope: python
  category: structure
  dimension: modules
  title: "Module Organization"
  applicable_extensions: [py]
  analysis_type: grep_pattern
  grep_patterns:
    - label: "__init__.py with exports"
      check: init_exports
    - label: "__all__ defined"
      pattern: '__all__\s*=\s*\['
    - label: "relative imports"
      pattern: 'from\s+\.\w*\s+import'
    - label: "absolute imports"
      pattern: 'from\s+[a-zA-Z]\w+(\.\w+)+\s+import'
  sample_limit: 30

- id: py-error-handling
  scope: python
  category: patterns
  dimension: error-handling
  title: "Error Handling Patterns"
  applicable_extensions: [py]
  analysis_type: grep_pattern
  grep_patterns:
    - label: "try-except blocks"
      pattern: '\btry\s*:'
    - label: "custom exception classes"
      pattern: 'class\s+\w+(Error|Exception)\s*\('
    - label: "specific exception catching"
      pattern: 'except\s+[A-Z]\w+(Error|Exception)'
    - label: "logging errors"
      pattern: '(logger|logging)\.(error|exception|warning)\('
  sample_limit: 30

- id: py-docs
  scope: python
  category: docs
  dimension: docstrings
  title: "Documentation Standards"
  applicable_extensions: [py]
  analysis_type: grep_pattern
  grep_patterns:
    - label: "triple-quote docstrings"
      pattern: '"""[\s\S]'
    - label: "Google-style docstrings (Args:)"
      pattern: '^\s+Args:\s*$'
    - label: "NumPy-style docstrings (Parameters)"
      pattern: '^\s+Parameters\s*$'
    - label: "Returns section"
      pattern: '^\s+Returns?:\s*$'
  sample_limit: 30
```

#### Scope filtering

If a `scope_filter` was specified in the input, keep only dimensions whose `scope` matches the filter **or** whose `scope` is `general`.

### Step 3: Analyze Each Dimension

For each dimension, collect evidence and calculate consistency score.

For each dimension, select the analysis method based on its `analysis_type`:

- If `analysis_type` is **"grep_pattern"** — use the Grep-based analysis procedure (below)
- If `analysis_type` is **"filename_pattern"** — use the Filename-based analysis procedure (below)
- If `analysis_type` is **"directory_pattern"** — use the Directory structure analysis procedure (below)
- If `analysis_type` is **"file_analysis"** — use the File-level analysis procedure (same as grep-based, but for check_patterns with custom checks)

**Grep-based analysis:**

**Step A — Find relevant files:** For each extension in the dimension's `applicable_extensions`, use **Glob** with `pattern=**/*.<ext>` and `path=<scan_path>`. Filter out non-source directories (see Helper: filter_source_files). Combine all results into `all_files`. Record `total_relevant_files` as the count. If zero files found, mark the dimension as skipped with reason "No relevant files found".

**Step B — Analyze each grep pattern:** For each entry in the dimension's `grep_patterns`:

- If the entry has a `check` key (custom check), run the corresponding custom check procedure (see Helper Functions) and record the count and sample files.
- Otherwise, use **Grep** with `pattern=<the regex pattern>`, `glob=*.{extensions}`, `path=<scan_path>`, and `output_mode="files_with_matches"`. Filter results through filter_source_files. Record matching file count and up to 5 sample files.
- If a Grep call fails, record count=0 with an error flag.

```text
COUNTING LOGIC (reference only — apply mentally, do NOT execute as code):

For each pattern result, compute:
  count = number of matching files
  pct   = count / total_relevant_files  (or 0 if total_relevant_files is 0)
  sample_files = first 5 matching files
```

**Step C — Sample code:** Collect code samples from top-matching files (see Step 4: Collect Code Samples below).

**Step D — Calculate confidence:** Apply the confidence calculation procedure (see Step 5 below).

**Step E — Determine dominant pattern:** The pattern with the highest `count` is the dominant pattern.

Assemble the result with: `skip=false`, `confidence`, `file_count`, `dominant_pattern`, `patterns_found`, `sample_code`, and up to 10 unique sample files across all patterns.

**Filename-based analysis:**

```text
FILENAME ANALYSIS LOGIC (reference only — apply mentally, do NOT execute as code):

1. For each extension in the dimension's applicable_extensions, use Glob with
   pattern=**/*.<ext> and path=<scan_path>. Filter through filter_source_files.
   Combine into all_files.

2. If no files found, mark dimension as skipped with reason "No files found".

3. Extract just the filename stem (without path and without extension) for each file.

4. Test each naming pattern regex from the dimension's patterns against each filename
   (append ".x" to the stem before matching, so the regex can match the extension part).

5. For each pattern, record:
     count    = number of matching filenames
     pct      = count / total filenames
     examples = first 5 matching filenames

6. The dominant pattern is the one with the highest count.

7. Build patterns_found list (only patterns with count > 0), sorted by count descending.

8. Return:
     skip             = false
     confidence       = dominant pattern's pct
     file_count       = total number of filenames
     dominant_pattern = "<pattern_name> (<pct as percentage>)"
     patterns_found   = sorted list from step 7
     sample_code      = ["# Examples: <first 5 dominant examples joined by comma>"]
     sample_files     = first 10 dominant examples
```

**Directory structure analysis:**

```text
DIRECTORY ANALYSIS LOGIC (reference only — apply mentally, do NOT execute as code):

1. Use Bash to list top-level and second-level directories:
     find <scan_path> -maxdepth 2 -type d | head -50
   Extract the basename of each directory path.

2. For each pattern group (feature-based, layer-based, type-based) in the dimension's
   patterns, check which marker directories (without trailing slash) appear in the
   discovered directory names.

3. For each pattern group with matches, record:
     pattern      = "<group_name> (<matched dirs joined by comma>)"
     count        = number of matched markers
     pct          = count / total markers in that group
     sample_files = list of matched directory names

4. If no patterns matched, mark dimension as skipped with reason
   "No recognized directory patterns".

5. The dominant pattern is the one with the highest pct.

6. Return:
     skip             = false
     confidence       = dominant pattern's pct
     file_count       = total number of discovered directory names
     dominant_pattern = dominant pattern string
     patterns_found   = all matched pattern groups
     sample_code      = ["# Structure: <dir>" for first 10 directory names]
     sample_files     = first 10 directory names
```

### Step 4: Collect Code Samples

1. **Rank files by pattern coverage:** For each file appearing in any pattern's `sample_files`, count how many patterns it matched. Sort files by this score (highest first). This prioritizes files that demonstrate multiple conventions at once.

2. **Read top-ranked files:** Iterate through ranked files (up to `sample_limit`), skipping duplicates. For each file, use **Read** to get its content.

3. **Extract relevant snippet:** For each file's content, scan line by line looking for matches against the dimension's grep patterns. When a match is found, extract that line plus the next 3-5 lines as a snippet. Cap snippet length at 200 characters (append "..." if truncated).

4. **Collect up to 6 snippets total.** Stop once 6 valid snippets have been collected, even if more ranked files remain.

### Step 5: Calculate Confidence Score

```text
SCORING LOGIC (reference only — apply mentally, do NOT execute as code):

Confidence is based on:
  1. Dominant pattern percentage (primary factor)
  2. Number of files matching (minimum threshold)
  3. Pattern clarity (gap between #1 and #2 pattern)

If no patterns found or total_files == 0:
  return 0.0

Sort patterns by count (descending).

dominant_pct = sorted_patterns[0].pct

Minimum file threshold — at least 5 files must match:
  if sorted_patterns[0].count < 5:
    dominant_pct *= 0.5    (penalize small sample)

Clarity bonus — if dominant pattern is clearly ahead:
  if there are >= 2 patterns:
    gap = sorted_patterns[0].pct - sorted_patterns[1].pct
    if gap > 0.3:
      dominant_pct = min(1.0, dominant_pct * 1.1)    (slight bonus for clarity)

return round(dominant_pct, 2)
```

### Step 6: Assemble Discovery Plan

For each analyzed dimension, skip it if it was marked as skipped or its confidence is below `min_confidence / 100`. For remaining dimensions, look up the suggested type using the dimension-to-type mapping:

| Dimension | Suggested Type |
|-----------|---------------|
| functions | convention |
| variables | convention |
| components | convention |
| classes | convention |
| modules | convention |
| files | convention |
| imports | pattern |
| exports | pattern |
| error-handling | pattern |
| jsdoc | style |
| docstrings | style |
| state-management | pattern |
| hooks | pattern |
| directories | environment |
| project-structure | environment |
| config-files | environment |
| ci-cd | environment |
| docker | environment |
| toolchain | environment |

If a dimension is not in the table, default to `convention`.

For each qualifying dimension, assemble a discovered entry with: `id`, `scope`, `category`, `dimension`, `suggested_type`, `title`, `confidence`, `file_count`, `dominant_pattern`, `evidence` (containing `patterns_found`, `sample_files`, `sample_code`), `linter_rules` (from Helper: extract_relevant_linter_rules), and `suggested_tags` (from Step 7).

```text
SORTING LOGIC (reference only — apply mentally, do NOT execute as code):

Sort discovered entries by:
  Primary:   confidence descending
  Secondary: file_count descending

After sorting, re-assign sequential IDs starting from 1.

Final output structure:
  project_stack        = stack from Step 1
  discovered           = sorted list of discovered entries
  total_files_analyzed = sum of all values in stack.file_counts
```

### Step 7: Generate Tags

```text
TAG GENERATION LOGIC (reference only — apply mentally, do NOT execute as code):

Start with an empty set of tags.

1. Add the dimension's scope (e.g., "typescript", "python", "general")
2. Add the dimension's category (e.g., "naming", "patterns", "structure")
3. Check the dominant_pattern string (case-insensitive) against these keywords:

     Keyword      →  Tag
     ─────────────────────────
     camelCase    →  camelCase
     PascalCase   →  PascalCase
     snake_case   →  snake_case
     kebab-case   →  kebab-case
     try-catch    →  error-handling
     async        →  async
     hooks        →  hooks
     useState     →  state
     import       →  imports

   For each keyword found in the dominant pattern string, add the corresponding tag.

4. Add the dimension's dimension name (e.g., "functions", "classes")

5. Sort tags alphabetically and return at most 8 tags.
```

## Output Format

Return structured text to the agent (not JSON, following plugin conventions):

```
=== DISCOVERY PLAN ===

Project Stack:
  Languages: typescript, python
  Frameworks: react, fastapi
  Tools: .eslintrc.json, tsconfig.json, .prettierrc
  Files: ts=120, tsx=67, py=98

Total Files Analyzed: 342

--- Discovered Convention ---
ID: 1
Scope: typescript
Category: naming
Dimension: functions
Title: Function Naming Conventions
Confidence: 0.94
File Count: 156
Dominant Pattern: camelCase with verb prefix
Patterns:
  - camelCase functions: 147 files (94%)
  - verb prefix (get/set/is/has): 132 files (85%)
  - async functions: 89 files (57%)
  - arrow functions: 134 files (86%)
Linter Rules: camelcase=error
Tags: typescript, naming, functions, camelCase
Sample Code:
  ```typescript
  async function fetchUserProfile(userId: string): Promise<UserProfile> {
  const isValidEmail = (email: string): boolean =>
  function getUserSettings(userId: string): Settings {
  ```
Sample Files: src/services/userService.ts, src/utils/format.ts

--- Discovered Convention ---
ID: 2
...

=== END PLAN ===
```

The agent parses this text format to build the interactive plan display.

## Helper Functions

**Helper: filter_source_files**

When filtering file lists, exclude any file whose path contains a directory segment matching any of these names:

- node_modules
- dist
- build
- .next
- out
- __pycache__
- .venv
- venv
- env
- .git
- .svn
- vendor
- target
- .tox
- .mypy_cache
- .pytest_cache
- coverage
- .nyc_output
- .kb (don't scan knowledge base itself)

**Helper: extract_relevant_linter_rules**

Extract linter rules relevant to the current dimension:

1. **ESLint naming rules**: If `eslint` rules exist and the dimension's category includes "naming", filter ESLint rules whose keys contain any of: `camel`, `pascal`, `naming`, `id-`, `func-name`. Add matching rules to the result.

2. **TypeScript compiler options**: If `typescript` rules exist and the dimension's scope is "typescript", extract these keys if present: `strict`, `noImplicitAny`, `strictNullChecks`, `paths`, `baseUrl`. Add them to the result.

**Helper: run_custom_check**

Custom checks that cannot be done with simple grep. For each check, return a count and sample_files list.

1. **"import_grouping"** — Check if imports are separated by blank lines into groups:
   - Use **Read** on up to 30 files from the file list
   - Extract lines starting with `import ` or `from `
   - Check if there is a blank line (`\n\n`) between import lines (indicating grouped imports)
   - Count files where grouping is detected AND there are more than 3 import lines
   - Record matching files as sample_files

2. **"barrel_exports"** — Check for index.ts files with re-exports:
   - Use **Glob** with `pattern=**/index.{ts,js}` and `path=<scan_path>`
   - Filter results through filter_source_files
   - Use **Read** on up to 20 index files
   - Count files whose content contains both `export` and `from`
   - Record matching files as sample_files

3. **"init_exports"** — Check for __init__.py with exports:
   - Use **Glob** with `pattern=**/__init__.py` and `path=<scan_path>`
   - Filter results through filter_source_files
   - Use **Read** on up to 20 init files
   - Count files that have non-empty content
   - Record matching files as sample_files

4. **"hooks_file_pattern"** — Check if hooks are in dedicated files:
   - Use **Glob** with `pattern=**/use*.{ts,tsx,js,jsx}` and `path=<scan_path>`
   - Filter results through filter_source_files
   - Return count as total matching files, sample_files as first 5

5. **"filename_matches_component"** — Check if component filename matches the default export:
   - For up to 30 files from the file list, extract the filename stem (without path/extension)
   - Use **Read** to get each file's content
   - Check if content contains `function <filename>` or `const <filename>`
   - Count matching files and record as sample_files

If the check name is not recognized, return count=0 and empty sample_files.

## Performance Guidelines

**File limits:**
- Glob results: process first 500 files per extension
- Grep: use `--max-count=1` per file (just need presence, not all matches)
- Read: sample max 30 files per dimension for code extraction
- Total Read calls: aim for < 100 across entire scan

**Caching:**
- Reuse Glob results across dimensions with same extensions
- Cache file_counts from Step 1 for all subsequent steps
- Share linter config reads across dimensions

**Parallelism hints:**
- Dimensions are independent — agent can process them in any order
- Group dimensions by applicable_extensions to minimize Glob calls

## Edge Cases

**Monorepo:**
- If multiple package.json found, detect monorepo
- Suggest scanning each package separately
- Report: "Monorepo detected. Consider: /knowledge-scan packages/api/"

**Mixed conventions:**
- If no pattern > 50%, report as "inconsistent" rather than "low confidence"
- Suggest: "Codebase has mixed patterns. Consider establishing explicit convention."

**Empty/small project:**
- If < 10 source files total: warn about small sample size
- Reduce confidence scores by 0.8x multiplier

**Generated code:**
- Detect common generated file markers:
  - "// auto-generated", "# Generated by", "DO NOT EDIT"
  - Files in generated/, __generated__/, .generated/
- Exclude from analysis
