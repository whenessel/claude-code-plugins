---
name: codebase-discover
description: >
  Analyze project codebase to discover de-facto coding conventions.
  Detects project stack, scans files across multiple dimensions,
  calculates consistency scores, and produces a ranked discovery plan.
  Does NOT generate knowledge entries — only produces the analysis.
allowed-tools: Read, Grep, Glob, Bash
---

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
```python
config_patterns = [
    # JavaScript/TypeScript ecosystem
    "package.json",
    "tsconfig.json",
    "tsconfig*.json",
    ".eslintrc*",
    ".prettierrc*",
    "prettier.config.*",
    "biome.json",
    "biome.jsonc",
    ".stylelintrc*",
    "vite.config.*",
    "next.config.*",
    "webpack.config.*",
    "tailwind.config.*",

    # Python ecosystem
    "pyproject.toml",
    "setup.py",
    "setup.cfg",
    "requirements*.txt",
    "Pipfile",
    ".flake8",
    "mypy.ini",
    ".mypy.ini",
    "ruff.toml",
    ".ruff.toml",
    ".isort.cfg",

    # Go
    "go.mod",
    "go.sum",

    # Rust
    "Cargo.toml",

    # Java/Kotlin
    "build.gradle*",
    "pom.xml",

    # General
    ".editorconfig",
    "Dockerfile*",
    "docker-compose*.yml",
    ".gitignore"
]

found_configs = {}
for pattern in config_patterns:
    results = Glob(pattern=pattern, path=scan_path)
    if results:
        found_configs[pattern] = results
```

**1B. Count files by extension:**
```python
extension_map = {
    "ts": "typescript",
    "tsx": "typescript",  # also signals React
    "js": "javascript",
    "jsx": "javascript",  # also signals React
    "py": "python",
    "go": "golang",
    "rs": "rust",
    "java": "java",
    "kt": "kotlin",
    "rb": "ruby",
    "php": "php",
    "vue": "vue",
    "svelte": "svelte"
}

file_counts = {}
for ext, lang in extension_map.items():
    files = Glob(pattern=f"**/*.{ext}", path=scan_path)
    # Exclude common non-source directories
    files = [
        f for f in files
        if not any(excl in f for excl in [
            "node_modules", "dist", "build", ".next",
            "__pycache__", ".venv", "venv", ".git",
            "vendor", "target", ".tox"
        ])
    ]
    if files:
        file_counts[ext] = len(files)
```

**1C. Detect frameworks from dependencies:**
```python
def detect_frameworks(configs: dict) -> list:
    frameworks = []

    # package.json
    if "package.json" in configs:
        pkg = json.loads(Read(configs["package.json"][0]))
        deps = {
            **pkg.get("dependencies", {}),
            **pkg.get("devDependencies", {})
        }

        framework_markers = {
            "react": "react",
            "next": "nextjs",
            "@angular/core": "angular",
            "vue": "vue",
            "svelte": "svelte",
            "express": "express",
            "fastify": "fastify",
            "@nestjs/core": "nestjs",
            "tailwindcss": "tailwind",
            "@chakra-ui/react": "chakra-ui",
            "zustand": "zustand",
            "@reduxjs/toolkit": "redux",
            "prisma": "prisma",
            "drizzle-orm": "drizzle",
            "vitest": "vitest",
            "jest": "jest",
            "mocha": "mocha",
            "storybook": "storybook"
        }

        for marker, name in framework_markers.items():
            if marker in deps:
                frameworks.append(name)

    # pyproject.toml
    if "pyproject.toml" in configs:
        content = Read(configs["pyproject.toml"][0])

        python_markers = {
            "fastapi": "fastapi",
            "django": "django",
            "flask": "flask",
            "sqlalchemy": "sqlalchemy",
            "pydantic": "pydantic",
            "pytest": "pytest",
            "alembic": "alembic"
        }

        for marker, name in python_markers.items():
            if marker in content.lower():
                frameworks.append(name)

    return frameworks
```

