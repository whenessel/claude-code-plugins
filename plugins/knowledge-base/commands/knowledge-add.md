---
name: knowledge-add
description: Create a standardized knowledge entry from raw guidelines
argument-hint: "[scope:X] [category:Y] <guidelines text, file, directory, or URL>"
---

# Knowledge Add Command

Create a standardized knowledge entry in `.kb/` directory.

## Usage

```bash
/knowledge-add <guidelines>
/knowledge-add scope:typescript category:naming <guidelines>
/knowledge-add https://example.com/style-guide.md
/knowledge-add ./docs/guidelines.md scope:react
```

## Implementation

### Parse Arguments

1. **Auto-detect file paths/URLs** in arguments:
   ```python
   def detect_file_source(args: list[str]) -> tuple[str, str, list[str]]:
       """
       Returns: (source_type, source_path, remaining_args)
       source_type: "url", "directory", "path", or None
       """

       # Check for URLs
       url_pattern = r'https?://[^\s]+'
       for arg in args:
           if re.match(url_pattern, arg):
               remaining = [a for a in args if a != arg]
               return ("url", arg, remaining)

       # Check for directories FIRST (before files)
       for arg in args:
           if '/' in arg and os.path.isdir(arg):
               remaining = [a for a in args if a != arg]
               return ("directory", arg, remaining)

       # Check for file paths (.md, .txt, contains /)
       for arg in args:
           if (arg.endswith(('.md', '.txt')) or '/' in arg):
               if os.path.exists(arg):
                   remaining = [a for a in args if a != arg]
                   return ("path", arg, remaining)

       return (None, None, args)
   ```

2. **Read file source** if detected:
   ```python
   if source_type == "url":
       content = WebFetch(
           url=source_path,
           prompt="Extract full content preserving structure"
       )

   elif source_type == "path":
       # Security: no .. in path
       if ".." in source_path:
           error("Path cannot contain '..'")

       # Size limit: 10MB, 10k lines
       if file_size > 10*1024*1024:
           error("File too large")

       content = Read(source_path)
   ```

3. **Parse explicit scope/category**:
   ```python
   # Extract "scope:typescript" or "category:naming"
   for arg in remaining_args:
       if match := re.match(r'scope:(\w+)', arg):
           explicit_scope = match.group(1)
       elif match := re.match(r'category:(\w+)', arg):
           explicit_category = match.group(1)

   # Remove from remaining_args
   ```

4. **Remaining = guidelines**:
   ```python
   guidelines = ' '.join(remaining_args)
   ```

### Invoke Agent

Pass collected data to knowledge-writer agent:

```python
# Build prompt based on source type
if source_type == "directory":
    prompt = f"""
    Batch process all markdown files in directory: {source_path}

    For each .md file in the directory:
    1. Read the file content
    2. Create a convention from its guidelines
    3. Track results (success/failure)

    {"Explicit scope: " + explicit_scope if explicit_scope else ""}
    {"Explicit category: " + explicit_category if explicit_category else ""}

    Filter: Only process files ending in .md, exclude README.md
    """
else:
    prompt = f"""
    Create a knowledge entry from the following guidelines:

    {guidelines if not content else content}

    {"Explicit scope: " + explicit_scope if explicit_scope else ""}
    {"Explicit category: " + explicit_category if explicit_category else ""}
    {"Source: " + source_path if source_path else ""}
    """

Task(
  description="Create or update knowledge entry(s)",
  prompt=prompt,
  subagent_type="knowledge-writer"
)
```

### Error Handling

**No arguments**:
```
Show usage help:

Usage: /knowledge-add [scope:X] [category:Y] <guidelines or file/URL>

Options:
  scope:SCOPE       Explicit scope (typescript, python, react, etc.)
  category:CAT      Explicit category (naming, rules, patterns, etc.)

Examples:
  /knowledge-add TypeScript functions: camelCase, verb-based
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

**Example 2: With explicit scope/category**
```bash
/knowledge-add scope:react category:patterns Hooks naming: start with use, camelCase
```

**Example 3: From URL (auto-detected)**
```bash
/knowledge-add https://github.com/airbnb/javascript/blob/master/README.md
```

**Example 4: From file (auto-detected)**
```bash
/knowledge-add ./docs/api-guidelines.md scope:typescript
```

**Example 5: Update existing convention**
```bash
/knowledge-add Add rule: functions should document thrown errors
# Updates existing convention, version 1.0 → 1.1
```

**Example 6: Codebase analysis (Russian trigger)**
```bash
/knowledge-add Изучи стиль кода в TypeScript файлах и сформулируй конвенцию именования
# Triggers codebase analysis
```

**Example 7: Batch process directory**
```bash
/knowledge-add ./docs/guidelines/ scope:typescript
# Processes all .md files in directory
```

**Example 8: Directory with category**
```bash
/knowledge-add ./docs/naming-conventions/ scope:react category:naming
# All files use same scope and category
```
