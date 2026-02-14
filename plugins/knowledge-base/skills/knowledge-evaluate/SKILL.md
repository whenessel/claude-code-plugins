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

```python
# Read knowledge entry
content = Read(file_path)

# Parse YAML frontmatter
yaml_match = re.match(r'^---\n(.*?)\n---\n', content, re.DOTALL)
if not yaml_match:
    return error("Missing YAML frontmatter")

metadata = yaml.safe_load(yaml_match.group(1))
body = content[yaml_match.end():]

# Extract sections
sections = parse_markdown_sections(body)

# Extract code blocks
code_blocks = extract_code_blocks(body)

# Extract rules
rules = extract_numbered_rules(sections.get("Rules", ""))
```

### Step 2: Score Each Criterion

**Clarity (Fast Mode):**

```python
def score_clarity_fast(sections, code_blocks, rules):
    score = 0
    issues = []

    # Allowed section examples
    allowed = sections.get("Allowed", "")
    allowed_examples = count_code_blocks(allowed)
    if allowed_examples >= 4:
        score += 2
    else:
        issues.append(f"Allowed section has only {allowed_examples} examples (target: 4+)")

    # Forbidden section examples
    forbidden = sections.get("Forbidden", "")
    forbidden_examples = count_code_blocks(forbidden)
    if forbidden_examples >= 4:
        score += 2
    else:
        issues.append(f"Forbidden section has only {forbidden_examples} examples (target: 4+)")

    # Rules count
    if len(rules) >= 7:
        score += 3
    elif len(rules) >= 5:
        score += 2
        issues.append(f"Rules section has {len(rules)} rules (target: 7+)")
    else:
        issues.append(f"Insufficient rules: {len(rules)} (target: 7+)")

    # Total code blocks
    if len(code_blocks) >= 12:
        score += 2
    else:
        issues.append(f"Only {len(code_blocks)} code blocks (target: 12+)")

    # Auto-grant language specificity
    score += 1

    return score, issues
```

**Clarity (Deep Mode):**

```python
def score_clarity_deep(sections, code_blocks, rules):
    score, issues = score_clarity_fast(sections, code_blocks, rules)

    # Remove auto-granted point
    score -= 1

    # Analyze vague language in rules
    vague_patterns = [
        r'\bshould consider\b',
        r'\btry to\b',
        r'\bgenerally\b',
        r'\bmight\b',
        r'\bcould\b',
        r'\bpossibly\b'
    ]

    rules_text = sections.get("Rules", "")
    vague_count = 0
    for pattern in vague_patterns:
        matches = re.findall(pattern, rules_text, re.IGNORECASE)
        vague_count += len(matches)

    if vague_count == 0:
        score += 1  # Full language specificity point
    elif vague_count <= 2:
        score += 0.5
        issues.append(f"Some vague language detected ({vague_count} instances)")
    else:
        issues.append(f"Excessive vague language ({vague_count} instances) - use 'must' instead of 'should'")

    return score, issues
```

**Format:**

```python
def score_format(content, sections, code_blocks, metadata):
    score = 0
    issues = []

    # YAML frontmatter validation
    required_fields = ['type', 'version', 'scope', 'category', 'tags']
    missing = [f for f in required_fields if f not in metadata]
    if not missing:
        score += 2
    else:
        issues.append(f"Missing YAML fields: {', '.join(missing)}")

    # Format section
    format_section = sections.get("Format", "")
    bullet_count = len(re.findall(r'^\s*[-*]\s+\*\*', format_section, re.MULTILINE))
    if bullet_count >= 4:
        score += 2
    else:
        issues.append(f"Format section has {bullet_count} bullets (target: 4+)")

    # Code block language tags
    untagged = sum(1 for cb in code_blocks if not cb.get('language'))
    tagged_ratio = 1 - (untagged / len(code_blocks)) if code_blocks else 0
    if tagged_ratio >= 0.9:
        score += 2
    else:
        issues.append(f"{untagged} code blocks missing language tags")

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
        issues.append(f"Summary table has {table_rows} rows (target: 5+)")
    else:
        issues.append(f"Summary table too small: {table_rows} rows (target: 5+)")

    return score, issues
```

**Structure:**

