---
name: knowledge-evaluate
description: >
  Evaluate knowledge entry quality across 5 criteria (Clarity, Format,
  Structure, Completeness, Efficiency). Supports fast mode (structural
  analysis only, <2s) and deep mode (adds content analysis, <30s).
  Returns formatted text output (not JSON) to avoid serialization overhead.
allowed-tools: Read, Grep
context: fork
---

> **⚠️ EXECUTION CONSTRAINT**: All code blocks below are pseudocode for reference only.
> - **NEVER** create or execute `.py` scripts
> - **NEVER** use Bash to run `python` or `python3` commands
> - Implement all logic using Claude's built-in tools: Read, Grep, Glob, Write, Edit, Bash, Task
> - Parse YAML/JSON content mentally from Read tool output — do NOT use Python yaml/json libraries
> - `invoke_skill()` / `invoke_agent()` in pseudocode = use **Task** tool with appropriate subagent_type

# Knowledge Entry Quality Evaluation Skill

Evaluates knowledge entries using a 5-criteria framework with dual execution modes.

## Input Format

Receive file path as text prompt:

```
Evaluate knowledge entry quality for file: .conventions/typescript/naming/functions.md
Mode: fast
```

**Parameters:**
- First line: File path after "Evaluate knowledge entry quality for file: "
- Second line: Mode (optional, defaults to "fast")
  - `fast`: Structural analysis only (<2 seconds)
  - `deep`: Full content + structural analysis (<30 seconds)

## Evaluation Criteria

### 1. Однозначность (Clarity): 0-10

**Measures**: Can developer understand exactly how to apply the knowledge entry?

**Fast Mode Scoring:**
- Allowed section with 4+ code examples: +2 points
- Forbidden section with 4+ anti-patterns: +2 points
- Rules section with 7+ numbered rules: +3 points
- Total 12+ code blocks: +2 points
- Language specificity (auto-granted): +1 point

**Deep Mode Enhancement:**
- Replaces auto-granted point with vague language analysis
- Checks for patterns: "should consider", "try to", "generally", "might"
- Penalizes unclear wording in rules

### 2. Формат (Format): 0-10

**Measures**: Visual structure, readability, formatting quality

**Scoring (same for fast/deep):**
- Valid YAML frontmatter with all required fields: +2 points
- Format section with 4+ bullets: +2 points
- Code blocks with language tags (90%+): +2 points
- Valid heading hierarchy (no skipped levels): +2 points
- Summary table with 5+ rows: +2 points

### 3. Структура (Structure): 0-10

**Measures**: Logical organization and information flow

**Scoring (same for fast/deep):**
- Required sections present (Format, Allowed, Forbidden, Rules): +3 points
- Logical section order: +2 points
- Domain-specific subsections (6+ h3 headings): +2 points
- Cross-references (2+ internal links): +1 point
- Concept separation (no mixed topics): +2 points

### 4. Полнота (Completeness): 0-10

**Measures**: Comprehensive coverage of necessary aspects

**Fast Mode Scoring:**
- Multiple code examples (12+): +2 points
- Example variety (4+ different patterns): +2 points
- Both positive/negative examples (4+ each): +2 points
- Context description (300+ chars): +2 points
- Summary table (3+ cols, 5+ rows): +2 points

**Deep Mode Enhancement:**
- Replaces example variety with edge case analysis
- Checks for explicit edge case mentions (3+)

### 5. Польза/Размер (Utility/Efficiency): 0-10

**Measures**: High information density, optimal token usage

**Fast Mode Scoring:**
- Line count 400-650: +3 points (degrading outside range)
- Example-to-prose ratio 30-50%: +2 points
- Balanced sections (no section >40% of total): +2 points
- Average sentence length ≤25 words: +2 points
- Tags 4-8: +1 point

**Deep Mode Enhancement:**
- Replaces balanced sections with redundancy analysis
- Replaces sentence length with readability scoring (Flesch-Kincaid)

## Processing Steps

### Step 1: Read and Parse File