**1D. Read linter configurations:**
```python
def extract_linter_rules(configs: dict) -> dict:
    """Extract relevant linting rules from config files."""

    rules = {}

    # ESLint
    eslint_files = [
        f for pattern, files in configs.items()
        if ".eslintrc" in pattern
        for f in files
    ]
    if eslint_files:
        try:
            content = Read(eslint_files[0])
            eslint_config = json.loads(content)  # or parse YAML
            rules["eslint"] = eslint_config.get("rules", {})
        except:
            pass

    # TypeScript config
    if "tsconfig.json" in configs:
        try:
            content = Read(configs["tsconfig.json"][0])
            ts_config = json.loads(content)
            rules["typescript"] = ts_config.get("compilerOptions", {})
        except:
            pass

    # Prettier
    prettier_files = [
        f for pattern, files in configs.items()
        if ".prettierrc" in pattern or "prettier.config" in pattern
        for f in files
    ]
    if prettier_files:
        try:
            content = Read(prettier_files[0])
            rules["prettier"] = json.loads(content)
        except:
            pass

    # Ruff / flake8 / mypy
    for tool in ["ruff.toml", ".ruff.toml", ".flake8", "mypy.ini"]:
        if tool in configs:
            try:
                rules[tool.replace(".", "")] = Read(configs[tool][0])
            except:
                pass

    return rules
```

**1E. Build stack summary:**
```python
stack = {
    "languages": determine_languages(file_counts),
    "frameworks": detect_frameworks(found_configs),
    "tools": list(found_configs.keys()),
    "file_counts": file_counts,
    "linter_rules": extract_linter_rules(found_configs)
}
```

### Step 2: Build Dimension Registry

Define analysis dimensions based on detected stack. Each dimension specifies what to search for and how to measure consistency.

