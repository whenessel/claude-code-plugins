---
name: knowledge-draft
description: >
  Transform raw coding guidelines into standardized knowledge entry files.
  Handles parsing, optional codebase/web research, structure formalization,
  markdown generation, validation, and file writing. Produces .kb/
  files with YAML front matter, Format section, code examples, and cross-refs.
allowed-tools: Read, Grep, Glob, Write, Bash
---

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
```python
def should_research_web(guidelines: str, rule_count: int) -> bool:
    # Russian or English triggers
    ru_triggers = ["лучшие практики", "стандарты индустрии", "рекомендуемый подход", "как лучше"]
    en_triggers = ["best practices", "industry standard", "recommended approach", "state of the art"]

    has_trigger = any(t in guidelines.lower() for t in ru_triggers + en_triggers)

    # Low quality input (< 3 rules, no examples)
    low_quality = rule_count < 3 and "example" not in guidelines.lower()

    # Explicit skip
    skip = any(p in guidelines.lower() for p in
               ["skip research", "no research", "без исследования"])

    return (has_trigger or low_quality) and not skip
```

### Step 3: Conduct Research

**A. Codebase Analysis (ONLY if triggered):**

If codebase analysis triggered by keywords:

1. **Generate search patterns** based on topic:
   ```python
   def generate_patterns(topic: str) -> list[str]:
       patterns = {
           "error handling": ["try", "catch", "throw", "Error("],
           "function naming": ["function ", "const.*=>", "async function"],
           "imports": ["import ", "from '", "require("]
       }

       for key, pats in patterns.items():
           if key in topic.lower():
               return pats

       return topic.split()  # Generic: topic words
   ```

2. **Search codebase** using Grep:
   ```python
   results = []
   for pattern in search_patterns:
       grep_result = Grep(
           pattern=pattern,
           type=file_types[scope],  # e.g., "ts,tsx"
           output_mode="files_with_matches",
           head_limit=20
       )
       results.extend(grep_result)
   ```

3. **Read examples** (up to 20 files):
   ```python
   examples = []
   for file_path in unique_files[:20]:
       content = Read(file_path)
       relevant = extract_relevant_code(content, topic)
       examples.append({"path": file_path, "snippet": relevant})
   ```

4. **Analyze patterns**:
   ```python
   # Statistical analysis of naming, structure, etc.
   analysis = analyze_code_patterns(examples, topic)
   ```

5. **Check linter configs**:
   ```python
   configs = Glob(pattern="**/{.eslintrc*,tsconfig.json,.prettierrc*}")
   rules = extract_relevant_rules(configs, topic)
   ```

6. **Format as guidelines**:
   ```python
   codebase_guidelines = f"""
   Discovered patterns from codebase:
   - {analysis['dominant_pattern']}
   - Found in {len(examples)} files

   Examples:
   {examples[:3]}

   Linter rules:
   {rules}
   """
   ```

**B. Web Research (if triggered):**

Use WebSearch for best practices:

```python
query = f"{topic} best practices 2026"
if input_language == "russian":
    query = f"{translate_to_english(topic)} best practices 2026"

results = WebSearch(query, limit=3)
```

**Merge findings:**
```
Priority:
1. User's explicit rules (highest)
2. Codebase patterns (if analyzed)
3. Web best practices (supplement)
```

### Step 4: Fixed Comprehensive Tier

**ALWAYS generate Comprehensive tier:**

```python
tier = "comprehensive"  # Fixed, no detection needed
target_lines = 150-300
required_sections = [
    "Title",
    "Description (2-3 paragraphs)",
    "Format (4+ bullet points)",
    "Domain sections with subsections",
    "Allowed (4+ examples)",
    "Forbidden (4+ anti-patterns)",
    "Rules (7+ guidelines)",
    "Summary table"
]
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

```python
def detect_scope(guidelines: str, explicit_scope: str = None) -> tuple[str, float]:
    if explicit_scope:
        return explicit_scope, 1.0  # User-specified

    # Indicators for each scope
    scope_indicators = {
        "typescript": ["typescript", "ts", ".ts", "type", "interface"],
        "python": ["python", "py", "def ", "class ", "import "],
        "react": ["react", "component", "jsx", "tsx", "hook"],
        "fastapi": ["fastapi", "pydantic", "endpoint"],
        "general": ["any language", "language-agnostic"]
    }

    # Score each scope
    scores = {}
    for scope, indicators in scope_indicators.items():
        count = sum(1 for ind in indicators if ind in guidelines.lower())
        scores[scope] = count / len(indicators)

    best_scope = max(scores, key=scores.get)
    confidence = scores[best_scope]

    return best_scope, confidence

