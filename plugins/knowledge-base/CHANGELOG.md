# Changelog

All notable changes to the Knowledge Base Plugin will be documented in this file.

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