```python
def build_dimension_registry(stack: dict) -> list:
    """Build list of dimensions to analyze based on project stack."""

    dimensions = []

    # ─── Universal dimensions (all projects) ──────────────────────

    dimensions.append({
        "id": "file-naming",
        "scope": "general",
        "category": "naming",
        "dimension": "files",
        "title": "File Naming Conventions",
        "applicable_extensions": list(stack["file_counts"].keys()),
        "analysis_type": "filename_pattern",
        "patterns": {
            "kebab-case": r'^[a-z][a-z0-9]*(-[a-z0-9]+)*\.\w+$',
            "camelCase": r'^[a-z][a-zA-Z0-9]*\.\w+$',
            "PascalCase": r'^[A-Z][a-zA-Z0-9]*\.\w+$',
            "snake_case": r'^[a-z][a-z0-9]*(_[a-z0-9]+)*\.\w+$'
        }
    })

    dimensions.append({
        "id": "directory-structure",
        "scope": "general",
        "category": "structure",
        "dimension": "directories",
        "title": "Directory Structure Conventions",
        "analysis_type": "directory_pattern",
        "patterns": {
            "feature-based": ["features/", "modules/", "domains/"],
            "layer-based": ["controllers/", "services/", "models/", "repositories/"],
            "type-based": ["components/", "hooks/", "utils/", "types/", "helpers/"]
        }
    })

    # ─── TypeScript dimensions ────────────────────────────────────

    if "typescript" in stack["languages"] or "javascript" in stack["languages"]:
        ts_exts = ["ts", "tsx", "js", "jsx"]

        dimensions.extend([
            {
                "id": "ts-function-naming",
                "scope": "typescript",
                "category": "naming",
                "dimension": "functions",
                "title": "Function Naming Conventions",
                "applicable_extensions": ts_exts,
                "analysis_type": "grep_pattern",
                "grep_patterns": [
                    {"label": "camelCase functions", "pattern": r'(function|const|let)\s+[a-z][a-zA-Z0-9]*\s*[=(]'},
                    {"label": "verb prefix (get/set/is/has/create/update/delete/fetch/handle)", "pattern": r'(function|const)\s+(get|set|is|has|create|update|delete|fetch|handle|on|use)[A-Z]'},
                    {"label": "async functions", "pattern": r'async\s+(function\s+\w+|\w+\s*=\s*async)'},
                    {"label": "arrow functions", "pattern": r'const\s+\w+\s*=\s*(\([^)]*\)|[a-zA-Z]+)\s*=>'}
                ],
                "sample_limit": 30
            },
            {
                "id": "ts-class-naming",
                "scope": "typescript",
                "category": "naming",
                "dimension": "classes",
                "title": "Class and Interface Naming",
                "applicable_extensions": ts_exts,
                "analysis_type": "grep_pattern",
                "grep_patterns": [
                    {"label": "PascalCase classes", "pattern": r'(class|interface|type|enum)\s+[A-Z][a-zA-Z0-9]*'},
                    {"label": "I-prefix interfaces", "pattern": r'interface\s+I[A-Z][a-zA-Z0-9]*'},
                    {"label": "T-prefix types", "pattern": r'type\s+T[A-Z][a-zA-Z0-9]*'},
                    {"label": "Suffix patterns (Service/Controller/Repository)", "pattern": r'class\s+\w+(Service|Controller|Repository|Handler|Factory|Provider|Manager)'}
                ],
                "sample_limit": 30
            },
            {
                "id": "ts-imports",
                "scope": "typescript",
                "category": "structure",
                "dimension": "imports",
                "title": "Import Organization",
                "applicable_extensions": ts_exts,
                "analysis_type": "file_analysis",
                "check_patterns": [
                    {"label": "grouped imports (blank line between groups)", "check": "import_grouping"},
                    {"label": "path aliases (@/ or ~/)", "pattern": r"from\s+['\"][@~]/"},
                    {"label": "barrel imports (index.ts re-exports)", "check": "barrel_exports"},
                    {"label": "type imports (import type)", "pattern": r"import\s+type\s+"}
                ],
                "sample_limit": 30
            },
            {
                "id": "ts-error-handling",
                "scope": "typescript",
                "category": "patterns",
                "dimension": "error-handling",
                "title": "Error Handling Patterns",
                "applicable_extensions": ts_exts,
                "analysis_type": "grep_pattern",
                "grep_patterns": [
                    {"label": "try-catch blocks", "pattern": r'\btry\s*\{'},
                    {"label": "custom error classes", "pattern": r'class\s+\w+Error\s+extends\s+(Error|AppError)'},
                    {"label": "error logging", "pattern": r'(logger|console)\.(error|warn)\('},
                    {"label": "Result/Either types", "pattern": r'(Result|Either|Success|Failure)<'}
                ],
                "sample_limit": 30
            },
            {
                "id": "ts-exports",
                "scope": "typescript",
                "category": "structure",
                "dimension": "exports",
                "title": "Export Conventions",
                "applicable_extensions": ts_exts,
                "analysis_type": "grep_pattern",
                "grep_patterns": [
                    {"label": "named exports", "pattern": r'export\s+(const|function|class|interface|type|enum)\s+'},
                    {"label": "default exports", "pattern": r'export\s+default\s+'},
                    {"label": "barrel re-exports", "pattern": r"export\s+(\*|\{[^}]+\})\s+from\s+['\"]"},
                    {"label": "inline exports (export at declaration)", "pattern": r'^export\s+(const|function|class)'}
                ],
                "sample_limit": 30
            },
            {
                "id": "ts-docs",
                "scope": "typescript",
                "category": "docs",
                "dimension": "jsdoc",
                "title": "Documentation Standards",
                "applicable_extensions": ts_exts,
                "analysis_type": "grep_pattern",
                "grep_patterns": [
                    {"label": "JSDoc comments", "pattern": r'/\*\*'},
                    {"label": "@param tags", "pattern": r'@param\s+'},
                    {"label": "@returns tags", "pattern": r'@returns?\s+'},
                    {"label": "@throws/@example tags", "pattern": r'@(throws|example)\s+'}
                ],
                "sample_limit": 30,
                "consistency_base": "exported_functions"
            }
        ])

    # ─── React dimensions ─────────────────────────────────────────

    if "react" in stack["frameworks"] or "tsx" in stack["file_counts"]:
        dimensions.extend([
            {
                "id": "react-components",
                "scope": "react",
                "category": "naming",
                "dimension": "components",
                "title": "Component Naming and Structure",
                "applicable_extensions": ["tsx", "jsx"],
                "analysis_type": "grep_pattern",
                "grep_patterns": [
                    {"label": "PascalCase components", "pattern": r'(export\s+)?(default\s+)?function\s+[A-Z][a-zA-Z0-9]*\s*\('},
                    {"label": "arrow function components", "pattern": r'(export\s+)?const\s+[A-Z][a-zA-Z0-9]*\s*[=:]\s*.*=>\s*'},
                    {"label": "Props type/interface", "pattern": r'(type|interface)\s+\w+Props\s*[={]'},
                    {"label": "component file = component name", "check": "filename_matches_component"}
                ],
                "sample_limit": 30
            },
            {
                "id": "react-hooks",
                "scope": "react",
                "category": "patterns",
                "dimension": "hooks",
                "title": "Hooks Usage Patterns",
                "applicable_extensions": ["tsx", "jsx", "ts", "js"],
                "analysis_type": "grep_pattern",
                "grep_patterns": [
                    {"label": "custom hooks (use* prefix)", "pattern": r'(export\s+)?(function|const)\s+use[A-Z][a-zA-Z0-9]*'},
                    {"label": "hooks in dedicated files", "check": "hooks_file_pattern"},
                    {"label": "useCallback/useMemo usage", "pattern": r'(useCallback|useMemo)\s*\('},
                    {"label": "useEffect cleanup", "pattern": r'useEffect\s*\(\s*\(\)\s*=>\s*\{[\s\S]*return\s'}
                ],
                "sample_limit": 30
            },
            {
                "id": "react-state",
                "scope": "react",
                "category": "patterns",
                "dimension": "state-management",
                "title": "State Management Patterns",
                "applicable_extensions": ["tsx", "jsx", "ts", "js"],
                "analysis_type": "grep_pattern",
                "grep_patterns": [
                    {"label": "useState", "pattern": r'useState\s*[<(]'},
                    {"label": "useReducer", "pattern": r'useReducer\s*[<(]'},
                    {"label": "Zustand stores", "pattern": r'(create|useStore)\s*[<(]'},
                    {"label": "Redux toolkit", "pattern": r'(createSlice|useSelector|useDispatch)\s*[<(]'},
                    {"label": "React Query/TanStack", "pattern": r'(useQuery|useMutation)\s*[<(]'}
                ],
                "sample_limit": 30
            }
        ])

    # ─── Python dimensions ────────────────────────────────────────

    if "python" in stack["languages"]:
        dimensions.extend([
            {
                "id": "py-function-naming",
                "scope": "python",
                "category": "naming",
                "dimension": "functions",
                "title": "Function Naming Conventions",
                "applicable_extensions": ["py"],
                "analysis_type": "grep_pattern",
                "grep_patterns": [
                    {"label": "snake_case functions", "pattern": r'def\s+[a-z][a-z0-9_]*\s*\('},
                    {"label": "type hints in signatures", "pattern": r'def\s+\w+\(.*:\s*\w+.*\)\s*(->|:)'},
                    {"label": "return type annotations", "pattern": r'def\s+\w+\(.*\)\s*->\s*\w+'},
                    {"label": "async functions", "pattern": r'async\s+def\s+'}
                ],
                "sample_limit": 30
            },
            {
                "id": "py-class-naming",
                "scope": "python",
                "category": "naming",
                "dimension": "classes",
                "title": "Class Naming Conventions",
                "applicable_extensions": ["py"],
                "analysis_type": "grep_pattern",
                "grep_patterns": [
                    {"label": "PascalCase classes", "pattern": r'class\s+[A-Z][a-zA-Z0-9]*'},
                    {"label": "Base/Abstract prefix", "pattern": r'class\s+(Base|Abstract)[A-Z]'},
                    {"label": "Mixin suffix", "pattern": r'class\s+\w+Mixin'},
                    {"label": "dataclass/pydantic models", "pattern": r'(@dataclass|class\s+\w+\(BaseModel\))'}
                ],
                "sample_limit": 30
            },
            {
                "id": "py-module-structure",
                "scope": "python",
                "category": "structure",
                "dimension": "modules",
                "title": "Module Organization",
                "applicable_extensions": ["py"],
                "analysis_type": "grep_pattern",
                "grep_patterns": [
                    {"label": "__init__.py with exports", "check": "init_exports"},
                    {"label": "__all__ defined", "pattern": r'__all__\s*=\s*\['},
                    {"label": "relative imports", "pattern": r'from\s+\.\w*\s+import'},
                    {"label": "absolute imports", "pattern": r'from\s+[a-zA-Z]\w+(\.\w+)+\s+import'}
                ],
                "sample_limit": 30
            },
            {
                "id": "py-error-handling",
                "scope": "python",
                "category": "patterns",
                "dimension": "error-handling",
                "title": "Error Handling Patterns",
                "applicable_extensions": ["py"],
                "analysis_type": "grep_pattern",
                "grep_patterns": [
                    {"label": "try-except blocks", "pattern": r'\btry\s*:'},
                    {"label": "custom exception classes", "pattern": r'class\s+\w+(Error|Exception)\s*\('},
                    {"label": "specific exception catching", "pattern": r'except\s+[A-Z]\w+(Error|Exception)'},
                    {"label": "logging errors", "pattern": r'(logger|logging)\.(error|exception|warning)\('}
                ],
                "sample_limit": 30
            },
            {
                "id": "py-docs",
                "scope": "python",
                "category": "docs",
                "dimension": "docstrings",
                "title": "Documentation Standards",
                "applicable_extensions": ["py"],
                "analysis_type": "grep_pattern",
                "grep_patterns": [
                    {"label": "triple-quote docstrings", "pattern": r'"""[\s\S]'},
                    {"label": "Google-style docstrings (Args:)", "pattern": r'^\s+Args:\s*$'},
                    {"label": "NumPy-style docstrings (Parameters)", "pattern": r'^\s+Parameters\s*$'},
                    {"label": "Returns section", "pattern": r'^\s+Returns?:\s*$'}
                ],
                "sample_limit": 30
            }
        ])

    # ─── Filter by scope if specified ─────────────────────────────

    if scope_filter:
        dimensions = [d for d in dimensions if d["scope"] == scope_filter or d["scope"] == "general"]

    return dimensions
```