# Usage
scope, confidence = detect_scope(guidelines, explicit_scope)

if confidence < 0.7 and not explicit_scope:
    # Return to agent for AskUserQuestion
    return {
        "status": "scope_ambiguous",
        "detected": scope,
        "confidence": confidence,
        "options": list(scope_indicators.keys())
    }
else:
    use_detected_scope(scope)
```

**Category detection with confidence:**

```python
def detect_category(guidelines: str, topic: str, explicit_category: str = None) -> tuple[str, float]:
    if explicit_category:
        return explicit_category, 1.0

    category_indicators = {
        "naming": ["name", "naming", "camelCase", "snake_case"],
        "rules": ["limit", "constraint", "must not exceed", "maximum"],
        "docs": ["document", "comment", "jsdoc", "docstring"],
        "structure": ["organize", "folder", "directory", "file structure"],
        "patterns": ["pattern", "practice", "handle", "error", "state"]
    }

    combined = (guidelines + " " + topic).lower()

    scores = {}
    for category, indicators in category_indicators.items():
        count = sum(1 for ind in indicators if ind in combined)
        scores[category] = count / len(indicators)

    # Check for ambiguity
    sorted_scores = sorted(scores.values(), reverse=True)
    if len(sorted_scores) > 1 and sorted_scores[0] - sorted_scores[1] < 0.2:
        # Top 2 categories too close
        return max(scores, key=scores.get), 0.5

    best = max(scores, key=scores.get)
    return best, scores[best]

# Usage
category, confidence = detect_category(guidelines, topic, explicit_category)

if confidence < 0.7 and not explicit_category:
    return {
        "status": "category_ambiguous",
        "detected": category,
        "confidence": confidence,
        "options": list(category_indicators.keys())
    }
else:
    use_detected_category(category)
```

**Type detection with confidence:**

```python
def detect_type(guidelines: str, topic: str, explicit_type: str = None) -> tuple[str, float]:
    """Detect knowledge entry type from content analysis."""

    if explicit_type:
        return explicit_type, 1.0

    combined = (guidelines + " " + topic).lower()

    type_indicators = {
        "rule": {
            "strong": ["must not", "must never", "forbidden", "prohibited",
                       "limit", "maximum", "minimum", "constraint", "enforce",
                       "required", "mandatory", "not allowed"],
            "weak": ["should not", "avoid", "restrict", "cap at", "ceiling"]
        },
        "pattern": {
            "strong": ["pattern", "architecture", "design pattern", "approach",
                       "error handling", "state management", "data flow"],
            "weak": ["practice", "strategy", "technique", "method"]
        },
        "guide": {
            "strong": ["how to", "step by step", "tutorial", "walkthrough",
                       "getting started", "workflow", "instructions"],
            "weak": ["migrate", "upgrade", "follow these steps"]
        },
        "documentation": {
            "strong": ["documentation", "document this", "knowledge article",
                       "describe how", "explanation of", "overview of",
                       "save this documentation", "docs block"],
            "weak": ["docs", "describe", "explain", "context", "background"]
        },
        "reference": {
            "strong": ["reference", "specification", "api docs", "cheat sheet",
                       "options", "parameters", "configuration reference"],
            "weak": ["schema", "interface", "types", "lookup"]
        },
        "style": {
            "strong": ["formatting", "indentation", "spacing", "prettier",
                       "code style", "lint rule", "visual style"],
            "weak": ["format", "indent", "whitespace", "semicolons"]
        },
        "environment": {
            "strong": ["setup", "scaffold", "boilerplate", "environment",
                       "infrastructure", "ci/cd", "pipeline", "deploy",
                       "project initialization", "dev environment"],
            "weak": ["configure", "install", "docker", "terraform",
                      "github actions", ".env", "toolchain"]
        },
        "convention": {
            "strong": ["convention", "naming", "camelCase", "PascalCase",
                       "snake_case", "agreed", "team standard"],
            "weak": ["prefer", "consistently", "standardize", "uniform"]
        }
    }

    scores = {}
    for entry_type, indicators in type_indicators.items():
        strong_count = sum(1 for ind in indicators["strong"] if ind in combined)
        weak_count = sum(1 for ind in indicators["weak"] if ind in combined)
        score = (strong_count * 2 + weak_count) / (len(indicators["strong"]) + len(indicators["weak"]))
        scores[entry_type] = score

    best_type = max(scores, key=scores.get)
    confidence = scores[best_type]

    # If nothing detected clearly, default to convention
    if confidence < 0.1:
        return "convention", 0.5

    return best_type, min(1.0, confidence + 0.3)

