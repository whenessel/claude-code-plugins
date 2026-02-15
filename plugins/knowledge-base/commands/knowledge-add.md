---
name: knowledge-add
description: Create a standardized knowledge entry from raw guidelines
argument-hint: "[type:X] [scope:Y] [category:Z] <guidelines text, file, directory, or URL>"
---

> **EXECUTION CONSTRAINT**: All code blocks below are pseudocode for reference only.
> - **NEVER** create or execute `.py` scripts
> - **NEVER** use Bash to run `python` or `python3` commands
> - Implement all logic using Claude's built-in tools: Read, Grep, Glob, Write, Edit, Bash, Task
> - Parse YAML/JSON content mentally from Read tool output — do NOT use Python yaml/json libraries
> - `invoke_skill()` / `invoke_agent()` in pseudocode = use **Task** tool with appropriate subagent_type

# Knowledge Add Command

Create a standardized knowledge entry in `.kb/` directory.

## Usage

```bash
/knowledge-add <guidelines>
/knowledge-add type:rule scope:typescript Max file size: 800 lines
/knowledge-add scope:typescript category:naming <guidelines>
/knowledge-add type:environment Node.js project setup: npm init, eslint, prettier, husky
/knowledge-add https://example.com/style-guide.md
/knowledge-add ./docs/guidelines.md scope:react
```

## Implementation

### Parse Arguments

1. **Auto-detect file paths/URLs** in arguments:
   Detect the source type from arguments using this decision tree:

   - **URL check**: Scan arguments for one matching the pattern `https?://[^\s]+`. If found, source_type is `"url"`, remove it from remaining args.
   - **Directory check**: For each argument containing `/`, use **Bash** `test -d {arg}` to check if it is a directory. If found, source_type is `"directory"`, remove it from remaining args.
   - **File path check**: For each argument ending with `.md` or `.txt`, or containing `/`, use **Read** to check if the file exists (handle error for missing). If found, source_type is `"path"`, remove it from remaining args.
   - **No source detected**: If none of the above matched, source_type is `None`, all args remain.

2. **Read file source** if detected:
   - If source_type is `"url"`: Use **WebFetch** with the URL and prompt "Extract full content preserving structure"
   - If source_type is `"path"`:
     - **Security check**: If the path contains `..`, report error: "Path cannot contain '..'"
     - **Size limit**: 10MB max, 10k lines max
     - Use **Read** to load the file content

3. **Parse explicit type/scope/category**:
   Scan each remaining argument for explicit parameter patterns and extract values:

   - Match `type:(\w+)` — extract as `explicit_type`. Validate against allowed types: convention, rule, pattern, guide, documentation, reference, style, environment. If invalid, report error with the list of valid types.
   - Match `scope:(\w+)` — extract as `explicit_scope`
   - Match `category:(\w+)` — extract as `explicit_category`

   Remove any matched parameter arguments from the remaining arguments list.

4. **Remaining = guidelines**:
   Combine all remaining arguments (after removing source and explicit parameters) into a single guidelines text string, joined by spaces.

### Invoke Agent

Pass collected data to knowledge-writer agent:

Use the **Task** tool to delegate to the `knowledge-writer` agent. Build the prompt based on source type:

- **If source_type is "directory"**: Include the directory path and instruct the agent to batch-process all `.md` files (excluding README.md), reading each file and creating a knowledge entry. Include any explicit type, scope, and category if provided.
- **Otherwise**: Include the guidelines text (or fetched content if from URL/file) and any explicit type, scope, category, and source path.

The Task description should be "Create or update knowledge entry(s)" with `subagent_type="knowledge-writer"`.

### Error Handling