### Step 3: Analyze Each Dimension

For each dimension, collect evidence and calculate consistency score.

```python
def analyze_dimension(dimension: dict, scan_path: str, file_counts: dict) -> dict:
    """Analyze a single dimension across the codebase."""

    analysis_type = dimension["analysis_type"]

    if analysis_type == "grep_pattern":
        return analyze_grep_dimension(dimension, scan_path)
    elif analysis_type == "filename_pattern":
        return analyze_filename_dimension(dimension, scan_path, file_counts)
    elif analysis_type == "directory_pattern":
        return analyze_directory_dimension(dimension, scan_path)
    elif analysis_type == "file_analysis":
        return analyze_file_dimension(dimension, scan_path)
```

**Grep-based analysis:**

```python
def analyze_grep_dimension(dimension: dict, scan_path: str) -> dict:
    """Analyze dimension using grep patterns."""

    extensions = dimension["applicable_extensions"]
    sample_limit = dimension.get("sample_limit", 30)

    # Step A: Find relevant files
    all_files = []
    for ext in extensions:
        files = Glob(pattern=f"**/*.{ext}", path=scan_path)
        files = filter_source_files(files)  # Exclude node_modules, dist, etc.
        all_files.extend(files)

    total_relevant_files = len(all_files)

    if total_relevant_files == 0:
        return {"skip": True, "reason": "No relevant files found"}

    # Step B: Analyze each grep pattern
    patterns_found = []

    for gp in dimension["grep_patterns"]:
        if "check" in gp:
            # Custom check (not a simple grep)
            result = run_custom_check(gp["check"], all_files, scan_path)
            patterns_found.append({
                "pattern": gp["label"],
                "count": result["count"],
                "pct": result["count"] / total_relevant_files if total_relevant_files > 0 else 0,
                "sample_files": result.get("sample_files", [])[:5]
            })
            continue

        # Grep for pattern
        try:
            file_type_filter = ",".join(extensions)
            grep_result = Grep(
                pattern=gp["pattern"],
                include=file_type_filter,
                path=scan_path,
                output_mode="files_with_matches"
            )

            matching_files = parse_grep_files(grep_result)
            matching_files = filter_source_files(matching_files)

            patterns_found.append({
                "pattern": gp["label"],
                "count": len(matching_files),
                "pct": len(matching_files) / total_relevant_files if total_relevant_files > 0 else 0,
                "sample_files": matching_files[:5]
            })
        except Exception:
            patterns_found.append({
                "pattern": gp["label"],
                "count": 0,
                "pct": 0,
                "sample_files": [],
                "error": True
            })

    # Step C: Sample code from top-matching files
    sample_code = collect_code_samples(
        patterns_found,
        dimension["grep_patterns"],
        sample_limit
    )

    # Step D: Calculate overall dimension confidence
    confidence = calculate_dimension_confidence(patterns_found, total_relevant_files)

    # Step E: Determine dominant pattern
    dominant = determine_dominant_pattern(patterns_found)

    return {
        "skip": False,
        "confidence": confidence,
        "file_count": total_relevant_files,
        "dominant_pattern": dominant,
        "patterns_found": patterns_found,
        "sample_code": sample_code,
        "sample_files": collect_unique_sample_files(patterns_found, limit=10)
    }
```