1. Use **Read** tool to load the knowledge entry file at the given `file_path`.
2. Mentally parse the YAML frontmatter: locate the opening `---` and closing `---` delimiters, extract all key-value pairs (type, version, scope, category, tags, etc.) into a `metadata` structure. If no YAML frontmatter is found, return an error: "Missing YAML frontmatter".
3. Separate the `body` (everything after the closing `---`) from the frontmatter.
4. Split the body into sections by `## ` headings — each heading name maps to its content below it.
5. Extract all code blocks from the body by identifying triple-backtick fenced regions, noting the language tag (if any) and the code content of each block.
6. Extract numbered rules from the "Rules" section by identifying lines matching the pattern `^\s*\d+\.\s+(.+)$`.

### Step 2: Score Each Criterion

**Clarity (Fast Mode):**

SCORING LOGIC (reference only — apply mentally, do NOT execute as code):

```text
score_clarity_fast(sections, code_blocks, rules):
    score = 0
    issues = []

    # Allowed section examples
    allowed = sections.get("Allowed", "")
    allowed_examples = count_code_blocks(allowed)
    if allowed_examples >= 4:
        score += 2
    else:
        issues.append("Allowed section has only {allowed_examples} examples (target: 4+)")

    # Forbidden section examples
    forbidden = sections.get("Forbidden", "")
    forbidden_examples = count_code_blocks(forbidden)
    if forbidden_examples >= 4:
        score += 2
    else:
        issues.append("Forbidden section has only {forbidden_examples} examples (target: 4+)")

    # Rules count
    if len(rules) >= 7:
        score += 3
    elif len(rules) >= 5:
        score += 2
        issues.append("Rules section has {len(rules)} rules (target: 7+)")
    else:
        issues.append("Insufficient rules: {len(rules)} (target: 7+)")

    # Total code blocks
    if len(code_blocks) >= 12:
        score += 2
    else:
        issues.append("Only {len(code_blocks)} code blocks (target: 12+)")

    # Auto-grant language specificity
    score += 1

    return score, issues
```

**Clarity (Deep Mode):**

SCORING LOGIC (reference only — apply mentally, do NOT execute as code):

```text
score_clarity_deep(sections, code_blocks, rules):
    score, issues = score_clarity_fast(sections, code_blocks, rules)

    # Remove auto-granted point
    score -= 1

    # Analyze vague language in rules
    vague_patterns = [
        \bshould consider\b,
        \btry to\b,
        \bgenerally\b,
        \bmight\b,
        \bcould\b,
        \bpossibly\b
    ]

    rules_text = sections.get("Rules", "")
    vague_count = 0
    for each pattern in vague_patterns:
        count occurrences of pattern in rules_text (case-insensitive)
        vague_count += number of matches

    if vague_count == 0:
        score += 1  # Full language specificity point
    elif vague_count <= 2:
        score += 0.5
        issues.append("Some vague language detected ({vague_count} instances)")
    else:
        issues.append("Excessive vague language ({vague_count} instances) - use 'must' instead of 'should'")

    return score, issues
```

**Format:**

SCORING LOGIC (reference only — apply mentally, do NOT execute as code):

```text
score_format(content, sections, code_blocks, metadata):
    score = 0
    issues = []

    # YAML frontmatter validation
    required_fields = ['type', 'version', 'scope', 'category', 'tags']
    missing = [f for f in required_fields if f not in metadata]
    if not missing:
        score += 2
    else:
        issues.append("Missing YAML fields: {missing}")

    # Format section
    format_section = sections.get("Format", "")
    bullet_count = count lines matching ^\s*[-*]\s+\*\* in format_section (multiline)
    if bullet_count >= 4:
        score += 2
    else:
        issues.append("Format section has {bullet_count} bullets (target: 4+)")

    # Code block language tags
    untagged = count code_blocks where language tag is empty
    tagged_ratio = 1 - (untagged / len(code_blocks)) if code_blocks else 0
    if tagged_ratio >= 0.9:
        score += 2
    else:
        issues.append("{untagged} code blocks missing language tags")

    # Heading hierarchy
    headings = extract_headings(content)
    hierarchy_valid, hierarchy_issues = validate_heading_hierarchy(headings)
    if hierarchy_valid:
        score += 2
    else:
        issues.extend(hierarchy_issues)

    # Summary table
    summary_section = sections.get("Summary", "")
    table_rows = count_table_rows(summary_section)
    if table_rows >= 5:
        score += 2
    elif table_rows >= 3:
        score += 1
        issues.append("Summary table has {table_rows} rows (target: 5+)")
    else:
        issues.append("Summary table too small: {table_rows} rows (target: 5+)")

    return score, issues
```