# Usage
entry_type, confidence = detect_type(guidelines, topic, explicit_type)

if confidence < 0.7 and not explicit_type:
    return {
        "status": "type_ambiguous",
        "detected": entry_type,
        "confidence": confidence,
        "options": ["convention", "rule", "pattern", "guide", "documentation", "reference", "style", "environment"]
    }
else:
    use_detected_type(entry_type)
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

```python
scope = metadata["scope"]
category = metadata["category"]
name = convert_to_kebab_case(title)
output_path = f".kb/{scope}/{category}/{name}.md"
```

**Check for existing knowledge entry:**

```python
if file_exists(output_path):
    existing_content = read_file(output_path)
    existing_version = extract_version(existing_content)

    # Analyze user intent
    intent = analyze_update_intent(guidelines)

    if intent == "append_rules":
        strategy = "append"
        increment = "minor"

    elif intent == "update_values":
        strategy = "update_fields"
        increment = "minor"

    elif intent == "rewrite_section":
        strategy = "rewrite"
        increment = "minor"  # or "major" if Rules/Format section

    elif intent == "full_overwrite":
        strategy = "overwrite"
        increment = "major"

    else:  # ambiguous
        # Ask user for clarification
        return {"status": "update_confirmation_needed", ...}

    new_version = increment_version(existing_version, increment)

    return {
        "status": "update_ready",
        "strategy": strategy,
        "current_version": existing_version,
        "new_version": new_version,
        "path": output_path,
        "requires_confirmation": increment == "major"
    }

else:
    # New knowledge entry
    return {"status": "create_new", "path": output_path, "version": "1.0"}
```

**Intent analysis patterns:**

```python
def analyze_update_intent(guidelines: str) -> str:
    g = guidelines.lower()

    # Order matters - most specific first
    overwrite = ["completely rewrite", "полностью переписать"]
    rewrite = ["improve examples", "enhance examples", "улучши примеры"]
    append = ["add rule", "добавь правило", "include also", "ещё один"]
    update = ["change", "update", "increase", "decrease", "изменить", "увеличить"]

    if any(p in g for p in overwrite):
        return "full_overwrite"
    elif any(p in g for p in rewrite):
        return "rewrite_section"
    elif any(p in g for p in update):
        return "update_values"
    elif any(p in g for p in append):
        return "append_rules"
    else:
        return "ambiguous"
```

### Step 9: Execute File Operation

**For NEW conventions:**

```python
os.makedirs(os.path.dirname(output_path), exist_ok=True)
write_file(output_path, yaml_frontmatter + "\n\n" + content_sections)
```

**For UPDATES:**