**Filename-based analysis:**

```python
def analyze_filename_dimension(dimension: dict, scan_path: str, file_counts: dict) -> dict:
    """Analyze filename patterns."""

    all_files = []
    for ext in dimension["applicable_extensions"]:
        files = Glob(pattern=f"**/*.{ext}", path=scan_path)
        files = filter_source_files(files)
        all_files.extend(files)

    if not all_files:
        return {"skip": True, "reason": "No files found"}

    # Extract just filenames (without path and extension)
    filenames = [
        os.path.splitext(os.path.basename(f))[0]
        for f in all_files
    ]

    # Test each naming pattern
    pattern_counts = {}
    for name, regex in dimension["patterns"].items():
        matches = [fn for fn in filenames if re.match(regex, fn + ".x")]
        pattern_counts[name] = {
            "count": len(matches),
            "pct": len(matches) / len(filenames),
            "examples": matches[:5]
        }

    # Find dominant pattern
    dominant_name = max(pattern_counts, key=lambda k: pattern_counts[k]["count"])
    dominant = pattern_counts[dominant_name]

    patterns_found = [
        {
            "pattern": name,
            "count": data["count"],
            "pct": data["pct"],
            "sample_files": data["examples"]
        }
        for name, data in pattern_counts.items()
        if data["count"] > 0
    ]

    return {
        "skip": False,
        "confidence": dominant["pct"],
        "file_count": len(filenames),
        "dominant_pattern": f"{dominant_name} ({dominant['pct']:.0%})",
        "patterns_found": sorted(patterns_found, key=lambda x: x["count"], reverse=True),
        "sample_code": [f"# Examples: {', '.join(dominant['examples'][:5])}"],
        "sample_files": dominant["examples"][:10]
    }
```