**Structure:**

SCORING LOGIC (reference only — apply mentally, do NOT execute as code):

```text
score_structure(sections):
    score = 0
    issues = []

    # Required sections
    required = ["Format", "Allowed", "Forbidden", "Rules"]
    missing = [s for s in required if s not in sections]
    if not missing:
        score += 3
    else:
        issues.append("Missing required sections: {missing}")
        score += max(0, 3 - len(missing))

    # Logical section order
    ideal_order = ["Format", "Allowed", "Forbidden", "Rules"]
    actual_order = [s for s in sections.keys() if s in ideal_order]
    if actual_order == ideal_order:
        score += 2
    else:
        issues.append("Section order differs from standard (Format → Allowed → Forbidden → Rules)")

    # Domain subsections (h3)
    h3_count = count headings at level 3 within sections
    if h3_count >= 6:
        score += 2
    elif h3_count >= 4:
        score += 1
        issues.append("Only {h3_count} subsections (target: 6+ for comprehensive coverage)")
    else:
        issues.append("Insufficient subsections: {h3_count} (target: 6+)")

    # Cross-references
    cross_refs = find all matches of \[.*?\]\(.*?\.md\) across all section content
    if len(cross_refs) >= 2:
        score += 1
    elif len(cross_refs) == 1:
        score += 0.5
        issues.append("Consider adding more cross-references to related conventions")
    else:
        issues.append("No cross-references found - consider linking related conventions")

    # Concept separation
    # Check if sections mix multiple topics (heuristic: keyword overlap)
    separation_valid = check_concept_separation(sections)
    if separation_valid:
        score += 2
    else:
        issues.append("Some sections appear to mix multiple concepts")

    return score, issues
```

**Completeness (Fast Mode):**

SCORING LOGIC (reference only — apply mentally, do NOT execute as code):

```text
score_completeness_fast(sections, code_blocks):
    score = 0
    issues = []

    # Multiple code examples
    if len(code_blocks) >= 12:
        score += 2
    else:
        issues.append("Only {len(code_blocks)} code examples (target: 12+)")

    # Example variety (different patterns)
    unique_patterns = count_unique_code_patterns(code_blocks)
    if unique_patterns >= 4:
        score += 2
    else:
        issues.append("Limited example variety: {unique_patterns} patterns (target: 4+)")

    # Positive/negative balance
    allowed_examples = count_code_blocks(sections.get("Allowed", ""))
    forbidden_examples = count_code_blocks(sections.get("Forbidden", ""))
    if allowed_examples >= 4 and forbidden_examples >= 4:
        score += 2
    else:
        issues.append("Imbalanced examples: {allowed_examples} allowed, {forbidden_examples} forbidden (target: 4+ each)")

    # Context description
    description = sections.get("Description", "")
    if len(description.strip()) >= 300:
        score += 2
    else:
        issues.append("Brief description ({len(description)} chars, target: 300+)")

    # Summary table
    summary = sections.get("Summary", "")
    cols = count_table_columns(summary)
    rows = count_table_rows(summary)
    if cols >= 3 and rows >= 5:
        score += 2
    else:
        issues.append("Summary table: {cols} cols x {rows} rows (target: 3+ cols, 5+ rows)")

    return score, issues
```