```python
def execute_update(strategy, output_path, existing, new_content, new_version):

    if strategy == "append":
        # Add new rules to Rules section
        sections = parse_markdown_sections(existing)
        new_rules = new_content["rules"]
        sections["Rules"].extend(new_rules)
        sections["yaml"]["version"] = new_version
        updated = rebuild_markdown(sections)
        write_file(output_path, updated)

    elif strategy == "update_fields":
        # Replace specific values in Format section
        updated = existing
        for field, new_value in new_content["updates"].items():
            updated = replace_field_value(updated, field, new_value)
        updated = update_yaml_field(updated, "version", new_version)
        write_file(output_path, updated)

    elif strategy == "rewrite":
        # Replace specific section(s)
        sections = parse_markdown_sections(existing)
        target = new_content["target_section"]
        sections[target] = new_content["new_section_content"]
        sections["yaml"]["version"] = new_version
        updated = rebuild_markdown(sections)
        write_file(output_path, updated)

    elif strategy == "overwrite":
        # Complete replacement
        write_file(output_path, new_content["complete_file"])
```

**Version increment logic:**

```python
def increment_version(current: str, increment_type: str) -> str:
    major, minor = map(int, current.split('.'))

    if increment_type == "major":
        return f"{major + 1}.0"
    else:  # minor
        return f"{major}.{minor + 1}"
```

**Helper functions:**

```python
def parse_markdown_sections(content: str) -> dict:
    """Parse markdown into {yaml, section_name: [lines]} dict."""
    sections = {}

    # Extract YAML
    yaml_match = re.match(r'^---\n(.*?)\n---\n', content, re.DOTALL)
    sections["yaml"] = yaml.safe_load(yaml_match.group(1))

    # Parse sections by ## headers
    current = None
    for line in content[yaml_match.end():].split('\n'):
        if line.startswith('## '):
            current = line[3:].strip()
            sections[current] = []
        elif current:
            sections[current].append(line)

    return sections

def rebuild_markdown(sections: dict) -> str:
    """Rebuild markdown from sections dict."""
    yaml_str = yaml.dump(sections["yaml"], default_flow_style=False)
    result = f"---\n{yaml_str}---\n\n"

    for section, lines in sections.items():
        if section == "yaml":
            continue
        result += f"## {section}\n\n"
        result += '\n'.join(lines) + "\n\n"

    return result
```

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

```python
try:
    # Invoke quality evaluation skill in fast mode with text-based prompt
    prompt = f"""Evaluate knowledge entry quality for file: {output_path}
Mode: fast"""

    result_text = invoke_skill(skill="knowledge-evaluate", prompt=prompt)

    # Parse text output (no JSON deserialization!)
    # Expected format: "Quality: 8.4/10 (Clarity: 8, Format: 9, Structure: 9, Completeness: 8, Efficiency: 8)"
    match = re.search(
        r'Quality: ([\d.]+)/10 \(Clarity: (\d+), Format: (\d+), Structure: (\d+), Completeness: (\d+), Efficiency: (\d+)\)',
        result_text
    )

    if match:
        overall_score = float(match.group(1))
        scores = {
            'clarity': int(match.group(2)),
            'format': int(match.group(3)),
            'structure': int(match.group(4)),
            'completeness': int(match.group(5)),
            'efficiency': int(match.group(6))
        }

        quality_summary = result_text.strip()  # Use formatted text directly

        # Store in result
        result = {
            "status": "success",
            "path": output_path,
            "version": version,
            "tier": tier,
            "line_count": line_count,
            "sections": sections_count,
            "examples": examples_count,
            "quality": quality_summary,
            "quality_score": overall_score,
            "quality_scores": scores,
            "validation": validation_result
        }

        # Add warning if quality is low
        if overall_score < 7.0:
            result["quality_warning"] = (
                "⚠ Quality below target. Run /knowledge-report for detailed analysis."
            )

        return result

    else:
        # Couldn't parse result - continue without quality metrics
        log_warning(f"Quality evaluation format error: {result_text}")

except Exception as e:
    # Don't fail knowledge entry creation if evaluation fails
    log_warning(f"Quality evaluation skipped: {str(e)}")

# Return result without quality metrics if evaluation failed
return {
    "status": "success",
    "path": output_path,
    "version": version,
    "tier": tier,
    "line_count": line_count,
    "sections": sections_count,
    "examples": examples_count,
    "quality": "Evaluation skipped",
    "validation": validation_result
}
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