**Directory structure analysis:**

```python
def analyze_directory_dimension(dimension: dict, scan_path: str) -> dict:
    """Analyze directory organization patterns."""

    # List top-level and second-level directories
    top_dirs = Bash(f"find {scan_path} -maxdepth 2 -type d | head -50")
    dir_names = [os.path.basename(d) for d in top_dirs.split('\n') if d.strip()]

    patterns_found = []
    for pattern_name, markers in dimension["patterns"].items():
        matches = [
            m.rstrip("/") for m in markers
            if m.rstrip("/") in dir_names
        ]
        if matches:
            patterns_found.append({
                "pattern": f"{pattern_name} ({', '.join(matches)})",
                "count": len(matches),
                "pct": len(matches) / len(markers),
                "sample_files": matches
            })

    if not patterns_found:
        return {"skip": True, "reason": "No recognized directory patterns"}

    dominant = max(patterns_found, key=lambda x: x["pct"])

    return {
        "skip": False,
        "confidence": dominant["pct"],
        "file_count": len(dir_names),
        "dominant_pattern": dominant["pattern"],
        "patterns_found": patterns_found,
        "sample_code": [f"# Structure: {d}" for d in dir_names[:10]],
        "sample_files": dir_names[:10]
    }
```

### Step 4: Collect Code Samples

```python
def collect_code_samples(patterns_found: list, grep_patterns: list, limit: int) -> list:
    """Read actual code samples from matching files."""

    samples = []
    seen_files = set()

    # Prioritize files that match multiple patterns
    file_scores = {}
    for pf in patterns_found:
        for f in pf.get("sample_files", []):
            file_scores[f] = file_scores.get(f, 0) + 1

    # Sort by score (most patterns matched)
    ranked_files = sorted(file_scores.keys(), key=lambda f: file_scores[f], reverse=True)

    for file_path in ranked_files[:limit]:
        if file_path in seen_files:
            continue
        seen_files.add(file_path)

        try:
            content = Read(file_path)
            # Extract first relevant snippet (function/class declaration + body preview)
            snippet = extract_relevant_snippet(content, grep_patterns)
            if snippet:
                samples.append(snippet)
        except:
            continue

        if len(samples) >= 6:
            break

    return samples


def extract_relevant_snippet(content: str, grep_patterns: list) -> str:
    """Extract a meaningful code snippet from file content."""

    lines = content.split('\n')

    for gp in grep_patterns:
        if "pattern" not in gp:
            continue
        pattern = gp["pattern"]
        for i, line in enumerate(lines):
            if re.search(pattern, line):
                # Extract context: line + next 3-5 lines
                start = i
                end = min(i + 5, len(lines))
                snippet = '\n'.join(lines[start:end])
                if len(snippet) > 200:
                    snippet = snippet[:200] + "..."
                return snippet

    return None
```