**Completeness (Deep Mode):**

SCORING LOGIC (reference only — apply mentally, do NOT execute as code):

```text
score_completeness_deep(sections, code_blocks):
    score, issues = score_completeness_fast(sections, code_blocks)

    # Replace example variety with edge case analysis
    score -= 2  # Remove variety points

    # Check for edge case coverage
    edge_case_keywords = [
        'edge case', 'corner case', 'special case',
        'exception', 'boundary', 'limit', 'empty',
        'null', 'undefined', 'zero', 'negative'
    ]

    content = join all section values, lowercased
    edge_mentions = count how many of edge_case_keywords appear in content

    if edge_mentions >= 3:
        score += 2
    elif edge_mentions >= 2:
        score += 1
        issues.append("Limited edge case coverage (add 1-2 more edge cases)")
    else:
        issues.append("Insufficient edge case coverage ({edge_mentions} mentions, target: 3+)")

    return score, issues
```

**Efficiency (Fast Mode):**

SCORING LOGIC (reference only — apply mentally, do NOT execute as code):

```text
score_efficiency_fast(content, sections, code_blocks, metadata):
    score = 0
    issues = []

    line_count = number of lines in content

    # Optimal line count (400-650)
    if 400 <= line_count <= 650:
        score += 3
    elif 350 <= line_count < 400:
        score += 2
        issues.append("Line count below optimal ({line_count}, target: 400-650)")
    elif 650 < line_count <= 700:
        score += 2
        issues.append("Line count above optimal ({line_count}, target: 400-650)")
    else:
        score += 1
        issues.append("Line count outside optimal range ({line_count}, target: 400-650)")

    # Example-to-prose ratio
    code_chars = sum of character lengths of all code block contents
    total_chars = len(content)
    ratio = code_chars / total_chars if total_chars > 0 else 0

    if 0.30 <= ratio <= 0.50:
        score += 2
    elif 0.25 <= ratio < 0.30 or 0.50 < ratio <= 0.55:
        score += 1
        issues.append("Example ratio {ratio} slightly off (target: 30-50%)")
    else:
        issues.append("Example ratio {ratio} outside optimal range (target: 30-50%)")

    # Balanced sections
    section_sizes = {name: len(text) for each section}
    total_size = sum(section_sizes.values())
    max_section_pct = max(size / total_size for size in section_sizes.values()) if total_size > 0 else 0

    if max_section_pct <= 0.40:
        score += 2
    else:
        largest = section with maximum size
        issues.append("Unbalanced sections: '{largest}' is {max_section_pct} of content (max: 40%)")

    # Average sentence length
    sentences = split content by [.!?] followed by whitespace
    avg_words = sum(word count per sentence) / len(sentences) if sentences else 0

    if avg_words <= 25:
        score += 2
    elif avg_words <= 30:
        score += 1
        issues.append("Average sentence length {avg_words} words (target: <=25)")
    else:
        issues.append("Complex sentences: avg {avg_words} words (target: <=25)")

    # Tags count
    tags = metadata.get('tags', [])
    if 4 <= len(tags) <= 8:
        score += 1
    else:
        issues.append("Suboptimal tag count: {len(tags)} (target: 4-8)")

    return score, issues
```

**Efficiency (Deep Mode):**

SCORING LOGIC (reference only — apply mentally, do NOT execute as code):

```text
score_efficiency_deep(content, sections, code_blocks, metadata):
    score, issues = score_efficiency_fast(content, sections, code_blocks, metadata)

    # Remove balanced sections check (-2) and sentence length check (-2)
    # (already calculated in fast mode, will be replaced)

    # Redundancy analysis
    prose_text = content with all code blocks removed
    redundancy_pct = calculate_redundancy(prose_text)
        (see Deep Mode Analysis section for redundancy calculation logic)

    if redundancy_pct < 5:
        score += 2  # Replace balanced sections points
    elif redundancy_pct < 10:
        score += 1
        issues.append("Some redundancy detected ({redundancy_pct}%)")
    else:
        issues.append("High redundancy ({redundancy_pct}%) - consider consolidating similar content")

    # Readability (Flesch-Kincaid)
    readability_score = calculate_readability(prose_text)
        (see Deep Mode Analysis section for Flesch-Kincaid formula)

    if readability_score >= 60:  # Easy to read
        score += 2  # Replace sentence length points
    elif readability_score >= 50:
        score += 1
        issues.append("Moderate readability score ({readability_score}, target: 60+)")
    else:
        issues.append("Low readability score ({readability_score}, target: 60+) - simplify language")

    return score, issues
```