**No arguments**:
```
Show usage help:

Usage: /knowledge-add [type:X] [scope:Y] [category:Z] <guidelines or file/URL>

Options:
  type:TYPE         Explicit type (convention, rule, pattern, guide, documentation, reference, style, environment)
  scope:SCOPE       Explicit scope (typescript, python, react, etc.)
  category:CAT      Explicit category (naming, rules, patterns, etc.)

Examples:
  /knowledge-add TypeScript functions: camelCase, verb-based
  /knowledge-add type:rule scope:typescript Max file size: 800 lines
  /knowledge-add type:pattern Error handling best practices
  /knowledge-add type:environment Node.js project setup: npm init, eslint, prettier, husky
  /knowledge-add scope:react category:patterns Hooks naming: start with use
  /knowledge-add https://github.com/airbnb/javascript/blob/master/README.md
  /knowledge-add ./docs/guidelines.md scope:typescript

Description:
  Create a standardized knowledge entry in .kb/ directory.
  Guidelines can be in English or Russian - output is always in English.
  Output path is always auto-generated: .kb/{scope}/{category}/{name}.md
```

**Invalid file path**:
```
If path contains ..:
  Error: "Path cannot contain '..' for security reasons"

If file too large (> 10MB):
  Error: "File too large (max 10MB)"
```

### Display Results

After agent completes:

1. **Success case (new convention):**
   ```
   ✓ Convention created: .kb/typescript/naming/functions.md
   Version: 1.0
   Lines: 180-250
   Sections: 8+

   Run /rebuild-index to update the conventions catalog.
   ```

2. **Success case (updated convention - minor):**
   ```
   ✓ Convention updated: .kb/typescript/naming/functions.md
   Version: 1.0 → 1.1
   Changes: Added 2 rules to Rules section
   ```

3. **Confirmation needed (major update):**
   ```
   Detected existing convention: .kb/typescript/patterns/error-handling.md
   Version: 1.2

   Proposed changes:
   - Complete rewrite of Rules section
   - Version: 1.2 → 2.0 (major)

   Proceed? [y/n]
   ```

4. **Scope/category ambiguous:**
   ```
   Question: Which language/framework does this convention apply to?

   Detected "error handling" - could apply to:
   Options:
   1. TypeScript (detected, 65% confidence)
   2. Python
   3. General (language-agnostic)
   ```

5. **Validation warnings:**
   ```
   ✓ Convention created: .kb/typescript/patterns/complex.md

   ⚠ Warnings:
   - Line count (185) is near comprehensive tier limit
   - Custom scope 'golang' created (not predefined)
   ```

## Notes

- **Multi-language support**: Guidelines can be in Russian or English; output is always in English
- **Auto-detect**: URLs and file paths are automatically detected in arguments
- **Updates**: Existing conventions are updated instead of creating conflicts
- **Auto-path**: Output path is always auto-generated from scope/category/title
- **Skill invocation**: Command delegates to knowledge-writer agent, which invokes knowledge-draft skill

## Examples

**Example 1: Simple text input**
```bash
/knowledge-add TypeScript async functions must have try-catch blocks
```

**Example 2: With explicit type**
```bash
/knowledge-add type:rule scope:typescript Max file size: 800 lines
```

**Example 3: With explicit scope/category**
```bash
/knowledge-add scope:react category:patterns Hooks naming: start with use, camelCase
```

**Example 4: From URL (auto-detected)**
```bash
/knowledge-add https://github.com/airbnb/javascript/blob/master/README.md
```

**Example 5: From file (auto-detected)**
```bash
/knowledge-add ./docs/api-guidelines.md scope:typescript
```

**Example 6: Update existing convention**
```bash
/knowledge-add Add rule: functions should document thrown errors
# Updates existing convention, version 1.0 → 1.1
```

**Example 7: Codebase analysis (Russian trigger)**
```bash
/knowledge-add Изучи стиль кода в TypeScript файлах и сформулируй конвенцию именования
# Triggers codebase analysis
```

**Example 8: Batch process directory**
```bash
/knowledge-add ./docs/guidelines/ scope:typescript
# Processes all .md files in directory
```

**Example 9: Directory with category**
```bash
/knowledge-add ./docs/naming-conventions/ scope:react category:naming
# All files use same scope and category
```