### Step 5: Calculate Confidence Score

```python
def calculate_dimension_confidence(patterns_found: list, total_files: int) -> float:
    """Calculate overall confidence for a dimension.

    Confidence is based on:
    1. Dominant pattern percentage (primary factor)
    2. Number of files matching (minimum threshold)
    3. Pattern clarity (gap between #1 and #2 pattern)
    """

    if not patterns_found or total_files == 0:
        return 0.0

    # Sort by match count
    sorted_patterns = sorted(patterns_found, key=lambda x: x["count"], reverse=True)

    # Primary: dominant pattern percentage
    dominant_pct = sorted_patterns[0]["pct"]

    # Minimum file threshold (at least 5 files must match)
    if sorted_patterns[0]["count"] < 5:
        dominant_pct *= 0.5  # Penalize small sample

    # Clarity bonus: if dominant pattern is clearly ahead
    if len(sorted_patterns) >= 2:
        gap = sorted_patterns[0]["pct"] - sorted_patterns[1]["pct"]
        if gap > 0.3:
            dominant_pct = min(1.0, dominant_pct * 1.1)  # Slight bonus for clarity

    return round(dominant_pct, 2)
```

### Step 6: Assemble Discovery Plan

```python
def assemble_plan(stack, dimension_results, min_confidence):
    """Build the final discovery plan."""

    discovered = []
    id_counter = 1

    for dim_id, result in dimension_results.items():
        if result.get("skip"):
            continue

        if result["confidence"] < min_confidence / 100:
            continue  # Below threshold, still include for display if > 0.3

        # Get dimension definition
        dim_def = get_dimension_by_id(dim_id)

        # Map dimension to suggested type
        dimension_to_type = {
            # Naming dimensions → convention
            "functions": "convention",
            "variables": "convention",
            "components": "convention",
            "classes": "convention",
            "modules": "convention",
            "files": "convention",

            # Structural dimensions → pattern
            "imports": "pattern",
            "exports": "pattern",
            "error-handling": "pattern",

            # Documentation dimensions → style
            "jsdoc": "style",
            "docstrings": "style",

            # State/architecture → pattern
            "state-management": "pattern",
            "hooks": "pattern",

            # Environment/config dimensions → environment
            "directories": "environment",
            "project-structure": "environment",
            "config-files": "environment",
            "ci-cd": "environment",
            "docker": "environment",
            "toolchain": "environment"
        }
        suggested_type = dimension_to_type.get(dim_def["dimension"], "convention")

        discovered.append({
            "id": id_counter,
            "scope": dim_def["scope"],
            "category": dim_def["category"],
            "dimension": dim_def["dimension"],
            "suggested_type": suggested_type,
            "title": dim_def["title"],
            "confidence": result["confidence"],
            "file_count": result["file_count"],
            "dominant_pattern": result["dominant_pattern"],
            "evidence": {
                "patterns_found": result["patterns_found"],
                "sample_files": result["sample_files"],
                "sample_code": result["sample_code"]
            },
            "linter_rules": extract_relevant_linter_rules(
                stack["linter_rules"],
                dim_def
            ),
            "suggested_tags": generate_tags(dim_def, result)
        })

        id_counter += 1

    # Sort: high confidence first, then by file count
    discovered.sort(key=lambda x: (-x["confidence"], -x["file_count"]))

    # Re-assign sequential IDs after sorting
    for i, item in enumerate(discovered):
        item["id"] = i + 1

    return {
        "project_stack": stack,
        "discovered": discovered,
        "total_files_analyzed": sum(stack["file_counts"].values())
    }
```

### Step 7: Generate Tags