```python
def score_structure(sections):
    score = 0
    issues = []

    # Required sections
    required = ["Format", "Allowed", "Forbidden", "Rules"]
    missing = [s for s in required if s not in sections]
    if not missing:
        score += 3
    else:
        issues.append(f"Missing required sections: {', '.join(missing)}")
        score += max(0, 3 - len(missing))

    # Logical section order
    ideal_order = ["Format", "Allowed", "Forbidden", "Rules"]
    actual_order = [s for s in sections.keys() if s in ideal_order]
    if actual_order == ideal_order:
        score += 2
    else:
        issues.append("Section order differs from standard (Format → Allowed → Forbidden → Rules)")

    # Domain subsections (h3)
    h3_count = sum(1 for h in extract_headings(sections) if h['level'] == 3)
    if h3_count >= 6:
        score += 2
    elif h3_count >= 4:
        score += 1
        issues.append(f"Only {h3_count} subsections (target: 6+ for comprehensive coverage)")
    else:
        issues.append(f"Insufficient subsections: {h3_count} (target: 6+)")

    # Cross-references
    cross_refs = re.findall(r'\[.*?\]\(.*?\.md\)', ' '.join(sections.values()))
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

```python
def score_completeness_fast(sections, code_blocks):
    score = 0
    issues = []

    # Multiple code examples
    if len(code_blocks) >= 12:
        score += 2
    else:
        issues.append(f"Only {len(code_blocks)} code examples (target: 12+)")

    # Example variety (different patterns)
    unique_patterns = count_unique_code_patterns(code_blocks)
    if unique_patterns >= 4:
        score += 2
    else:
        issues.append(f"Limited example variety: {unique_patterns} patterns (target: 4+)")

    # Positive/negative balance
    allowed_examples = count_code_blocks(sections.get("Allowed", ""))
    forbidden_examples = count_code_blocks(sections.get("Forbidden", ""))
    if allowed_examples >= 4 and forbidden_examples >= 4:
        score += 2
    else:
        issues.append(f"Imbalanced examples: {allowed_examples} allowed, {forbidden_examples} forbidden (target: 4+ each)")

    # Context description
    description = sections.get("Description", "")
    if len(description.strip()) >= 300:
        score += 2
    else:
        issues.append(f"Brief description ({len(description)} chars, target: 300+)")

    # Summary table
    summary = sections.get("Summary", "")
    cols = count_table_columns(summary)
    rows = count_table_rows(summary)
    if cols >= 3 and rows >= 5:
        score += 2
    else:
        issues.append(f"Summary table: {cols} cols × {rows} rows (target: 3+ cols, 5+ rows)")

    return score, issues
```

**Completeness (Deep Mode):**

```python
def score_completeness_deep(sections, code_blocks):
    score, issues = score_completeness_fast(sections, code_blocks)

    # Replace example variety with edge case analysis
    score -= 2  # Remove variety points

    # Check for edge case coverage
    edge_case_keywords = [
        'edge case', 'corner case', 'special case',
        'exception', 'boundary', 'limit', 'empty',
        'null', 'undefined', 'zero', 'negative'
    ]

    content = ' '.join(sections.values()).lower()
    edge_mentions = sum(1 for keyword in edge_case_keywords if keyword in content)

    if edge_mentions >= 3:
        score += 2
    elif edge_mentions >= 2:
        score += 1
        issues.append("Limited edge case coverage (add 1-2 more edge cases)")
    else:
        issues.append(f"Insufficient edge case coverage ({edge_mentions} mentions, target: 3+)")

    return score, issues