### Step 3: Generate Output

Return formatted text output (NOT JSON) to avoid serialization overhead.

**Fast Mode Output:**

```
Quality: 8.4/10 (Clarity: 8, Format: 9, Structure: 9, Completeness: 8, Efficiency: 8)
```

This compact single-line format allows the calling skill to easily parse scores using regex.

**Deep Mode Output:**

```
# Quality Report

Overall: 8.4/10

Criteria Scores:
- Clarity (Однозначность): 8/10
- Format (Формат): 9/10
- Structure (Структура): 9/10
- Completeness (Полнота): 8/10
- Efficiency (Польза/Размер): 8/10

Issues:
- Clarity: Rules 3 and 7 use 'should' instead of 'must'
- Format: Summary table has 4 rows (target: 5+)
- Completeness: Only 11 code blocks (target: 12+)

Recommendations:
1. Replace 'should' with 'must' in 2 rules requiring strict compliance
2. Add 1 row to summary table
3. Add 2-3 edge case examples to improve coverage
```

This detailed multi-line format provides comprehensive feedback for the /knowledge-report command.

## Helper Functions

### Parsing Utilities

**parse_markdown_sections(body):** Iterate through lines of the body. When a line starts with `## `, save the previous section (heading name mapped to its accumulated lines) and start a new section using the text after `## ` as the key. After the loop, save the final section. Returns a dictionary of section names to their content.

**extract_code_blocks(text):** Find all fenced code blocks by matching the regex `` ```(\w+)?\n(.*?)``` `` (with DOTALL flag). For each match, record the language tag (group 1, or empty string if absent) and the code content (group 2). Returns a list of objects with `language` and `code` fields.

**extract_numbered_rules(text):** Find all lines matching `^\s*\d+\.\s+(.+)$` (multiline) in the given text. Returns the captured rule text for each numbered item.

**extract_headings(text):** Find all lines matching `^(#{1,6})\s+(.+)$` (multiline). For each match, record the heading level (count of `#` characters) and the heading text. Returns a list of objects with `level` and `text` fields.

**count_code_blocks(text):** Count occurrences of `` ``` `` in the text, then divide by 2 (integer division). This gives the number of complete code blocks.

**count_table_rows(text):** Count lines containing `|` in the text. Subtract 1 for the header separator row. Return the result, with a minimum of 0.

**count_table_columns(text):** Find the first line containing `|`. Split it by `|` and count non-empty, stripped segments. Returns that count, or 0 if no table line is found.

### Validation Utilities

SCORING LOGIC (reference only — apply mentally, do NOT execute as code):

```text
validate_heading_hierarchy(headings):
    """Check if heading levels are valid (no skipped levels)."""
    issues = []
    prev_level = 0

    for h in headings:
        level = h['level']
        if level > prev_level + 1:
            issues.append("Heading hierarchy skip: h{prev_level} -> h{level} ('{h.text}')")
        prev_level = level

    return len(issues) == 0, issues

check_concept_separation(sections):
    """Heuristic: check if sections have distinct topics."""
    section_keywords = {}
    for name, text in sections.items():
        words = find all matches of \b[a-z]{4,}\b in text.lower()
        section_keywords[name] = top 10 most common words by frequency

    # Check for excessive overlap
    for each unique pair (s1, s2) of sections:
        overlap = intersection of top-10 keyword sets
        if len(overlap) > 6:  # >60% overlap
            return False

    return True