```python
def generate_tags(dimension_def: dict, result: dict) -> list:
    """Generate relevant tags for the discovered convention."""

    tags = set()

    # From scope
    tags.add(dimension_def["scope"])

    # From category
    tags.add(dimension_def["category"])

    # From dominant pattern
    pattern_keywords = {
        "camelCase": "camelCase",
        "PascalCase": "PascalCase",
        "snake_case": "snake_case",
        "kebab-case": "kebab-case",
        "try-catch": "error-handling",
        "async": "async",
        "hooks": "hooks",
        "useState": "state",
        "import": "imports"
    }

    dominant = result["dominant_pattern"].lower()
    for keyword, tag in pattern_keywords.items():
        if keyword.lower() in dominant:
            tags.add(tag)

    # From dimension name
    tags.add(dimension_def["dimension"])

    return sorted(list(tags))[:8]  # Max 8 tags
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

```python
def filter_source_files(files: list) -> list:
    """Exclude non-source directories."""
    exclude = [
        "node_modules", "dist", "build", ".next", "out",
        "__pycache__", ".venv", "venv", "env",
        ".git", ".svn",
        "vendor", "target",
        ".tox", ".mypy_cache", ".pytest_cache",
        "coverage", ".nyc_output",
        ".kb"  # Don't scan knowledge base itself
    ]
    return [
        f for f in files
        if not any(excl in f.split(os.sep) for excl in exclude)
    ]


def extract_relevant_linter_rules(all_rules: dict, dimension: dict) -> dict:
    """Extract linter rules relevant to this dimension."""
    relevant = {}

    scope = dimension["scope"]
    dim = dimension["dimension"]

    # ESLint rules relevant to naming
    if "eslint" in all_rules and "naming" in dimension["category"]:
        naming_rules = {
            k: v for k, v in all_rules["eslint"].items()
            if any(keyword in k for keyword in ["camel", "pascal", "naming", "id-", "func-name"])
        }
        if naming_rules:
            relevant.update(naming_rules)

    # TypeScript compiler options relevant to structure
    if "typescript" in all_rules and scope == "typescript":
        ts_opts = all_rules["typescript"]
        relevant_keys = ["strict", "noImplicitAny", "strictNullChecks", "paths", "baseUrl"]
        for key in relevant_keys:
            if key in ts_opts:
                relevant[key] = ts_opts[key]

    return relevant


def run_custom_check(check_name: str, files: list, scan_path: str) -> dict:
    """Run custom analysis checks that can't be done with simple grep."""

    if check_name == "import_grouping":
        # Check if imports are separated by blank lines into groups
        count = 0
        sample_files = []
        for f in files[:30]:
            content = Read(f)
            import_lines = [l for l in content.split('\n') if l.startswith('import ') or l.startswith('from ')]
            has_groups = '\n\n' in '\n'.join(
                l for l in content.split('\n')
                if l.startswith('import ') or l.startswith('from ') or l.strip() == ''
            )
            if has_groups and len(import_lines) > 3:
                count += 1
                sample_files.append(f)
        return {"count": count, "sample_files": sample_files}

    elif check_name == "barrel_exports":
        # Check for index.ts files with re-exports
        index_files = Glob(pattern="**/index.{ts,js}", path=scan_path)
        index_files = filter_source_files(index_files)
        count = 0
        sample_files = []
        for f in index_files[:20]:
            content = Read(f)
            if 'export' in content and 'from' in content:
                count += 1
                sample_files.append(f)
        return {"count": count, "sample_files": sample_files}

    elif check_name == "init_exports":
        # Check for __init__.py with exports
        init_files = Glob(pattern="**/__init__.py", path=scan_path)
        init_files = filter_source_files(init_files)
        count = 0
        sample_files = []
        for f in init_files[:20]:
            content = Read(f)
            if content.strip():  # Non-empty __init__.py
                count += 1
                sample_files.append(f)
        return {"count": count, "sample_files": sample_files}

    elif check_name == "hooks_file_pattern":
        # Check if hooks are in dedicated files (useXxx.ts)
        hook_files = Glob(pattern="**/use*.{ts,tsx,js,jsx}", path=scan_path)
        hook_files = filter_source_files(hook_files)
        return {"count": len(hook_files), "sample_files": hook_files[:5]}

    elif check_name == "filename_matches_component":
        # Check if component filename matches the default export
        count = 0
        sample_files = []
        for f in files[:30]:
            filename = os.path.splitext(os.path.basename(f))[0]
            content = Read(f)
            if f'function {filename}' in content or f'const {filename}' in content:
                count += 1
                sample_files.append(f)
        return {"count": count, "sample_files": sample_files}

    return {"count": 0, "sample_files": []}
```

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