```

**Efficiency (Fast Mode):**

```python
def score_efficiency_fast(content, sections, code_blocks, metadata):
    score = 0
    issues = []

    line_count = len(content.split('\n'))

    # Optimal line count (400-650)
    if 400 <= line_count <= 650:
        score += 3
    elif 350 <= line_count < 400:
        score += 2
        issues.append(f"Line count below optimal ({line_count}, target: 400-650)")
    elif 650 < line_count <= 700:
        score += 2
        issues.append(f"Line count above optimal ({line_count}, target: 400-650)")
    else:
        score += 1
        issues.append(f"Line count outside optimal range ({line_count}, target: 400-650)")

    # Example-to-prose ratio
    code_chars = sum(len(cb.get('code', '')) for cb in code_blocks)
    total_chars = len(content)
    ratio = code_chars / total_chars if total_chars > 0 else 0

    if 0.30 <= ratio <= 0.50:
        score += 2
    elif 0.25 <= ratio < 0.30 or 0.50 < ratio <= 0.55:
        score += 1
        issues.append(f"Example ratio {ratio:.1%} slightly off (target: 30-50%)")
    else:
        issues.append(f"Example ratio {ratio:.1%} outside optimal range (target: 30-50%)")

    # Balanced sections
    section_sizes = {name: len(text) for name, text in sections.items()}
    total_size = sum(section_sizes.values())
    max_section_pct = max(size / total_size for size in section_sizes.values()) if total_size > 0 else 0

    if max_section_pct <= 0.40:
        score += 2
    else:
        largest = max(section_sizes, key=section_sizes.get)
        issues.append(f"Unbalanced sections: '{largest}' is {max_section_pct:.0%} of content (max: 40%)")

    # Average sentence length
    sentences = re.split(r'[.!?]\s+', content)
    avg_words = sum(len(s.split()) for s in sentences) / len(sentences) if sentences else 0

    if avg_words <= 25:
        score += 2
    elif avg_words <= 30:
        score += 1
        issues.append(f"Average sentence length {avg_words:.1f} words (target: ≤25)")
    else:
        issues.append(f"Complex sentences: avg {avg_words:.1f} words (target: ≤25)")

    # Tags count
    tags = metadata.get('tags', [])
    if 4 <= len(tags) <= 8:
        score += 1
    else:
        issues.append(f"Suboptimal tag count: {len(tags)} (target: 4-8)")

    return score, issues
```

**Efficiency (Deep Mode):**

```python
def score_efficiency_deep(content, sections, code_blocks, metadata):
    score, issues = score_efficiency_fast(content, sections, code_blocks, metadata)

    # Remove balanced sections check (-2) and sentence length check (-2)
    # (already calculated in fast mode, will be replaced)

    # Redundancy analysis
    prose_text = remove_code_blocks(content)
    redundancy_pct = calculate_redundancy(prose_text)

    if redundancy_pct < 5:
        score += 2  # Replace balanced sections points
    elif redundancy_pct < 10:
        score += 1
        issues.append(f"Some redundancy detected ({redundancy_pct:.1f}%)")
    else:
        issues.append(f"High redundancy ({redundancy_pct:.1f}%) - consider consolidating similar content")

    # Readability (Flesch-Kincaid)
    readability_score = calculate_readability(prose_text)

    if readability_score >= 60:  # Easy to read
        score += 2  # Replace sentence length points
    elif readability_score >= 50:
        score += 1
        issues.append(f"Moderate readability score ({readability_score:.0f}, target: 60+)")
    else:
        issues.append(f"Low readability score ({readability_score:.0f}, target: 60+) - simplify language")

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

```python
def parse_markdown_sections(body: str) -> dict:
    """Extract sections by ## headers."""
    sections = {}
    current_section = None
    current_lines = []

    for line in body.split('\n'):
        if line.startswith('## '):
            if current_section:
                sections[current_section] = '\n'.join(current_lines)
            current_section = line[3:].strip()
            current_lines = []
        else:
            current_lines.append(line)

    if current_section:
        sections[current_section] = '\n'.join(current_lines)

    return sections

def extract_code_blocks(text: str) -> list:
    """Extract all code blocks with metadata."""
    pattern = r'```(\w+)?\n(.*?)```'
    blocks = []
    for match in re.finditer(pattern, text, re.DOTALL):
        blocks.append({
            'language': match.group(1) or '',
            'code': match.group(2)
        })
    return blocks

def extract_numbered_rules(text: str) -> list:
    """Extract numbered list items."""
    pattern = r'^\s*\d+\.\s+(.+)$'
    return re.findall(pattern, text, re.MULTILINE)

def extract_headings(text: str) -> list:
    """Extract all headings with levels."""
    headings = []
    for match in re.finditer(r'^(#{1,6})\s+(.+)$', text, re.MULTILINE):
        headings.append({
            'level': len(match.group(1)),
            'text': match.group(2)
        })
    return headings

def count_code_blocks(text: str) -> int:
    """Count code blocks in text."""
    return len(re.findall(r'```', text)) // 2

def count_table_rows(text: str) -> int:
    """Count table rows (lines with |)."""
    lines = [l for l in text.split('\n') if '|' in l]
    # Subtract header separator
    return max(0, len(lines) - 1)

def count_table_columns(text: str) -> int:
    """Count table columns from first row."""
    for line in text.split('\n'):
        if '|' in line:
            return len([c for c in line.split('|') if c.strip()])
    return 0