count_unique_code_patterns(code_blocks):
    """Estimate unique patterns by clustering similar code."""
    first_lines = set()
    for block in code_blocks:
        code = block.code stripped
        if code:
            first_line = first line of code, stripped
            # Normalize: replace all word tokens with 'X' using \b\w+\b
            first_lines.add(normalized first_line)

    return len(first_lines)
```

### Deep Mode Analysis

SCORING LOGIC (reference only — apply mentally, do NOT execute as code):

```text
calculate_redundancy(text):
    """Calculate percentage of repeated phrases (3+ words)."""
    # Extract trigrams
    words = text.lower().split()
    trigrams = [join words[i] through words[i+2] for i in range(len(words) - 2)]

    counts = frequency count of each trigram
    repeated = sum(count - 1 for each trigram where count > 1)

    return (repeated / len(trigrams) * 100) if trigrams else 0
    # Redundancy > 20% is flagged in deep mode

calculate_readability(text):
    """Flesch Reading Ease score (0-100, higher = easier)."""
    sentences = count of segments split by [.!?]
    words = count of whitespace-separated tokens
    syllables = estimate_syllables(text)

    if sentences == 0 or words == 0:
        return 0

    score = 206.835 - 1.015 * (words / sentences) - 84.6 * (syllables / words)
    return max(0, min(100, score))

estimate_syllables(text):
    """Rough syllable count estimation."""
    words = text.lower().split()
    syllables = 0

    for word in words:
        # Count vowel groups matching [aeiouy]+
        syllables += number of vowel group matches in word

    return max(syllables, len(words))  # At least 1 syllable per word
```

## Error Handling

**Error handling decision tree:**

1. **File not found**: If the **Read** tool returns an error indicating the file does not exist, return: `"Knowledge entry not found: {file_path}"`
2. **Invalid YAML frontmatter**: If the content between `---` delimiters cannot be parsed as valid YAML (e.g., malformed key-value pairs, incorrect indentation), return: `"Invalid YAML frontmatter: {description of the parsing issue}"`
3. **Any other failure**: If evaluation cannot be completed for any other reason (missing sections, unreadable content, etc.), return: `"Evaluation failed: {description of the error}"`

In all error cases, output the error message as plain text and stop further processing.

## Implementation

When invoked, this skill:

1. **Parses text input prompt**:
   Parse the input prompt text to extract `file_path` and `mode`. From the first line, remove the prefix "Evaluate knowledge entry quality for file:" and trim whitespace to get the file path. From the second line (if present), check if it starts with "Mode:" and extract the value (either "fast" or "deep"). Default to "fast" if no mode line is provided.

2. **Reads the knowledge entry**:
   Use **Read** tool to get the knowledge entry content at the extracted `file_path`.

3. **Parses YAML frontmatter and markdown sections** (using helper functions)

4. **Executes scoring algorithms based on mode** (all scoring logic unchanged)

5. **Returns formatted text output** (NOT JSON):
   If mode is "fast", produce output using the single-line compact template. If mode is "deep", produce output using the multi-line detailed template. Output as plain text (NOT JSON).

   **Fast mode output template:**

   ```text
   Quality: {overall_score}/10 (Clarity: {clarity}, Format: {format}, Structure: {structure}, Completeness: {completeness}, Efficiency: {efficiency})
   ```

   **Deep mode output template:**

   ```text
   # Quality Report

   Overall: {overall_score}/10

   Criteria Scores:
   - Clarity (Однозначность): {clarity}/10
   - Format (Формат): {format}/10
   - Structure (Структура): {structure}/10
   - Completeness (Полнота): {completeness}/10
   - Efficiency (Польза/Размер): {efficiency}/10

   Issues:
   {formatted_issues}

   Recommendations:
   {formatted_recommendations}
   ```

The skill runs in an isolated context (`context: fork`) to prevent pollution of the main conversation and enable parallel evaluations. Text-based I/O eliminates JSON serialization overhead.