```

### Validation Utilities

```python
def validate_heading_hierarchy(headings: list) -> tuple[bool, list]:
    """Check if heading levels are valid (no skipped levels)."""
    issues = []
    prev_level = 0

    for h in headings:
        level = h['level']
        if level > prev_level + 1:
            issues.append(f"Heading hierarchy skip: h{prev_level} → h{level} ('{h['text']}')")
        prev_level = level

    return len(issues) == 0, issues

def check_concept_separation(sections: dict) -> bool:
    """Heuristic: check if sections have distinct topics."""
    # Extract keywords from each section
    from collections import Counter

    section_keywords = {}
    for name, text in sections.items():
        words = re.findall(r'\b[a-z]{4,}\b', text.lower())
        section_keywords[name] = Counter(words).most_common(10)

    # Check for excessive overlap
    for s1, kw1 in section_keywords.items():
        for s2, kw2 in section_keywords.items():
            if s1 >= s2:
                continue
            overlap = set(k for k, _ in kw1) & set(k for k, _ in kw2)
            if len(overlap) > 6:  # >60% overlap
                return False

    return True

def count_unique_code_patterns(code_blocks: list) -> int:
    """Estimate unique patterns by clustering similar code."""
    # Simplified: count distinct first lines
    first_lines = set()
    for block in code_blocks:
        code = block.get('code', '').strip()
        if code:
            first_line = code.split('\n')[0].strip()
            # Normalize
            first_line = re.sub(r'\b\w+\b', 'X', first_line)  # Replace identifiers
            first_lines.add(first_line)

    return len(first_lines)
```

### Deep Mode Analysis

```python
def calculate_redundancy(text: str) -> float:
    """Calculate percentage of repeated phrases (3+ words)."""
    # Extract trigrams
    words = text.lower().split()
    trigrams = [' '.join(words[i:i+3]) for i in range(len(words) - 2)]

    from collections import Counter
    counts = Counter(trigrams)
    repeated = sum(count - 1 for count in counts.values() if count > 1)

    return (repeated / len(trigrams) * 100) if trigrams else 0

def calculate_readability(text: str) -> float:
    """Flesch Reading Ease score (0-100, higher = easier)."""
    sentences = len(re.split(r'[.!?]', text))
    words = len(text.split())
    syllables = estimate_syllables(text)

    if sentences == 0 or words == 0:
        return 0

    score = 206.835 - 1.015 * (words / sentences) - 84.6 * (syllables / words)
    return max(0, min(100, score))

def estimate_syllables(text: str) -> int:
    """Rough syllable count estimation."""
    words = text.lower().split()
    syllables = 0

    for word in words:
        # Count vowel groups
        syllables += len(re.findall(r'[aeiouy]+', word))

    return max(syllables, len(words))  # At least 1 per word
```

## Error Handling

```python
try:
    # Main evaluation flow
    content = Read(file_path)
    # ... evaluation logic

except FileNotFoundError:
    return {
        "status": "error",
        "message": f"Knowledge entry not found: {file_path}"
    }

except yaml.YAMLError as e:
    return {
        "status": "error",
        "message": f"Invalid YAML frontmatter: {str(e)}"
    }

except Exception as e:
    return {
        "status": "error",
        "message": f"Evaluation failed: {str(e)}"
    }
```

## Implementation

When invoked, this skill:

1. **Parses text input prompt**:
   ```python
   # Extract file path from first line
   lines = prompt.strip().split('\n')
   file_path = lines[0].replace('Evaluate knowledge entry quality for file:', '').strip()

   # Extract mode from second line (default: fast)
   mode = 'fast'
   if len(lines) > 1 and lines[1].startswith('Mode:'):
       mode = lines[1].replace('Mode:', '').strip()
   ```

2. **Reads the knowledge entry**:
   ```python
   content = Read(file_path)
   ```

3. **Parses YAML frontmatter and markdown sections** (using helper functions)

4. **Executes scoring algorithms based on mode** (all scoring logic unchanged)

5. **Returns formatted text output** (NOT JSON):
   ```python
   # Fast mode: single-line compact format
   if mode == 'fast':
       output = f"Quality: {overall_score:.1f}/10 (Clarity: {clarity}, Format: {format}, Structure: {structure}, Completeness: {completeness}, Efficiency: {efficiency})"
       print(output)

   # Deep mode: multi-line detailed format
   else:
       output = f"""# Quality Report

Overall: {overall_score:.1f}/10

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
"""
       print(output)
   ```

The skill runs in an isolated context (`context: fork`) to prevent pollution of the main conversation and enable parallel evaluations. Text-based I/O eliminates JSON serialization overhead.
